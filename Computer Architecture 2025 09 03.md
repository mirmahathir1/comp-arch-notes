# Corrected and expanded notes: 5-stage pipelining and control

## Model and scope

Assume a classic 5-stage, in-order, single-issue RISC pipeline (MIPS-like):
IF (fetch) → ID (decode & register read) → EX (execute/ALU) → MEM (data memory) → WB (write-back).

Where the original wording was ambiguous or incorrect, I correct it explicitly below.

---

# 1) Fetch (IF)

* **What happens**

  * Read instruction from **instruction memory** using the **Program Counter (PC)**.
  * Compute `PC+4` for the next sequential fetch.
* **Why instruction and data memories must be independent**

  * IF and MEM can occur in the same cycle for different instructions. To avoid a structural hazard, use separate I-cache and D-cache (Harvard split) or a multiported memory.
* **Outputs carried forward (IF/ID register)**

  * Fetched instruction bits.
  * `PC` and `PC+4` (needed for PC-relative targets and for link addresses).

---

# 2) Decode & register read (ID)

* **Decoding**

  * Inspect opcode/funct fields to classify instruction and set **control signals** for later stages.
* **Register file access**

  * Read up to two source registers in this stage.
  * Many pipelines implement **WB writes in the first half** of the cycle and **ID reads in the second half**. This avoids a same-cycle read-after-write conflict without extra ports.
* **Immediates**

  * Extract immediates from instruction fields and **sign-extend** (or zero-extend, depending on ISA and instruction).
* **PC-relative target precompute (common optimization)**

  * Add `PC+4` to the sign-extended branch offset (shifted by 2 for word alignment) using a small **separate adder**. This enables early branch target availability.
* **Speculative register read (clarified)**

  * It is safe to read both register operands speculatively in ID, even if some instructions later use an immediate instead of `rt`. A **multiplexer** in EX will select the correct second operand.
* **Key control signals produced by decode (canonical names)**

  * `RegWrite` (write register in WB)
  * `MemRead`, `MemWrite` (for loads/stores in MEM)
  * `MemToReg` (WB selects memory data vs ALU result)
  * `ALUSrc` (EX second input selects **immediate** vs **register**)
  * `ALUOp` (selects add/sub/and/or/shift/compare)
  * `Branch` and `Jump` class signals (steer PC selection logic)

*Correction:* the transcript’s “load register signal” is actually **`RegWrite`**. “AU/au” is **ALU**. “memorite/memory B” are **`MemWrite`/`MemRead`**.

* **Outputs carried forward (ID/EX register)**

  * Read data `rs`, `rt`, the extended immediate, destination register id, control bits, and possibly `PC+4` or precomputed branch target.

---

# 3) Execute (EX)

* **ALU operations**

  * R-type: compute e.g., `R1 = R2 op R3`.
  * I-type arithmetic: `R1 = R2 op imm` when `ALUSrc=1`.

* **Address generation for memory ops (base+displacement)**

  * Effective address = `base (rs) + signext(imm)`. This is computed by the ALU in EX.
  * The result is the **effective/virtual address**. If virtual memory is present, translation to a physical address occurs in MEM alongside the D-cache/TLB, not in EX.

* **Branch decision and target**

  * Two computations are needed:

    1. **Condition** (equality, less-than, ≤0, etc.).
    2. **Target address** (`PC+4 + offset<<2`) or a jump target.
  * Implementations commonly use:

    * A **separate adder** for target calculation, while the ALU evaluates the condition; or
    * Do both in EX if the ALU and an extra adder are available.
  * **Where branches resolve:** Good designs resolve in **ID or EX** to keep penalties low. Resolving at the end of MEM is functionally correct but unnecessarily late and increases the penalty.
    *Correction:* the statement “branches are resolved in MEM” describes a suboptimal variant. The standard 5-stage MIPS resolves branches in ID (classic textbook variant) or EX (also common).

* **Jumps and immediate field widths**

  * **PC-relative branches** typically use a **signed 16-bit** offset (MIPS example) scaled by 4. Range ≈ ±2^15 instructions × 4 bytes = ±128 KB from `PC+4`.
  * **Absolute jump** (MIPS `J`/`JAL`) uses a **26-bit** field. Target = `{PC[31:28], instr_index[25:0], 2'b00}`. This yields a 28-bit aligned range within the current 256 MB region.
    If a target lies outside, use an **indirect jump** via a register (e.g., `JR`) after loading the full address into a register.
  * Other ISAs differ (e.g., RISC-V: 12-bit B-type branches, 20-bit JAL), but the same principles hold.

* **Outputs carried forward (EX/MEM register)**

  * ALU result, store data (forwarded `rt`), branch target, condition outcome, and control bits for MEM/WB.

---

# 4) Memory access (MEM)

* **Loads**

  * Use the effective address. Assert `MemRead`. Access D-cache/memory (with TLB/translation if present). The **read data** goes forward to WB.
* **Stores**

  * Use the effective address. Assert `MemWrite`. Write the **store data** captured from ID (and held in EX/MEM) to memory.
* **Nothing to do for pure ALU ops**

  * ALU instructions simply pass through MEM.

*Correction:* “virtual address used when performing the load/store” is incomplete. The pipeline uses the **effective address**; a TLB translates to a **physical** address for the cache/memory access in this stage on systems with virtual memory.

* **Outputs carried forward (MEM/WB register)**

  * Memory read data (for loads), ALU result (for ALU ops), destination register id, and `MemToReg/RegWrite`.

---

# 5) Write-back (WB)

* **Select source**

  * If `MemToReg=1`: write memory data to the destination register (loads).
  * Else: write ALU result (ALU and address-calc instructions that write).
* **Destination**

  * ISA-specific: e.g., `rt` for I-type loads, `rd` for R-type ALU ops in MIPS.
* **Control**

  * `RegWrite=1` must be set for any instruction that updates the register file.

---

# 6) Multiplexers and operand selection

* **ALUSrc multiplexer**

  * Chooses between `register` and `sign-extended immediate` for the ALU’s second input using the `ALUSrc` control.
* **PC selection multiplexer**

  * Chooses **next PC** among `{PC+4, branch_target if taken, jump_target, register_target (JR)}` using `Branch`, condition result, and `Jump` signals.
* **Write-back data multiplexer**

  * Chooses between `mem_data` and `alu_result` using `MemToReg`.

*Correction:* the transcript’s ad-hoc names map to these standard muxes and control signals.

---

# 7) Pipeline registers (isolation and timing correctness)

* **Purpose**

  * Prevent values for instruction *n+1* from overwriting or “clobbering” values still needed by instruction *n*. They **latch** each stage’s outputs on the clock edge.
* **Locations and contents**

  * **IF/ID**: instruction, `PC`, `PC+4`.
  * **ID/EX**: register operands, extended immediate, destination reg id, ALU control, `ALUSrc`, `MemRead/Write`, `Branch`, `Jump`, `RegWrite`, `MemToReg`.
  * **EX/MEM**: ALU result, store data, branch target, condition, memory controls.
  * **MEM/WB**: memory read data, ALU result, destination register id, `MemToReg`, `RegWrite`.
* **Timing note**

  * Do not reason in “partial nanoseconds” within a stage. The contract is: **combinational logic within a stage settles by the clock edge**, then the next stage sees stable inputs via the pipeline register on the next edge.

---

# 8) Throughput vs latency and CPI

* **Without pipelining**

  * If each stage’s work were done serially, a single instruction would take ~5 cycles. CPI ≈ 5.
* **With ideal pipelining**

  * After the pipeline fills, complete **one instruction per cycle**. **Throughput** improves by ≈ the number of stages. **Latency per instruction** remains ≈ 5 cycles.
* **Real pipelines**

  * Hazards, stalls, and flushes increase CPI above 1. Fill and drain also add a few cycles at boundaries. The “×5” improvement is an upper bound, not a guarantee.

*Correction:* “multiply throughput by exactly the number of stages” is only true in the ideal, hazard-free case.

---

# 9) Hazards and structural conflicts (preview, with precise fixes)

* **Structural hazards**

  * Same resource needed by two stages in the same cycle.
    Fix: split I/D memories or add ports. Use regfile write-first-half/read-second-half or more ports.
* **Data hazards**

  * Read-after-write (RAW): later instruction needs a result that is not yet written back.
    Fix: **forwarding/bypassing** from EX/MEM/MEM/WB to EX inputs, plus **stalls** when forwarding is insufficient (e.g., load-use).
* **Control hazards**

  * Branches and jumps change control flow after some instructions have already been fetched.
  * **Mitigations**

    * **Early resolve** in ID or EX.
    * **Flush** wrong-path instructions by clearing IF/ID (and possibly ID/EX) when a branch is taken.
    * **Static or dynamic branch prediction** to reduce taken-branch penalty.
* **“Bogus instruction” question answer**

  * If you speculatively fetched along the wrong path, you **squash/flush** those instructions: allow their combinational work to occur but **prevent any state updates** (`RegWrite=0`, `MemWrite=0`) and clear their pipeline registers.

---

# 10) Branch mechanics, precisely

* **Equality compare**

  * Use the ALU’s subtract or a dedicated comparator to test `rs == rt` and produce a **Zero** flag.
* **Less-than / ≤0 tests**

  * Use ALU set-less-than or dedicated logic to evaluate conditions like `BLEZ`, `BLTZ`, etc.
* **Target computation**

  * **Branch**: `target = PC+4 + (signext(offset16) << 2)`.
  * **Jump**: `target = {PC[31:28], instr_index[25:0], 2'b00}` (MIPS).
  * **Indirect jump**: `target = register_value`.
* **PC select**

  * Next PC = `branch_target` iff `Branch && condition_true`, else `jump_target` if `Jump`, else `PC+4`.

*Correction:* “both computations cannot use the same ALU in the same cycle” is only true if you require them simultaneously. Designs often add a **small target adder** so the ALU can handle the comparison concurrently.

---

# 11) Loads and stores, precisely

* **Load example**

  * `LW R1, offset(R2)`
    EX: `EA = R2 + signext(offset)`
    MEM: read `Mem[EA]`
    WB: write `R1 = read_data`
* **Store example**

  * `SW R1, offset(R2)`
    EX: `EA = R2 + signext(offset)`
    MEM: write `Mem[EA] = R1`
* **Control**

  * Loads: `MemRead=1`, `MemWrite=0`, `RegWrite=1`, `MemToReg=1`.
  * Stores: `MemRead=0`, `MemWrite=1`, `RegWrite=0`.

---

# 12) Control signal summary (canonical)

* `RegWrite`: write to register file in WB.
* `MemRead` / `MemWrite`: access D-cache in MEM.
* `MemToReg`: WB mux selects memory data (1) vs ALU result (0).
* `ALUSrc`: EX mux selects immediate (1) vs register (0).
* `ALUOp`: selects specific ALU function.
* `Branch`: instruction is a conditional branch.
* `Jump`: instruction is an absolute or register jump.

*Correction:* the transcript’s “PC source” is a **PC-select control** driven by a combination of `Branch`, the condition result, and `Jump`, not just a binary “is branch” flag.

---

# 13) Practical implementation notes

* **Stage balancing**

  * Aim for similar logic delay per stage to maximize clock frequency.
* **Forwarding network**

  * Provide paths EX→EX and MEM→EX for both ALU inputs. Detect load-use and insert a 1-cycle stall when needed.
* **Pipeline register contents are stable for one cycle**

  * Each pipeline register is a bank of edge-triggered flip-flops. Values update only on the clock edge, ensuring isolation across overlapping instructions.

---

# 14) Common misconceptions corrected

* *“Sign executed”* → **sign-extended**.
* *“AU”* → **ALU**.
* *“load register signal”* → **`RegWrite`**.
* *“memory B / memorite”* → **`MemRead` / `MemWrite`**.
* *“Branches resolved in MEM”* → feasible but **not standard**; standard 5-stage resolves in **ID or EX** to reduce penalty.
* *“Virtual address is used directly for memory”* → memory stage uses **effective address**; with virtual memory a **TLB** translates to a **physical** address before the D-cache/DRAM access.

---

# 15) Minimal pipeline timing diagram (steady state, schematic)

```
Cycle:      1     2     3     4     5     6
Instr i:   IF -> ID -> EX -> MEM -> WB
Instr i+1:       IF -> ID -> EX -> MEM -> WB
Instr i+2:             IF -> ID -> EX -> MEM -> WB
...
```

* After fill, CPI ≈ 1 in the ideal case. Latency per instruction ≈ 5 cycles.

---

If you want, I can add hazard detection logic, forwarding paths, and the flush/stall control equations next.

# Corrected and expanded notes: hazards, timing, forwarding, and scheduling

## 1) Ideal speedup vs reality

* Ideal: an **n-stage** pipeline can approach **n× throughput** with CPI ≈ 1 after fill.
* Reality: speedup < n due to

  * **Imbalance**: the slowest stage sets the clock period.
  * **Register overhead**: `t_clk ≥ max(stage_delay) + t_clk-to-Q + t_setup + t_skew + t_jitter`.
  * **Hazards**: structural, data, and control insert bubbles.
* Correction: “increase stages by a factor of n → speedup 10×” is imprecise. Upper bound is **n×**, not guaranteed.

## 2) Pipeline balance

* Goal: partition logic so each stage has similar delay.
* If one stage dominates, the clock is limited by that stage. Throughput falls to that bound.

## 3) Clocking, skew, and jitter

* Modern CPUs are **synchronous**. A global clock drives stage latches.
* **Clock skew** = difference in clock arrival times at two flops. **Jitter** = cycle-to-cycle clock variation.
* **Clock tree synthesis** adds buffers and structured routing to **equalize arrival**, not to raise frequency.
* Correction: “add delay to increase clock frequency” is wrong. Buffers increase insertion delay and margin; they **reduce** skew but do **not** raise fmax. fmax is limited by logic+overheads above.

## 4) Structural hazards

* Definition: two pipeline stages need the **same hardware** in the same cycle.
* Classic case: single-ported unified memory. IF and MEM cannot run together.
* Fixes:

  * **Split I/D memories** (I-cache + D-cache) or
  * **Multiport** the shared structure or
  * **Stall** the younger instruction (insert a bubble).

## 5) Data hazards (RAW, WAR, WAW in context)

* In-order 5-stage has **RAW** hazards. **WAR/WAW** do not occur because writes happen in order at WB.
* Example timeline (no forwarding):

```
Cycle:   1   2   3   4   5
I1=ADD  IF  ID  EX  MEM  WB   ; R1 ready only at end of 5
I2=SUB      IF  ID  EX  MEM  WB
```

* I2 needs R1 in **EX at cycle 4**. Without forwarding, R1 is not yet written in regfile until **end of 5** → stall.

## 6) Register file ports and half-cycle convention

* Typical regfile: **2 read ports + 1 write port**.
* Common convention: **WB write in first half** of cycle. **ID reads in second half**.
  This allows ID to observe a value written in WB in the same cycle.

## 7) Forwarding (bypassing)

* Purpose: supply results **before WB** to remove unnecessary stalls.
* Sources into EX operand muxes:

  * From **EX/MEM.ALUResult** (result produced last cycle).
  * From **MEM/WB.WriteData** (ALU result or load data reaching WB).
* Control: a small **hazard/forwarding unit** compares register numbers in pipeline registers:

  * If `ID/EX.rs == EX/MEM.rd` and `EX/MEM.RegWrite=1` → forward EX/MEM to ALU A.
  * Else if `ID/EX.rs == MEM/WB.rd` and `MEM/WB.RegWrite=1` → forward MEM/WB to ALU A.
  * Repeat for `rt` to ALU B.
* Correction: you do **not** need a per-register “metadata table” in a simple in-order 5-stage. Comparators across **pipeline register fields** suffice.
  Scoreboards/register-status tables appear in **dynamic scheduling/out-of-order** designs.

## 8) Load-use hazard (unavoidable 1-cycle stall)

* `I1: LW R1, 0(R2)` produces data **end of MEM (cycle 4)**.
* `I2: SUB R3, R1, R5` needs R1 **at start of EX (cycle 4)**.
* Even with forwarding, the load’s data arrives too late for I2’s EX. Insert **one bubble** so I2’s EX moves to cycle 5.
* If a cache miss stretches MEM to many cycles, the dependent instruction stalls that many cycles.

## 9) Control hazards (branches and jumps)

* Problem: next PC unknown until the branch resolves.
* Penalty depends on **resolve stage**:

  * ID-resolve or EX-resolve → shorter penalty.
  * MEM-resolve → longer penalty. Functional but suboptimal.
* Remedies:

  * **Early target adder** and **early comparison** (ID/EX).
  * **Flush** wrong-path instructions by clearing IF/ID (and possibly ID/EX) and killing `RegWrite/MemWrite`.
  * **Prediction**: static (e.g., not-taken) or dynamic (2-bit counters, BTB) to keep IF busy.

## 10) Stalls and bubbles precisely

* A **stall** freezes upstream pipeline registers. A **bubble** is an injected NOP that advances.
* Structural hazard example (single memory):

  * IF of In+4 vs MEM of In conflict → hold IF one cycle and inject a bubble so MEM proceeds.

## 11) Example: dependent ALU chain with forwarding

```
Cycle:   1     2     3     4     5     6
I1: ADD IF -> ID -> EX -> MEM -> WB
I2: SUB    IF -> ID -> EXf-> MEM -> WB    ; EX uses EX/MEM forward from I1
I3: OR        IF -> ID -> EXf-> MEM -> WB ; EX uses MEM/WB forward if needed
```

* `EXf` indicates an EX operand taken from a forwarding path.

## 12) Example: load-use with required stall

```
Cycle:   1     2     3     4     5     6
I1: LW  IF -> ID -> EX -> MEM -> WB
I2: ADD    IF -> ID -> ST  -> EXf-> MEM -> WB   ; ST = bubble
```

* One bubble ensures I2’s EX sees the load data forwarded from MEM/WB.

## 13) Compiler scheduling vs hardware stalls

* **Correctness**: hardware must always insert stalls/flushes when needed. The ISA must not require programmers to count pipeline stages.
* **Performance**: the compiler may **reorder independent instructions** to fill hazard slots. This lowers CPI but is optional.
* Correction: this does **not** contradict hardware responsibility. Compiler scheduling is an **optimization**, not a correctness requirement.

## 14) Reordering example (minimize stalls)

* Original (conceptual):
  `LW R1, 0(R2)`
  `LW R7, 0(R1)`        ; uses R1 for address → depends on prior LW
  `ADD R4, R1, R8`      ; uses loaded R1 → load-use
  `SUB R9, R5, R6`      ; independent
* Better order:

  * Move `SUB` **between the two loads** to separate them.
  * Move `ADD` **after** the second load with at least one independent op in between if available.
* Result: fewer or zero stalls if independent work exists to cover the required bubble.

## 15) ILP and more advanced machines

* The simple 5-stage is **in-order**, single-issue. It exposes limited ILP.
* **Dynamic scheduling / out-of-order** can let an independent later instruction bypass a stalled dependent one.
* **VLIW/EPIC** shifts scheduling to the **compiler** (e.g., Itanium). Works only when the binary matches the microarchitecture’s latencies and resources.

## 16) Precise control signals recap (canonical names)

* `RegWrite`, `MemRead`, `MemWrite`, `MemToReg`, `ALUSrc`, `ALUOp`, `Branch`, `Jump`.
* PC select mux chooses among `{PC+4, branch_target if taken, jump_target, register_target}`.

## 17) Where values live each cycle (5-stage, with forwarding)

* **IF/ID**: instruction, `PC+4`.
* **ID/EX**: `rs`, `rt`, `imm`, `rd/rt`, control bits.
* **EX/MEM**: `ALUResult`, `store_data`, `branch_target`, condition.
* **MEM/WB**: `ReadData` (for loads), `ALUResult`, destination reg id.

## 18) Quantifying CPI with hazards

* `CPI ≈ 1 + (bubbles per committed instruction)`.
* Sources of bubbles:

  * Structural conflicts (avoidable by design).
  * **1-cycle** for each load-use that the scheduler cannot hide.
  * Control penalties on taken branches that prediction cannot hide.

## 19) Terminology and transcription fixes

* “AU/au” → **ALU**.
* “Sign executed” → **sign-extended**.
* “PC source” → **PC select control** driven by `Branch`, condition, and `Jump`.
* “End time speed up by factor n” → **ideal ≈ n× throughput**, not guaranteed.
* “Add delay to raise frequency” → **false**; buffers reduce skew, not fmax.
* “Metadata in regfile for forwarding” → unnecessary in 5-stage; use **pipeline register comparisons**.
* “States” when counting pipeline depth → **stages**.
* “Fed stage” → **fetch stage**.
* “Dynamic scheduleuling / bandwidth scheduleuling” → **dynamic scheduling**.

## 20) Design rules of thumb

* Balance stage delays.
* Keep IF and MEM independent (Harvard split).
* Implement EX/MEM and MEM/WB forwarding plus a **load-use stall**.
* Resolve branches early and flush on mispredict.
* Use regfile **write-first-half / read-second-half** to avoid unnecessary stalls.
* Treat compiler scheduling as a performance aid, not a contract.
