# Announcements

* Homework 2 is released. Due Wednesday with an automatic one-day grace period. Start early.

# Review: Pipelining and Hazards

* Pipelining overlaps instruction execution to approach CPI ≈ 1.
* Hazards prevent ideal CPI:

  * **Data hazards** (true dependences).
  * **Control hazards** (branches).
  * **Structural hazards** (resource conflicts).

Example:

```
1) LOAD  R1, [addr]         ; load to R1
2) ADD   R2, R1, R3         ; uses R1
3) ADD   R8, R9, R10        ; independent of 1 and 2
```

* If the load **hits** in L1 and has 1-cycle latency with forwarding, the load-use pair may still require a one-cycle bubble.
* If the load **misses** in cache, latency can be tens to hundreds of cycles. Instruction 2 must wait for R1. In a simple in-order pipeline, instruction 3 also stalls even though it is independent. This serialization is not required by program semantics; it is an artifact of the in-order pipeline.

# Static scheduling (compiler)

* The compiler can move independent instructions above the load-use pair:

```
1) LOAD  R1, [addr]
2) ADD   R8, R9, R10        ; hoisted to hide part of the load latency
3) ADD   R2, R1, R3
```

Limits:

* The compiler cannot know actual miss events or exact latency at runtime.
* It cannot always find enough independent work to cover worst-case latencies across levels (L1, L2, memory).
* Excessive code motion can bloat code or violate other constraints (e.g., exceptions, aliasing).

# Dynamic scheduling (hardware)

Core idea:

* Let the **hardware** find and run independent instructions at runtime.
* Issue instructions in program order, but allow **out-of-order execution** when operands are ready and a functional unit is free.
* Commit results to architectural state in order to preserve precise exceptions.

Benefits:

* Hides unpredictable latencies (cache misses, variable FP latencies, branch outcomes).
* Uses available parallel resources (multiple adders, multipliers, load/store units).

Terminology (precise):

* **Fetch**: read the next instruction(s).
* **Issue**: decode, check structural availability, and dispatch the instruction to a queue or reservation station associated with a functional unit.
* **Execute**: the functional unit runs when operands are ready. If not ready, the instruction **waits at the unit/queue**, not in the decode stage. This prevents front-end blockage.
* **Complete/Writeback**: the unit produces a result.
* **Commit/Retire**: update architectural registers and memory in **program order** (typically via a Reorder Buffer) to maintain a precise state.

Typical policy for dynamically scheduled, precise processors:

* **In-order issue**, **out-of-order execute**, **in-order commit**.

# Why independent instructions stall in a simple pipeline

* In a basic in-order, single-issue pipeline, decode cannot advance if the next instruction needs an operand that is not ready. The pipeline registers hold the stalled instruction, which blocks following instructions from entering decode/execute even if they are independent.
* Dynamic scheduling fixes this by letting the blocked instruction **wait near its functional unit**, freeing the front end to continue issuing later independent instructions.

# How to draw stalls in pipeline diagrams

* If an instruction stalls in **EX** for N cycles, show the EX stage stretched across N cycles for that instruction; following stages shift right accordingly.
* If the stall is at **IF** (e.g., an I-cache miss), show **no instruction fetch** for those cycles; the IF stage is idle, and later stages continue draining.
* Do not insert fake “noops” unless modeling a design that explicitly injects bubbles. A **hardware stall** is drawn as the stage holding the same instruction across multiple cycles.

# Functional units and superscalar note

* Multiple functional units (adders, multipliers, load/store units) allow overlap. Modern cores often have several instances due to **Moore’s law** (correct spelling).
* **Superscalar** means multiple instructions may be **fetched/decoded/issued** per cycle. That is orthogonal to dynamic scheduling. You can have dynamic scheduling with single-issue; superscalar amplifies the effect.

# Dependences and hazards: correct terms

We classify dependences between instructions using architectural registers or memory:

1. **RAW** = Read After Write (**true dependence**)

   * Later instruction needs a value produced by an earlier instruction.
   * Must be preserved to be correct.
   * Example:

     ```
     MUL R3, R1, R2   ; produces R3
     ADD R4, R3, R5   ; consumes R3
     ```

2. **WAR** = Write After Read (**anti-dependence**, name reuse)

   * Earlier instruction reads a register that a later instruction will write.
   * Not a flow of data; arises from **register name reuse**.
   * Can be eliminated by **register renaming**.

3. **WAW** = Write After Write (**output dependence**, name reuse)

   * Two instructions write the same register; order must ensure the later write is the final value.
   * Also removable via **register renaming**.

Only **RAW** is a true data flow constraint. **WAR** and **WAW** are **name dependences**. Hardware that performs register renaming can eliminate WAR/WAW hazards while preserving RAW.

# Why WAR/WAW matter under out-of-order

Out-of-order execution risks reordering writes and reads to the **same architectural name**:

```
I1: R1 = R2 / R3        ; long latency
I2: R4 = R1 + R5        ; RAW on R1 from I1
I3: R1 = R8 + R9        ; writes R1
```

* **WAW** between I1 and I3 on R1. If I3 completes before I1, the “final” R1 would be wrong unless ordered or renamed.
* **WAR** between I2 (reads R1) and I3 (writes R1). If I3 writes R1 before I2 reads, I2 could read the wrong value.

**Fix**: **Register renaming**

* Map architectural register names (R1, R2, …) to a larger set of **physical registers**.
* Give each producing instruction a fresh physical destination. Consumers read precisely the producing physical register.
* This removes WAR and WAW by construction and preserves RAW.

# “Set it aside” intuition, made concrete

Your intuition to “set the instruction aside” is correct. In hardware terms:

* Decode/issue places instructions into **reservation stations** or an **issue queue** associated with functional units.
* Each entry tracks:

  * Operation and destination physical register.
  * Source operands: either ready values or **tags** of producing instructions.
* When all sources are ready and a functional unit is free, the instruction **wakes up** and is selected to execute.
* Results broadcast on a common data bus to wake waiting consumers.
* Completion writes the result into the physical register; **commit** later updates the architectural mapping in order via the **Reorder Buffer** for precise state.

# Clarifications from the Q&A

**Q:** “If the load misses and an earlier instruction stalls, how do we draw later independent instructions in the timeline?”
**A:** In a simple in-order pipeline you cannot. They wait behind the stalled instruction. In a dynamically scheduled design, they issue into queues and may execute as soon as their operands and a unit are available. In timing diagrams, show the stalled instruction holding EX (or MEM) for many cycles, while independent instructions execute on other units during those cycles.

**Q:** “What if the stall is at instruction fetch?”
**A:** Show IF idle for those cycles. No new instruction enters the pipe. Downstream stages may continue if they have work; otherwise the pipe drains to empty.

**Q:** “Could multiple fetch/decode paths fix this?”
**A:** That is **superscalar** front-end width. It increases throughput but does not by itself solve dependency stalls. Dynamic scheduling plus register renaming is the standard solution.

**Q:** “Can vector or SIMD ideas help?”
**A:** SIMD/vector expresses parallelism explicitly in the ISA (e.g., `A[i]=B[i]+C[i]`). Dynamic scheduling extracts parallelism **implicitly** from scalar code. They are complementary. Hardware “loop unrolling” is analogous to looking ahead in the instruction stream.

**Q:** “Are we doing out-of-order issue?”
**A:** Practical precise designs use **in-order issue** into queues, **out-of-order execute**, **in-order commit**. Fully out-of-order issue complicates dependency tracking and is rarely used for precise, general-purpose cores.

Here is a corrected, polished continuation with clear elaboration. I keep your sequence, fix terminology, and tighten the logic.

# Picking up the dependency example

* Question in context:

  ```
  I1: R1 = R2 / R3        ; long-latency divide
  I2: R4 = R1 + R5        ; uses R1  → RAW on R1 from I1
  I3: R4 = R8 + R9        ; writes R4  → potential WAW with I2’s dest if same name
  ```
* A **WAR** hazard exists if a later instruction writes a source register before an earlier instruction has read it (e.g., if a later writer to `R1` arrives before `I2` has read `R1`). More importantly, a later instruction that writes the **same architectural name** as an earlier producer or consumer can create **WAW** or **WAR** problems when we execute out of order.
* Core correctness rule: **semantics** = for every register read in single-threaded code, you must return the value defined by the latest preceding write in **program order**. That is fully captured by **RAW (true) dependences**. **WAR/WAW** are name conflicts and are not true dataflow, but they can cause wrong reads or wrong final values if you reorder without care.

> Takeaway: Out-of-order execution must never violate RAW. WAR and WAW must be neutralized so they cannot corrupt RAW.

# Historical context

* **CDC 6600** (announced 1964), architect **Seymour Cray**, introduced dynamic scheduling via **scoreboarding** (principal designer **Jim Thornton**).
* **IBM System/360 Model 91** (mid-1960s, c. 1966) introduced **Tomasulo’s algorithm** (by **Robert Tomasulo**) for floating-point.
* **Gene Amdahl** was a leading IBM architect (Amdahl’s Law) but **not** the CDC 6600 architect.

# Scoreboarding: key mechanism and cautions

## Core idea

* **Decouple register reading from issue.**
* **Issue**: If the target functional unit (FU) is free, dispatch the instruction to that FU’s waiting area and mark metadata.
* **Execute**: The instruction **waits at the FU** until all sources are ready; then the FU runs.
* **Write result**: When done, write to the register file.

## How hazards are handled

* **Structural hazards**: If no FU is free, **stall at issue**.
* **WAW hazards**: At issue, check whether any earlier uncompleted instruction is scheduled to write the same architectural register. If yes, **block issue** to preserve the final-writer order.
* **RAW hazards**: The waiting-at-FU model naturally waits for the producing instruction to finish, then reads.
* **WAR hazards**: Classic scoreboard reads both sources **simultaneously** only when both are available. A later writer to a source register must not overwrite that source before the earlier reader has actually read it, so the scoreboard may **stall the later write-back** until all earlier reads have taken place.

## What this implies

* Reads and writes go **through the register file** (no global forwarding fabric).
* Conservative stalling:

  * At issue for **structural** and **WAW**.
  * At write-back for potential **WAR** (delay the write until earlier readers have read).

This keeps correctness but can leave performance on the table when latencies are large or when many instructions could overlap.

## Scoreboarding (CDC 6600): rules recap and worked example

### Rules

* **Issue**
  * Only issue if the target functional unit (FU) is free.
  * Stall if an earlier instruction writes the same destination (**WAW**).
  * Stall on structural hazard if the needed FU is busy.

* **Read operands**
  * Wait until all source operands are available in the register file.
  * Avoids **RAW** hazards.

* **Execute**
  * Execute in the FU; on completion, signal the scoreboard.

* **Write result**
  * Write back only when all earlier instructions have already read that destination register.
  * Avoids **WAR** hazards.

* **No forwarding**
  * All reads happen via the register file after writes; results are not forwarded directly from producing units.

### Example

```text
I1: R2 ← R1 + R3   ; ADD
I2: R4 ← R2 × R5   ; MUL
I3: R2 ← R6 − R7   ; SUB
I4: R8 ← R4 + R9   ; ADD
```

#### Walkthrough

* **I1 (ADD R2)**
  * Issues if ADD unit is free; reads R1, R3 when ready; executes.
  * Delays writing R2 if any earlier instruction still needs to read R2.

* **I2 (MUL R4 uses R2)**
  * Issues, then waits at read-operands for R2 (RAW on I1).

* **I3 (SUB R2)**
  * Stalls at issue due to **WAW** with I1 on R2; cannot proceed until I1 writes.

* **I4 (ADD R8 uses R4)**
  * Independent of I1/I3; can issue if an ADD FU is free.
  * Waits to read R4 until I2 produces it (RAW).

### Hazards handled in this flow

* **RAW**: Blocked at read-operands until sources are ready (e.g., I2 waiting on I1 → R2).
* **WAW**: Blocked at issue when an older instruction targets the same destination (I3 vs I1 on R2).
* **WAR**: Blocked at write-result until all earlier reads have occurred.
* **Structural**: Blocked at issue when the needed FU is busy.

### Key takeaway

* Scoreboarding delays only when hazards are real, allowing independent instructions to proceed.
* No forwarding: reads occur from the register file after writes commit.

# Why scoreboarding can feel “stiff”

* No global broadcast of results to all waiting consumers.
* No register renaming, so **WAR** and **WAW** must be enforced with stalls.
* Multiple long-latency operations cause backpressure at issue and at write-back.

# Tomasulo’s algorithm: three ideas that remove WAR/WAW stalls

Problem to solve: stop stalling for **name** dependences and reduce conservative blocking. Solution = three mechanisms that today underpin OoO cores.

## 1) Reservation Stations (RS)

* Each FU has a small **buffer array** (the RS) that holds pending micro-ops and their operand status.
* If the FU is busy but an RS slot is free, **issue proceeds** into the RS. You stall only when **RS is full**, not whenever the FU is busy.
* Multiple FUs may share a logical queue; designs vary. The concept: **don’t stall front-end issue just because the execution slot is busy**.

## 2) Register Renaming

* Map architectural registers to **transient producer identifiers** so consumers wait on **producers**, not names.
* In Tomasulo, the producer identifiers are **RS tags** (in modern designs: **physical register numbers**).
* Example:

  ```
  I1: R1 = R2 / R3    → allocate RS tag D1 for the divide
  I2: R4 = R1 + R5    → sources: (tag D1, value of R5); waits on D1, not “R1”
  I3: R1 = R8 + R9    → allocate RS tag A7 for the adder; final writer to R1 is A7
  ```

  * I2 will only accept the value **tagged D1**. I3 writing “R1” early is harmless because I2 isn’t reading “R1” by name; it is waiting for **D1**.
  * **WAR eliminated**: a later writer cannot clobber a name the earlier reader needs, because the reader tracks a **tag**, not the architectural name.
  * **WAW eliminated**: multiple writers to the same architectural name each get unique tags; only the **last** mapping becomes architecturally visible at commit.

## 3) Common Data Bus (CDB)

* When an FU finishes, it **broadcasts the result value + tag** on the CDB.
* Any RS entry that has that tag as a missing source **captures the value** and becomes ready.
* This implements **forwarding** across the machine without going through the register file.

> Limitation: a **single** CDB can be a bandwidth bottleneck. Designs may widen or bank the broadcast fabric, or cluster FUs, to allow multiple completions per cycle.

# Putting the mechanisms together

### Issue

* Decode converts architectural sources to either:

  * **ready values**, if available, or
  * **tags** of the producing RS/physical register, if not ready.
* Allocate an RS entry; record:

  * `op` (e.g., ADD, MUL)
  * For each source: `Vj/Vk` (value) **or** `Qj/Qk` (producer tag)
  * Destination tag (this RS or a physical register id)
* Structural stall only when the **RS is full**.

### Execute

* When both sources are values (no `Q*` outstanding) and the FU is free, start execution.
* On completion, **broadcast (tag, value) on CDB**; waiting consumers capture it.

### Write/Commit to architectural state

* Tomasulo-style renaming keeps track of **which tag owns the current architectural name**. Only the instruction that **currently owns** the name updates the architectural register mapping; older producers’ results are ignored for architectural state.
* In modern precise machines, a **Reorder Buffer (ROB)** commits in order; Tomasulo’s original FP unit handled in-order result status to maintain correct final state.

# Example walkthrough (RS tags and renaming)

Example 1 (RAW preserved):

```
LD r0, 0(r7)        →  RS1:  LD    RS1, 0, 0x1000
LD r1, 8(r7)        →  RS2:  LD    RS2, 8, 0x1000
MUL.D r4, r0, r1    →  RS3:  MUL.D RS3, RS1, RS2
```

* `MUL.D` consumes by **tags** (`RS1`, `RS2`), not by architectural names (`r0`, `r1`).
* When `RS1`/`RS2` complete, they broadcast (tag, value); `RS3` wakes up. **RAW is preserved** because consumers bind to producing tags.

Example 2 (WAW avoided by renaming):

```
LD r0, 0(r7)        →  RS1:  LD    RS1, 0, 0x1000
LD r1, 8(r7)        →  RS2:  LD    RS2, 8, 0x1000
MUL.D r4, r0, r1    →  RS3:  MUL.D RS3, RS1, RS2
ADD.D r1, r0, r3    →  RS4:  ADD.D RS4, RS1, 0x16
```

* Two writers to architectural `r1` exist: `RS2` (from the load) and `RS4` (from the add). Renaming assigns each a distinct destination tag.
* The Rename Map makes `r1 → RS4` after `ADD.D` issues; earlier `RS2` may still feed consumers by tag, but it no longer “owns” architectural `r1`.
* At commit, only the instruction that currently owns `r1` in the Rename Map updates architectural state. Thus **WAW is neutralized**.

Q: Which `r1` should be written into the register file?

A: Only the last mapping (from `ADD.D → RS4`). The earlier `LD` to `r1` (`RS2`) can still serve consumers via its tag but must not overwrite architectural `r1` at commit if it no longer owns the name. The ROB enforces in-order commit of the current owner; older, non-owning results are suppressed for architectural writeback.



# Answers to the inline questions

* **Can we “just latch” what we have and wait for the rest?**
  Yes. That is exactly what RS entries do: they store ready operands and hold producer **tags** for the not-yet-ready ones.

* **How do we prevent an old, slow producer from overwriting the final architectural register?**
  Track the **current owner tag** for each architectural name. On completion, only the instruction whose tag matches the current owner is permitted to update architectural state. Others’ results are used only by tag-waiting consumers and then discarded for the architectural file.

* **Why not just add many architectural registers?**
  That enlarges the ISA encoding (more bits per operand), impacts code size and compatibility, and still does not remove **name** hazards when multiple producers target the **same** name in tight windows. **Renaming** adds many internal registers/tags without changing the ISA.

* **Does one shared bus serialize completions?**
  A single CDB does. Real designs solve this with multiple buses, banking, clustering, or selective wakeup to increase completion bandwidth.

# Limitations recap and transition

* **Scoreboarding**: conservative stalls at issue (structural, WAW) and at write-back (WAR), no global forwarding fabric.
* **Tomasulo**: RS + **register renaming** + CDB remove WAR and WAW stalls and allow wide forwarding. RAW is preserved by tags.
* Modern precise OoO cores extend Tomasulo with a **ROB** for in-order commit, plus larger physical-register files and multi-ported wakeup/select logic.

## Tomasulo Step-by-Step Timeline

### Instruction Set

* **i1**: `LD F6, 34(R2)`
* **i2**: `LD F2, 45(R3)`
* **i3**: `MULTD F0, F2, F4`
* **i4**: `SUBD F8, F6, F2`
* **i5**: `DIVD F10, F0, F6`
* **i6**: `ADDD F6, F8, F2`

Latencies:

* LD = 2 cycles
* ADDD/SUBD = 2 cycles
* MULTD = 10 cycles
* DIVD = 40 cycles

---

### Cycle-by-Cycle

**1**: i1 issues. LD starts [1/2].
**2**: i2 issues. LD i1 still executing [2/2].
**3**: i3 issues. i1 completes exec. LD i2 [1/2].
**4**: i4 issues. WB i1→F6. i2 [2/2].
**5**: i5 issues. WB i2→F2. i3 waits for F2, i4 waits for F2.
**6**: i6 issues. i3 starts MULTD [1/10]. i4 starts SUBD [1/2]. i6 waits for F8.
**7**: i4 SUBD [2/2]. i3 [2/10].
**8**: WB i4→F8. i3 [3/10]. i6 can now start.
**9**: i6 ADDD [1/2]. i3 [4/10].
**10**: i6 ADDD [2/2]. i3 [5/10].
**11**: WB i6→F6. i3 [6/10].
**12–14**: i3 [7–9/10].
**15**: i3 [10/10].
**16**: WB i3→F0. i5 gets F0 and F6, can start.
**17–56**: i5 DIVD runs for 40 cycles.
**57**: WB i5→F10.

---

### Key Dependencies

* **i3** waits for F2 from i2 (WB at 5) → can only start at 6.
* **i4** waits for F6 (WB at 4) and F2 (WB at 5) → can only start at 6.
* **i6** waits for F8 from i4 (WB at 8) → starts at 9.
* **i5** waits for F0 from i3 (WB at 16) → starts at 17.


### Tomasulo schedule (given latencies: LD=2, ADD/SUB=2, MULT=10, DIV=40)

| Instr | Operation          | Issue | Exec start | Exec end | Write |
| ----: | ------------------ | :---: | :--------: | :------: | :---: |
|    i1 | `LD F6, 34(R2)`    |   1   |      2     |     3    |   4   |
|    i2 | `LD F2, 45(R3)`    |   2   |      3     |     4    |   5   |
|    i3 | `MULTD F0, F2, F4` |   3   |      6     |    15    |   16  |
|    i4 | `SUBD F8, F6, F2`  |   4   |      6     |     7    |   8   |
|    i5 | `DIVD F10, F0, F6` |   5   |     17     |    56    |   57  |
|    i6 | `ADDD F6, F8, F2`  |   6   |      9     |    10    |   11  |
