# Branch Target Buffer (BTB), Branch Predictors, and Returns

## BTB vs. Branch Predictor

* **BTB**: A cache keyed by the branch’s PC that supplies a **predicted target address** (used for taken conditional branches, unconditional jumps/calls, and many indirect branches).
* **Branch predictor**: A separate structure that predicts **direction** (taken/not-taken). Many designs also predict indirect targets with a dedicated **indirect predictor**.
* **Access timing**: BTB is consulted very early in fetch. Direction prediction may come from a parallel predictor. They are **logically distinct**.

## When a BTB is needed

* Conditional branch predicted **not taken** → next PC = **PC + 4**. No BTB target needed.
* Conditional branch predicted **taken**, unconditional jumps/calls → need a **target** from BTB (or decoder-immediate for PC-relative forms).

## Returns and multiple call sites

* `ret` is an **unconditional indirect branch**. The correct target depends on **the most recent call**.
* A BTB with **one target per PC** performs poorly for `ret` from functions with **many call sites**.
* Modern CPUs use a **Return Address Stack (RAS)**: on `call`, push `PC+4`; on `ret`, pop and use that as the target. This solves the “multiple call sites” issue efficiently.
* Indirect branches other than returns may have **multiple targets**; a simple 1-target BTB will thrash. Indirect predictors track **per-branch target sets**.

## Replacement and misprediction

* If a BTB entry is wrong or missing:

  * Fetch uses the predicted target (if any).
  * On execute/resolve, if the real target differs, the pipeline **flushes** and fetch restarts at the correct PC.
* Many BTBs use simple **LRU/NRU** replacement; last-used target will overwrite prior state unless a multi-target scheme is in place.

---

# Tomasulo’s Algorithm: Context and Core Ideas

## Historical context

* **CDC 6600** (1964, Seymour Cray): out-of-order-like **scoreboarding**, multiple FUs.
* **IBM 7030 “Stretch”** (early 1960s): early **pipelining** techniques.
* **IBM System/360 Model 91** (late 1960s): introduced **Tomasulo’s algorithm** (register renaming with reservation stations and a common data bus) for high-performance floating-point.
* **CDC 7600** (1969): successor to 6600; outperformed the 360/91 on many workloads.
* Later eras: **vector processing** (Cray systems), **shared-memory multiprocessing**, and modern **GPU-accelerated** supercomputers. The **TOP500** list began in 1993.

## The three pillars

1. **Reservation stations (RS)**: Buffers per functional unit to hold issued instructions and operands or **tags** of producers.
2. **Register renaming**: Map architectural registers → **physical producers** (RS IDs or physical regs) to eliminate **WAW** and **WAR** hazards. **RAW** hazards still honored.
3. **Common Data Bus (CDB)**: When an instruction completes, it **broadcasts** `<tag, value>`. Any RS or the register file waiting on that tag **captures** the value.

---

# Detailed Walkthrough (corrected and precise)

## Issuing an instruction

* Example: `R3 = R1 + R2`

  * Consult the **rename table** for `R1` and `R2`.
  * If a source has **no pending producer**, read its **value** from the register file and store it in the RS operand field.
  * If a source has a **pending producer tag** (e.g., `RS_mul1`), store the **tag** in the RS operand field. Do **not** read the stale register-file value.
  * Allocate an RS for the add (e.g., `RS_add1`) and **rename the destination** `R3` to `RS_add1` in the rename table.

## Dependency example (renaming removes false hazards)

* Suppose:

  * `L1: R0 <- MEM[...]   ⇒ renamed to RS1`
  * `L2: R1 <- MEM[...]   ⇒ renamed to RS2`
  * `MUL: R4 <- R0 * R1   ⇒ allocate RS3; sources are tags RS1, RS2`
  * `ADD: R1 <- ...       ⇒ allocate RS4; destination rename R1→RS4`
* **Outcome**:

  * `MUL` waits for **RS1** and **RS2** on the CDB.
  * Even if `ADD` (writing to architectural `R1`) finishes **before** `L2`, there is **no interference**: `MUL` still depends on **RS2**, not the architectural name `R1`.
  * **WAW/WAR** hazards vanish; only **RAW** is enforced via tags.

## Write-back and “who updates the register file?”

* The register file has, for each architectural register, the **current producer tag** from the rename table.
* When a result `<tag, value>` appears on the CDB:

  * Any RS waiting on `tag` captures the value.
  * The register file updates **only if** its current producer tag **equals** `tag`. Older producers’ broadcasts are **ignored**, so only the latest mapped writer updates the register file.

## Rename table vs. “pending bits”

* The essential state: for each architectural register, either

  * **No pending producer** → read register file value, or
  * **Pending producer tag** → wait for that tag on the CDB.
* Implementations vary: some store the tag in a **separate rename table**, some co-locate metadata with the register file. The behavior above is required; the storage choice is an implementation detail.

---

# Loads, Stores, and Memory Disambiguation

## Load/Store Queue (LSQ)

* Loads and stores also use **reservation stations/buffers**, typically organized as a **Load/Store Queue** that preserves **program order** among memory ops.
* **Store**: must eventually commit its **address and data** to memory in **program order**.
* **Load**: may execute **before** it reaches the head if it is safe w.r.t. **older stores**.

## Why care about order?

* Example:

  * `STORE [R1+1000] = RE` (address depends on `R1`)
  * `LOAD  R8 = [R0+1000]` (address known early; `R0=0`)
* If `R1` later resolves to `0`, both refer to **the same address** `+1000`.
  Executing the **load early** could read the **old memory value** instead of the store’s value. That violates **RAW through memory**.

## Safe early loads

* A load can issue early if **all older stores** either

  * Have computed addresses and **none match** the load’s address, or
  * Provide the **forwarded data** if an address matches.
* If any older store has an **unknown address**, a **conservative** design stalls the load. More **aggressive** designs use a **memory disambiguation predictor** to guess “no alias,” execute the load, and **squash/replay** if later proven wrong.

---

# Control Dependences

* Tomasulo’s dataflow machinery is **orthogonal** to branch handling.
* A baseline presentation may require **prior branches to resolve** before issuing younger instructions. Real processors pair Tomasulo-like scheduling with **branch prediction** so younger instructions can be fetched/issued **speculatively** and squashed on mispredict.

---

# Quick Clarifications to Earlier Questions

* **“Only one return address stored?”**
  In a plain BTB, yes per-PC you often have **one target**. For returns this fails; use a **RAS** that tracks **many** nested calls.
* **“Does the BTB overwrite with the most recent?”**
  Often yes under simple replacement, unless a multi-target or set-associative design retains more history.
* **“Incorrect BTB target?”**
  Fetch proceeds down the predicted path. On resolution, mismatch triggers **flush** and refetch at the correct target.
* **“Loads/stores executed strictly one-by-one?”**
  Not required. They are **queued in order**, but **multiple** can be in flight. Loads may **bypass** older stores when proven safe or predicted safe with **disambiguation**.

---

# Terminology and Names

* **Tomasulo’s algorithm** (Robert Tomasulo), not “Thomas/tomosulos/tomosuro.”
* **CDC 6600**, **CDC 7600**; **IBM System/360 Model 91**.
* **Dennard scaling**, not “Denard,” refers to later transistor scaling theory; historical clock gains for those specific machines came from **design and technology**, not Dennard-era scaling.

# Homework 2.3 constraints

* Do **not** change the number of instructions or their semantic order.
* If the problem states “branch resolves in stage X,” use **that** stage. Do not move branch resolution across stages for the homework answers.

---

# Where to resolve a branch in a pipeline

* You can architecturally resolve a branch in **ID/Decode**, **EX**, or **MEM**.
* Earlier resolution lowers control hazard penalty but needs more decode/compare/target hardware earlier in the pipe.
* Later resolution saves early-stage area/complexity but increases mispredict squash distance.
* For the assignment, follow the **given** stage.

---

# Tomasulo + branch prediction: what is missing and why

* Plain Tomasulo (as first taught) **executes and writes back out of order**. Without a commit barrier, once a value is broadcast on the **Common Data Bus (CDB)** it can update consumers and the architectural map immediately.
* With speculation, a mispredicted-path instruction could **publish** results to other RS entries or the architectural state. Once published, it is hard to “un-publish.”
* Solution used in real CPUs:

  * **Reorder Buffer (ROB)** for **in-order commit** and precise exceptions.
  * Results write to the **ROB entry** first; the architectural register file updates **only at commit** and only for instructions at the ROB head on the correct path.
  * On a mispredict, the ROB squashes all **younger** speculative instructions and their effects, restoring rename state.
* Short mnemonic: **In-order issue, out-of-order execute, in-order commit.** The commit step enables safe branch prediction with Tomasulo-style execution.

---

# Common Data Bus (CDB) mechanics

* The CDB is a broadcast fabric that carries **〈tag, value〉**. The tag is the producing **RS/physical-register ID**.
* Any RS entry whose operand tag matches the broadcast tag **captures** the value. The register file (or ROB) also captures it **if** its current producer tag matches.
* **Same-cycle read-after-write**: designs usually permit a producer to broadcast and waiting consumers to latch in the **same cycle** (often implemented as write in first phase, read in second phase).
* **Arbitration**: if multiple FUs complete in the same cycle and there is one CDB, they **arbitrate**; only one broadcasts. Wide superscalar designs often use **multiple result buses/ports** or banked fabrics.
* The CDB is the vehicle for **forwarding** between producers and consumers without going through the architectural register file.

---

# Area and scaling limits

* Adding more **functional units**, **RS entries**, and **rename-table** capacity is usually not the dominant limit.
* The critical scaling bottleneck is **wakeup/select**: every completing tag must be compared against many RS entries, and many RS entries must contend for a few FU issue slots. This creates **O(N²)**-like comparator and wiring pressure and raises cycle time and energy.

---

# Worked example: Tomasulo schedule

**Assumptions**

* Latencies (EX cycles only): `LOAD=2`, `ADD/SUB=2`, `MUL=10`, `DIV=40`.
* Single-cycle CDB broadcast; consumers can latch in the **same cycle**.
* Plenty of RS entries; loads have an LSQ but no address conflicts.
* Floating-point registers: `F*`.
* No branch speculation in this example.

**Instruction sequence**

1. `L.D   F6, 34(R2)`
2. `L.D   F2, 45(R3)`
3. `MUL.D F0, F2, F4`      ← depends on (2) and F4
4. `SUB.D F8, F6, F2`      ← depends on (1) and (2)
5. `DIV.D F10, F0, F6`     ← depends on (3) and (1)
6. `ADD.D F12, F8, F2`     ← depends on (4) and (2)

**Cycle-by-cycle (IS=Issue, EX=Execute cycles inclusive, WB=Broadcast on CDB)**

| Inst                | IS | EX start–end |     WB | Notes                                 |
| ------------------- | -: | -----------: | -----: | ------------------------------------- |
| 1 `L.D F6,34(R2)`   |  1 |          2–3 |  **4** | F6 ready at c4                        |
| 2 `L.D F2,45(R3)`   |  2 |          3–4 |  **5** | F2 ready at c5                        |
| 3 `MUL.D F0,F2,F4`  |  3 |     **6–15** | **16** | Waits for F2 at c5, F4 already ready  |
| 4 `SUB.D F8,F6,F2`  |  4 |      **6–7** |  **8** | Captures F6 at c4 and F2 at c5        |
| 5 `DIV.D F10,F0,F6` |  5 |    **17–56** | **57** | Waits for F0 at c16; F6 already ready |
| 6 `ADD.D F12,F8,F2` |  6 |     **9–10** | **11** | Waits for F8 at c8; F2 already ready  |


