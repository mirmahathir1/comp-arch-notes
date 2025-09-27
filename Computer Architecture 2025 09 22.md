# Admin

* Assignment 3 releases today. Topic: out-of-order (OoO) execution.
* Exams: material **through OoO and superscalar basics** is in Midterm 1; **after that** is on the Final.

# Recap: Tomasulo’s Algorithm (corrected)

* **Reservation stations (RS):** small buffers per functional unit. Issue checks only for a free RS entry, not for operand availability.
* **Register renaming:** renames architectural registers to tags, eliminating **WAR** and **WAW** hazards. **RAW** remains the true dependency.
* **Forwarding via the Common Data Bus (CDB):** when a unit finishes, it **broadcasts** the result tag+value on the CDB; all RS entries waiting on that tag latch it simultaneously.

# Strengths

* Issue proceeds when an RS entry is free.
* Renaming removes false dependencies, raising ILP.
* CDB provides just-in-time operand delivery to many consumers.

# Limits of “classic” Tomasulo (as in IBM 360/91 style, no ROB)

* **No speculation / no branch prediction support:** branches stall younger work until resolved because results are committed as soon as they’re produced.
* **Memory ordering is conservative:** loads and stores use **load/store buffers** but execute safely only when older memory ops will not conflict; naïve forms approach in-order behavior.
* **Imprecise exceptions:** results can update architected state out of program order, so an exception does not present a clean “all older done, none younger done” point.
* **Side-effects early:** a speculative **store** cannot safely update memory because undoing it requires logging old values.
* **CDB bottleneck:** a single broadcast path limits width and scaling.

# Why speculation is hard without in-order commit

### Branch example (corrected)

```
I1: R1 ← R2 / R3         ; long latency
I2: BEQZ R1, L1          ; depends on I1
I3: R2 ← R3 + R4         ; fall-through
I4: R4 ← R5 / R6         ; long latency
L1: ...
```

* If we **predict not-taken** and let I3/I4 execute, classic Tomasulo would **write back** their results before I2 resolves.
* If the prediction is wrong, there is **no architectural mechanism to “un-do”** those already-committed writes.

### Exceptions during speculation

* If I4 triggers **divide-by-zero** or a load triggers a **page fault**, the system must present **precise exceptions**: all older instructions complete, no younger ones do.
* Classic Tomasulo cannot guarantee that because younger results may have updated state already. Therefore exceptions would be **imprecise** unless we forbid speculation.

### Stores are special

* A **store** on a speculative path must not update memory early. Undo would require logging the old cacheline value. Correct fix: **delay stores until commit**.

# The standard fix: Reorder Buffer (ROB) + in-order commit

* **ROB holds all in-flight instructions in program order.**
* Units still use RS and CDB to execute and **write back to the ROB (or a physical register file)**, not to the architectural state.
* **Commit/retire occurs from the ROB head in order.** This enables:

  * **Branch prediction and speculative execution:** on mispredict, **flush** all younger ROB entries and restore the rename map (via checkpoint).
  * **Precise exceptions:** take the trap when the faulting instruction reaches the ROB head; all older have committed, none younger have.
  * **Safe stores:** place store data/address in a **store queue**; actually write to cache **at commit**.

# Making memory faster: LSQ and disambiguation

* **Load/Store Queue (LSQ):** tracks program order of memory ops.
* **Execute rules (modern OoO):**

  * A **load** may bypass older stores **when their addresses are known and do not alias**.
  * If an older store’s address is unknown, either stall or use **memory-dependence prediction**. If a violation is detected, **replay** the load and younger dependents.
* This recovers concurrency that naïve “top-of-queue only” execution would lose.

# Multiple-issue: Superscalar vs VLIW (corrected terms)

* **Superscalar:** hardware discovers independent instructions each cycle and issues N of them. All dependence checks, renaming, wakeup/select, and scheduling happen **at run time**.
* **VLIW (Very Long Instruction Word):** compiler statically packs independent ops into one wide instruction; hardware issues the whole bundle with little dynamic scheduling.
* **Intel Itanium (2001+)** used **EPIC** (a VLIW-like exposed-parallelism model with predication and speculation). It struggled due to binary compatibility, unpredictable latencies, code size, and complex compilers.
* **GPUs are not VLIW** in mainstream designs; they are **SIMT**: many threads execute a single instruction stream per warp with divergence handling.

# CPI, IPC, and width

* Single-issue best case: **CPI = 1** (one cycle per instruction).
* With multiple-issue, we target **IPC > 1** (effective **CPI < 1**).
* A superscalar core described as “N-way” can **fetch/rename/issue/commit up to N** instructions per cycle, subject to dependences and resources.
* Practical front-end/issue widths today: **~2–8**. Wider is limited by:

  * O(N²) dependency checks during rename/issue.
  * Wakeup/select complexity across many RS entries.
  * Bypass/CDB fabric scaling.
  * Front-end bandwidth and branch prediction accuracy.

# ILP vs threads vs cores (clean separation)

* **Everything above so far extracts ILP from a single software thread.**
* If hardware also issues from **multiple threads on one core**, that is **Simultaneous Multithreading (SMT)**, e.g., “2 threads per core.”
* If you duplicate the entire core, that is **multicore**.

# OS and placement (corrected)

* The OS sees **logical processors** (e.g., 4 cores × 4 SMT threads = 16 logical CPUs). Modern OSes also know topology (core vs thread) and can schedule accordingly.
* You can request placement with **processor affinity** APIs or with runtimes such as **OpenMP** pragmas; tightly communicating threads may benefit from sharing the same core’s caches.

# Pipeline vs OoO: why branch speculation was easy in a simple pipeline

* In a 5-stage in-order pipeline, a branch resolves before any younger instruction **commits**, so a misprediction just **squashes younger in-flight** ops.
* Classic Tomasulo **commits early** out of order, so mispredicted younger ops may have already updated state. The ROB restores the in-order commit point needed for safe squashing.

# Side note on Spectre/Meltdown (tightened)

* **Spectre:** exploits **mispredicted** transient execution to leak data via microarchitectural side channels.
* **Meltdown:** exploits **deferred privilege checks** so a transient load reads kernel data; the architectural fault arrives later, but the cache state leaks the value.
* Lesson: **Do not make architectural updates until commit;** also harden transient behaviors and side channels.

# Terminology fixes from the transcript

* Tomasulo’s algorithm, not “Thomas/tomosulus/tools.”
* **Superscalar**, not “supercala/supercaler.”
* **VLIW**, not “VIW.”
* **Common Data Bus (CDB)**, not “class.”
* **TLB (Translation Lookaside Buffer)**, not “translation side/look buffer.”
* **Precise exceptions** require **in-order commit**.

# Where this lecture goes next

* Add **ROB + rename map checkpoints** for speculation and precise exceptions.
* Strengthen the **LSQ** with address disambiguation and dependence prediction to increase memory concurrency.
* Scale to **N-way superscalar**: widen fetch/rename/issue/commit and address the hardware costs listed above.


# Unifying fix: **Reorder Buffer (ROB)** with in-order commit

* Single mechanism solves all three: branch speculation control, delayed exceptions on speculative paths, precise exceptions.
* **Buffering:** hold results of executed but uncommitted instructions. No architectural state change yet (no register file write, no memory write).
* **Ordering:** keep all in-flight instructions in **program order** inside the ROB. Commit only from the **ROB head**.

# What “in-order commit” guarantees

* **Branch prediction:** execute down the predicted path; if the branch later proves wrong, **flush all younger ROB entries** and restore the rename map. No architectural state was updated by flushed ops.
* **Exceptions on speculative ops:** detect at execute, but **raise only when the faulting instruction reaches the ROB head**. Guarantees the path is non-speculative.
* **Precise exceptions:** at trap time, **all older instructions have committed**; **no younger instruction has committed**.

# Tagging and renaming with a ROB

* Tags are **ROB IDs** (not RS IDs). All dependency tracking, wakeup, and bus broadcasts use the ROB ID.
* **Operand read cases** for an issued op `Rd ← Ra ⊕ Rb`:

  1. If `Ra/Rb` have **no rename mapping**: read the architectural register file.
  2. If mapped to a **ROB entry that is ready**: read the value from the ROB (or unified physical register file).
  3. If mapped to a **ROB entry not ready**: record the producing **ROB tag** and wait for the CDB wakeup.

# Issue conditions

* Issue only if both:

  * A **reservation station (RS)** entry is free for the target functional unit.
  * A **ROB entry** is free.
* On issue:

  * Allocate ROB entry and record destination register mapping (`rename_map[Rd] = rob_id`).
  * Push RS entry with source operands or source **ROB tags**.

# Execute and writeback

* RS fires when all source operands are present.
* On completion, the unit **broadcasts `<rob_id, value>` on the CDB**.
* All waiting RS entries capture the value; the ROB marks that entry **ready**.

# Commit rules by instruction type

* **ALU / FP:** when at ROB head and ready → **write result to architectural register file**, free ROB entry; clear `rename_map` for Rd if this ROB still owns it.
* **Load:** same as ALU (commit writes the register). Exceptions (e.g., page fault) are taken **at commit**.
* **Store:** compute address/data early, place in **Store Queue (SQ)**; **actual cache/memory write happens at commit**, in order.
* **Branch:** branches are **resolved at execute** (compare+target). If correct, it simply retires when reaching ROB head. If mispredicted, **flush younger ROB/SQ/RS state immediately** and restart fetch at the correct PC; commit pointer advances after flush.

# Loads, stores, and memory ordering (making them concurrent)

* Use a **Load/Store Queue (LSQ)** that keeps memory ops in program order and holds per-op address/data/status.
* **Safe early load execution:** a load may issue to memory when:

  1. **All older stores with known addresses** do **not** match the load’s address, and
  2. There is **no older store with unknown address** that could alias.
* If an older store’s address is **unknown**, either stall the load or use **memory-dependence prediction**; on violation, **replay** the load and its dependents.
* **Store-to-load forwarding:** if an older store to the **same address** has its data, forward directly from the SQ to the load result.
* **Stores never update memory speculatively.** Address and data can be computed early, but the write happens **only at commit**.

# Speculative loads vs. speculative stores

* **Loads:** may execute speculatively; their results sit in the ROB (or physical RF). If the path is squashed, the value is discarded; it never became architectural.
* **Stores:** may resolve address/data early and enter the SQ, but **do not write caches/memory** until they are **at the ROB head** and the branch history to them is validated.

# Branch recovery mechanics

* On branch execute:

  * Compute taken/not-taken and target.
  * If outcome/target ≠ prediction, **flush all younger** ROB and LSQ entries, invalidate their RS entries, and **restore a checkpointed rename map** taken at branch issue.
* Fetch resumes at the correct target with the restored map.

# Exception handling details

* On execute, faults are **recorded** in the ROB entry.
* The core **stops committing** younger entries once the faulting instruction reaches the head.
* At that point it raises the trap with architectural state consistent with a precise point.

# Putting it together: updated Tomasulo with a ROB

* **Front end:** fetch, decode, predict branches, **rename to ROB IDs**.
* **Issue:** need **RS + ROB** space. Sources come from regfile, ROB, or future CDB via ROB tags.
* **Execute:** like classic Tomasulo, but all IDs are **ROB IDs**; LSQ arbitrates memory ops using disambiguation and forwarding.
* **Writeback:** CDB broadcasts `<rob_id, value>`.
* **Commit (in order):** apply register writes, perform store writes, retire branches, take exceptions, advance head.

# Notes on advanced variants

* Many designs use a **Physical Register File (PRF)** with a **free list**; the ROB then holds status, not values. Commit frees the old physical register.
* **Value prediction** exists but is rare in general-purpose cores; it can start consumers of long-latency producers before the real value arrives, with misprediction recovery via replay.
* **Width scaling limits:** the ROB, RS, LSQ, bypass network, and wakeup/select logic drive practical widths (~2–8 wide).
