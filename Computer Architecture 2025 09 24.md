# Baseline Tomasulo’s algorithm (what it actually is)

* Core pieces: reservation stations (RS) near each functional unit, a common data bus (CDB) for result broadcast, and a register alias table (RAT) that maps architectural registers to producer tags.
* Original Tomasulo did **not** include a reorder buffer (ROB). It allowed out-of-order execution and in-order completion of each instruction’s *writeback to the register file* via tag-based wakeup/broadcast, but it did **not** by itself guarantee precise exceptions or easy branch recovery.

# Why add a Reorder Buffer (ROB)

* Precise state at commit: even if execution is out of order, the ROB enforces **in-order commit**. This gives precise exceptions, easy recovery on branch mispredicts, and clean interaction with the architectural register file and memory.
* Where results live: a common design writes results into the ROB entry at execution finish, then copies to the architectural register file at commit.
* Naming with ROB vs. physical registers: two common variants:

  * **ROB-as-name**: the destination tag is the ROB index. Operands carry ROB tags until values appear on the CDB.
  * **Physical register file (PRF)**: rename to physical regs; the ROB holds a pointer to the dest physical reg and commit updates the architectural mapping. This is the dominant modern design.

# Why you still need Reservation Stations with a ROB

* RSs are **staging buffers near the functional units**. They hold ready operands and outstanding tags so units can start the cycle operands become ready.
* The ROB orders and commits; it is **not** a good place to stage per-FU ready operands. Physically and logically, RSs reduce wire delay and fan-out on operand delivery and allow many instructions to wait close to their target units.
* You could, in theory, issue directly out of the ROB, but it scales poorly in wiring, ports, and timing.

# Loads and stores out of order (the accurate constraints)

* You may execute loads and stores out of order **subject to memory dependences**. You must not violate:

  * RAW: a load must get the most recent older store to the same address.
  * WAW/WAR: stores must commit to memory in program order.
* Hardware structures:

  * **Load/Store Queue (LSQ)** or separate **Load Queue (LQ)** and **Store Queue (SQ)**, ordered by program order.
  * **Store-to-load forwarding** from older stores with known addresses.
  * **Memory dependence prediction / disambiguation**: allow a load to speculatively bypass older stores whose addresses are unknown; if a conflict is later detected, squash and re-execute the load and its dependents.
* So the correct statement is: “Loads and stores can execute out of order when the LSQ enforces dependences; they always **commit** to architectural state in order.”

# Superscalar goals and pipeline bandwidth

* **Issue width** (K): max instructions sent to rename/RS per cycle. IPC can exceed 1 if the front end and back end sustain bandwidth.
* To support width (K), the machine needs, in the steady state:

  * Fetch ≈ (K) instructions/cycle.
  * Decode/rename ≈ (K)/cycle.
  * Issue/wakeup/select ≈ (K)/cycle.
  * Execute enough units to keep ≈ (K) ops finishing per cycle (mix-dependent).
  * Writeback up to ≈ (K)/cycle.
  * Commit up to ≈ (K)/cycle.
* Any stage that falls below (K) becomes the bottleneck.

# Fetching across cache-line boundaries

Assumptions used in the lecture example: fixed 4-byte instructions; I-cache line = 32 B ⇒ 8 instructions/line.

**Problem:** With width 4, the next 4 instructions may straddle two cache lines. A single-ported I-cache delivers only one line per cycle, so you’d need 2 cycles.

**Standard solutions:**

1. **Banked / interleaved I-cache**: two or more banks with addresses interleaved across banks. Consecutive lines map to different banks, so you can fetch two lines in one cycle if they’re in distinct banks. This is the common fix.
2. **Wider I-cache line** or **double-line fetch**: fetch 64 B or fetch two adjacent lines each cycle; needs bandwidth and tag/valid support.
3. **Instruction alignment buffer**: a small buffer that re-packages bytes/words into aligned instruction groups spanning line boundaries. Essential for variable-length ISAs; still useful with fixed-length.

Compiler “force alignment” does not solve branch targets that land arbitrarily; hardware must handle boundaries.

# Branches during wide fetch

Even with a BTB, if one of the (K) fetched instructions is a taken branch whose target is not in the same next-line window, you risk under-fetch.

**Industry techniques:**

* **BTB (branch target buffer)**: indexed by PC, returns predicted-taken target address. Lets the fetch unit redirect on the *next* cycle. Good but may still cap instantaneous width if the target sits in a different line not fetched this cycle.
* **Multi-block/dual-path fetch**: fetch the fall-through block *and* the predicted target block in parallel if banks/ports allow; select the correct path next cycle. Costs bandwidth.
* **Branch Target *Instruction* Cache (BTIC/BTC)**: a small cache that stores a few instructions at the predicted target so the fetch unit can deliver target bytes in the same cycle it detects a taken branch. Some ARM/MIPS parts did this.
* **Trace cache / µop cache**:

  * **Trace cache**: stores *dynamic* instruction sequences along predicted paths (e.g., loop body skipping rarely taken if-else). Intel Pentium 4 used a trace cache of decoded µops.
  * **Decoded µop cache** (e.g., Intel Core, AMD Zen): stores fixed-size decoded µops by block, not full cross-branch traces, but solves the same front-end bandwidth problem by bypassing decode and sometimes line-boundary issues.
* **Fetch target queue (FTQ)**: decouples prediction from cache access; allows the predictor to request multiple future blocks so the I-cache can prefetch/prepare them.

Note: the lecture’s “branch target cache that stores the **instructions**” corresponds to BTIC/µop cache. The **BTB** itself stores **addresses**, not instructions.

# Trace cache mechanics (corrected)

* Key idea: store a sequence of (decoded) instructions/µops **in predicted execution order**, including taken/not-taken outcomes for the branches in the trace.
* Indexing: by the first PC plus the branch outcome pattern. A hit requires that the predictor’s current outcomes match the stored pattern.
* Fill: on a miss, fetch/decode down the predicted path to build the trace; first few iterations of a loop may miss, subsequent iterations hit.
* Recovery: on a misprediction, you invalidate or partially invalidate only the younger portion that depended on the wrong outcome.
* Citation fix: the classic paper is **Eric Rotenberg**, Steve Bennett, and James E. Smith, “Trace Cache: A Low Latency Approach to High Bandwidth Instruction Fetching” (1996). The name is Rotenberg, not “Rottenberg.”

# Decode and execution width

* **Decode**: you need multiple parallel decoders. For variable-length ISAs (e.g., x86), you also need a predecoder and an alignment network to split bytes into instructions for each decoder.
* **Execution**: replicate functional units according to the expected mix (e.g., several INT ALUs, fewer FP units, one or more load/store pipes). Replication was made feasible by **Moore’s law**, but ports and bypass networks, not area, usually limit scaling.

# Issuing (n) instructions per cycle: real hazards and logic

Issuing a bundle ({I_1,\dots,I_n}) in the same cycle requires resolving **intra-bundle** dependences during rename/issue.

**Types of dependences (architectural registers):**

* **RAW** (true): later instruction reads a register written by an earlier instruction.
* **WAW** and **WAR** (name/false): eliminated by register renaming, provided you rename in program order.

**Rename in program order, same cycle:**

* For each dest, allocate a fresh tag (ROB entry or physical reg).
* Update the RAT after each instruction so later instructions in the same bundle see the newest mapping.
* If (I_j) reads R1 and (I_i) (with (i<j)) writes R1 in the same bundle, (I_j) must get (I_i)’s *new* tag, not the old mapping. This is handled by sequential RAT updates or a bypassed “rename CAM”.

**Comparator complexity (RAW checks) if you don’t rely purely on sequential RAT updates):**

* Each dest of earlier instructions must be compared to each source of later instructions.
* With at most 1 dest and 2 sources per instruction, number of dest-to-source comparisons per bundle =
  ( \sum_{i=1}^{n} 2,(n-i) = n(n-1) ).
* If you also explicitly detect WAW and WAR without trusting rename order, you add:

  * dest-to-dest: (n(n-1)/2) comparisons (WAW).
  * source-to-later-dest: another (n(n-1)) (WAR).
* In practice, **sequential RAT update in program order** removes the need for extra WAW/WAR compare trees; only the RAW “see-newest” mapping matters, which the sequential update naturally handles.

**Other bandwidth constraints while issuing (n) per cycle:**

* **Rename table ports**: (2n) source lookups + (n) dest updates per cycle.
* **Free list** (PRF designs): allocate (n) physical regs per cycle; free up to (n) per cycle at commit.
* **ROB**: allocate (n) entries/cycle; retire up to (n)/cycle.
* **RS/LSQ**: insert (n) entries/cycle; pick up to (n) ready ops/cycle.
* **Register file ports**: if operands are read from a central PRF, you need many read ports; designs often use operand capture into RSs and rely on broadcasts/forwarding to reduce PRF read pressure.
* **Result buses**: a single CDB does not scale. Wider machines use **multiple result buses** or segmented/banked bypass networks.

# Example: issuing a 4-wide bundle

Assume instructions:
I1: R1 = R2 + R3
I2: R4 = R1 + R5
I3: R1 = R6 + R7
I4: R8 = R1 + R9

Rename in program order with PRF:

1. I1 alloc P10 for R1; sources read RAT[R2], RAT[R3]. RAT[R1] ← P10.
2. I2 sees RAT[R1] = P10, so I2 reads P10 and RAT[R5]; dest R4 gets P11; RAT[R4] ← P11.
3. I3 overwrites R1 again, alloc P12; RAT[R1] ← P12. There is no WAW hazard at the physical level; I3 simply produces a newer version.
4. I4 reads RAT[R1] which is now **P12** (not P10). Correct intra-bundle RAW resolution is achieved by sequential RAT updates. Dest R8 gets P13; RAT[R8] ← P13.

Issue to RSs with tags P10/P12; wakeup/select starts I2 only after P10 is produced; I4 waits on P12, not P10. No compare tree beyond RAT sequencing is needed.

# Branch prediction breadth for wide fetch

* Wide front ends often predict **multiple branches per cycle** (at least the first taken) using a cascaded predictor or a “lookahead” predictor.
* The FTQ + multi-block fetch lets you stage multiple target blocks early.
* Without multi-branch capability, a single predicted taken branch in the group can cap delivered width.

# Store commit and memory ordering

* Even if a store executes early (address computed, data ready), it typically **does not update L1D** until it reaches the ROB head and commits. Until then, it resides in the **store queue** and forwards to younger loads as needed.
* This keeps memory state precise at traps and mispredictions.

# Corrections to specific transcript statements

* “Baseline Tomasulo had no way to perform branch prediction.”
  Correct nuance: Tomasulo itself is an execution/rename scheme. Branch prediction is **orthogonal**. You can add a predictor and BTB to any front end, Tomasulo or not. The point is that **without a ROB** it’s hard to get **precise recovery** on mispredictions; adding a ROB makes speculation practical.
* “Loads and stores have to be performed in order.”
  Incorrect. They may execute out of order with an LSQ and proper dependence enforcement; they **commit** in order.
* “Use ROB IDs instead of reservation-station IDs for renaming.”
  One valid variant. Modern designs typically rename to a physical register file and let the ROB hold pointers. The essential property is unique dest tags and in-order commit; the specific tag source (ROB vs. PRF index) is a design choice.
* “Branch target buffer that stores instructions.”
  A **BTB** stores target **addresses**. A small cache that stores **target instructions** is a **BTIC** or a **µop cache**.
* “Trace cache details and author.”
  Name is **Rotenberg**; mechanism requires matching stored branch outcomes to current predictions; you don’t necessarily flush the entire trace cache on a mispredict, only paths dependent on the wrong outcome.
* “How many dependency checks to issue (n) per cycle? n−1.”
  Incorrect. With 1 dest and 2 sources per instruction, dest-to-source RAW comparisons are (n(n−1)). Rename-in-order avoids needing an explicit full compare net, but the naive pairwise count is (O(n^2)), not (O(n)).

# Key structures summarized by role (for clarity, not to summarize content away)

* **BTB**: predicted target address for control-flow instructions.
* **BTIC/µop cache/trace cache**: deliver instructions or decoded µops at predicted target paths to maintain width.
* **FTQ**: queues predicted fetch blocks to decouple prediction and cache access.
* **RAT**: architectural→tag (ROB or physical) mapping; updated in program order.
* **ROB**: in-order commit, precise exceptions, branch recovery; may buffer results.
* **RS**: per-FU operand/tag buffers for ready/soon-ready ops.
* **LSQ (LQ+SQ)**: preserves memory ordering constraints; enables store-to-load forwarding and speculation.
* **Result buses/bypass**: carry produced values/tags to RSs without waiting for PRF reads.

Here is a corrected, polished, and expanded version in sections. I fix factual and logical errors and explain each point; I do not compress or summarize away content.

# Multi-issue dependence analysis: what must be checked

* For a bundle of (n) instructions issued in the same cycle, the number of **instruction pairs** is (\binom{n}{2}=n(n-1)/2).
* Real RAW checks compare each **earlier dest** against each **later source**. With two sources per instruction, the naive comparator count is (2\cdot \binom{n}{2}=n(n-1)). Asymptotically this is (O(n^2)).
* Practical designs avoid an explicit compare matrix by **renaming in program order** and updating the RAT sequentially within the cycle. Later instructions see the newest mapping, which implicitly handles intra-bundle RAW/WAW/WAR. The datapath still scales in complexity and ports with (n).

# Implementing wide rename/issue without “half-cycle tricks”

* Splitting a clock into “mini-cycles” is not a scalable answer beyond 2-wide. Timing closure and fan-out become the limit.
* Modern cores **pipeline rename/issue** into several stages, each stage handling the **entire bundle**:

  1. Allocate ROB entries / physical registers and RS slots for all (n).
  2. Perform sequential RAT updates across the (n) instructions to encode intra-bundle dependences.
  3. Generate RS operands: read ready values, latch tags for not-ready sources.
* After fill, a separate **wakeup/select** stage picks ready RS entries each cycle.
* Key bandwidth needs per cycle:

  * RAT lookups ≈ (2n), RAT updates ≈ (n).
  * ROB alloc/retire ≈ (n)/(n).
  * Free-list alloc/free ≈ (n)/(n) (PRF designs).
  * RS/LSQ insert ≈ (n).
  * Result broadcast capacity sufficient for multiple completions per cycle.

# Register file ports and data delivery

* If you read operands directly from a central register file, ports scale roughly with width: read ports ≈ (n\cdot r) (r = sources per inst), write ports ≈ (n). This explodes in area and energy.
* Common mitigations:

  * **Operand capture** into RS entries so many consumers get data via **broadcast** (bypass) instead of repeated RF reads.
  * **Banked/segmented physical register files** near clusters of units to limit wire length and port count.
  * **Multiple result buses / segmented bypass networks**. A single “common data bus” does not scale; designs use several buses and locality-aware forwarding.

# Out-of-order memory: correct rules

* Loads and stores may **execute** out of order when a Load/Store Queue enforces dependences and provides **store-to-load forwarding** from older stores to the same address.
* Memory becomes architecturally visible **in order** at commit. This preserves precise exceptions and recovery.

# Dual-issue in-order example: what stalls and what doesn’t

* In a **dual-issue in-order** machine, two independent integer ALU ops can issue together if structural rules permit.
* If the second instruction reads the first instruction’s result, two cases:

  * **Single-issue** pipeline with forwarding: no stall if producer latency is one and write/read phases are arranged.
  * **Dual-issue same cycle**: one of the two executes a cycle later to receive the forwarded result. **Issue** of the pair can still proceed; the **execute** stage of one pipe bubbles for a cycle.
* Branches can still stall the **front end** if prediction/target resolution latency exceeds the fetch cadence, even if decode/issue logic is otherwise flowing.

# Branch prediction and fetch timing: precise statements

* A **BTB** (branch target buffer) stores predicted **target addresses**, not instructions.
* Some cores add a small **branch-target instruction/uop cache** so that upon detecting a predicted-taken branch, a few target instructions/uops are delivered in the **same cycle**. This keeps fetch width up when a taken branch splits the fetch window.
* Even with a correct prediction, if the **target address** is not available until a later pipeline stage, the machine incurs fetch bubbles equal to the predictor latency to target-supply.
* Returns and indirect branches often consult a **return address stack** and a separate **indirect predictor**; these can have longer lookup paths and thus higher minimal latency, again causing bubbles even on correct predictions.

# Case study corrected: ARM Cortex-A53 (what is true)

* **Architecture class**: ARMv8-A, in-order, **dual-issue** superscalar core.
* **Pipeline**: integer pipeline roughly **8 stages**. Two instructions per cycle can be fetched/decoded/issued when hazards allow. It is **not** out-of-order.
* **Branching**: uses standard components such as BTB/BTAC, dynamic predictor, and return stack. Exact table sizes and cycle-by-cycle predictor timing vary by implementation; claiming “one-entry branch target cache” is incorrect.
* **Behavior**: if an early instruction stalls (e.g., waiting on a load), younger independent instructions generally **cannot** pass it; this is the defining trait of in-order issue/execute.
* **Deployment**: widely used in mid-2010s phones and SBCs (e.g., Raspberry Pi 3). Some phones paired A53 “LITTLE” clusters with big out-of-order cores in big.LITTLE systems.
* **Takeaway**: dual-issue improves best-case IPC, but in-order execution still suffers on dependence chains, cache misses, and branch penalties.

# Case study corrected: Intel Core-family “i7-class” core

* **Out-of-order** and speculative. Typical **4-wide** front end delivers up to ~4 decoded µops per cycle from decoders or a **decoded µop cache**. Widths vary by generation for rename/dispatch/issue/retire.
* **x86 decode**:

  * **Macro-op fusion**: merges patterns like CMP/TEST + Jcc into a single fused branch µop.
  * **Micro-fusion** and **microcode**: complex CISC instructions split into one or more µops; very complex ones invoke microcode sequences.
  * **Decoded µop cache**: supplies µops directly on hits, bypassing heavy decode and smoothing width across line and branch boundaries.
* **Predictors**: large BTBs, hybrid directional predictors, return stacks, and indirect predictors operate in concert. Vendor specifics are proprietary; do not ascribe a particular academic predictor (e.g., “TAGE”) to Intel without evidence.
* **Back end**: physical register file + ROB, reservation stations, clustered/banked bypass networks, multiple load/store pipes, and in-order commit.

# Branch pipeline discussion: cycle counts corrected

* Avoid hard claims like “branch resolves at stage 9” or “always a 2- or 3-cycle penalty even when correct” unless tied to a specific published microarchitecture. Real designs overlap predictor lookup with fetch and vary staging.
* The general, accurate rule: **penalty on correct prediction equals the predictor path’s additional latency to produce a usable target compared to straight-line fetch**. Penalty on misprediction equals **flush + refetch latency** from the resolution point back to fetch.

# Correct answers to posed questions

* **How many dependence checks?** Pairwise instruction relationships are (\binom{n}{2}). If you build explicit comparators, RAW checks with two sources scale as (n(n-1)). Rename-in-order implements the same effect without a full compare mesh.
* **Why ROB + RS?** ROB gives precise, in-order commit and recovery. RSs are near FUs to hold ready operands/tags and start work the cycle data arrives.
* **Can we split the issue cycle?** Not beyond small widths. The scalable solution is a **pipelined** multi-stage rename/issue that processes the whole bundle each stage.
* **Do we need “n CDBs”?** No single shared bus scales. Use multiple result buses and locality-aware, often clustered, bypass networks.
