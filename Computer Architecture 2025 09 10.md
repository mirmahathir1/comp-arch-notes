# 1) Why branch prediction matters

Branches limit pipeline throughput. A taken branch with an unknown direction or target stalls fetch. Good prediction lowers control hazards and raises IPC.

# 2) Static vs dynamic prediction

* **Static prediction:** Fixed at compile time or by a simple rule. Examples: always-not-taken, always-taken, “backward taken, forward not taken,” or ISA hint bits set by the compiler. No runtime learning. Works when a branch is strongly biased.
* **Dynamic prediction:** Hardware learns at run time. A table of small state machines records past outcomes and predicts the next one. Removes burden from the compiler and adapts to program behavior.

# 3) One-bit (1b) predictor: definition and implementation

* **Idea:** For each branch, remember the last outcome. Predict the same next time.
* **Structure:** A table of 1-bit entries. Index with low PC bits. This is typically **untagged**, so different PCs can map to the same entry. That collision is **aliasing**.
* **Aliasing and tags:** To reduce aliasing, you can add a **tag** per entry that stores some higher-order PC bits. On lookup, index with low bits, then compare the tag. If the tag mismatches, treat as a miss and fall back to a default (often not-taken). Many real BHTs are untagged to save area; BTBs are usually tagged.
* **Indexing example:** With (N) entries you index by (\text{PC} \bmod N). The “two-digit decimal” illustration in the transcript was just a toy analogy.

# 4) One-bit predictor: behavior on loops

Consider a loop branch that is taken for (K-1) iterations, then not-taken once to exit.

* If the predictor state at the start of this loop’s next entrance is **not-taken** (common, because the last use of this branch was the exit), it will:

  * Mispredict on the **first** iteration (it predicts NT, actual is T), then learn T.
  * Mispredict on the **last** iteration (it predicts T, actual is NT).
* **Steady state:** ~2 mispredictions per full loop execution.

# 5) Limits of a 1b predictor

Perfect alternation T,NT,T,NT defeats it. After warmup it mispredicts **every** time for that pattern.

# 6) Two-bit (2b) saturating counter predictor

* **Idea:** Add hysteresis. Use a 2-bit saturating counter per entry with four states:

  ```
  00: strongly NT   -> predict NT
  01: weakly   NT   -> predict NT
  10: weakly   T    -> predict T
  11: strongly T    -> predict T
  ```

  On **T**: increment toward 11. On **NT**: decrement toward 00.

* **Effect:** Needs two consecutive contrary outcomes to flip the prediction. Resists noise.

* **FSM clarity:** The transcript’s “1 z / 1 c” were typos. The states are 00, 01, 10, 11 only.

# 7) Two-bit predictor: behavior on loops

* First ever encounter starting at 00:

  * First taken causes mispredict (00→01), second taken causes mispredict (01→10), then predictions are correct until the loop exit, which is one more mispredict (predict T at 10 or 11, actual NT).
* **Steady state across loop re-entries:** The state when the loop last finished is typically **10** (weakly T). On the next entry, it predicts **taken** immediately and stays correct until the final not-taken. Net **~1 misprediction per loop** in steady state. This is the key win over 1b.

# 8) Two-bit predictor: limits

Strict alternation T,NT,T,NT also defeats 2b. It toggles between 01 and 10 and mispredicts **every** time after warmup. So 2b is not “slightly better” than 1b on alternation; both are effectively 0% accurate in steady state.

# 9) Generalizing to n-bit saturating counters

You can use (n>2) bits per counter. More bits increase stickiness and learning time. For typical workloads, (n=2) is a good trade-off. Larger (n) gives little benefit versus extra area and latency.

# 10) Table sizes and placement

Predictor tables are small SRAMs near fetch. Sizes are usually in the low kilobytes to tens of kilobytes depending on design goals. Bigger is not always better due to access time, power, and diminishing returns. Code footprints and server workloads can justify larger structures.

# 11) Direction vs target: BHT/PHT and BTB

* **Direction prediction:** Comes from a table of 1b/2b counters. Historically called **BHT** (branch history table) when indexed by PC.
* **Target prediction:** Comes from the **BTB** (branch target buffer) which stores target addresses for taken branches. BTBs are **tagged** and indexed by PC. They are separate from the direction predictor in most designs, though some academic diagrams merge them for simplicity.
* Predicting direction alone is not enough. On predicted-taken you must also supply the target early so fetch can continue without bubbles.

# 12) From bias to patterns: correlating (two-level) predictors

1b/2b per-PC predictors learn **bias** only. They fail on patterned sequences such as alternation, or on branches correlated with other branches.

Two-level schemes capture **history**:

* **Global correlation:** The outcome of branch B depends on previous outcomes of other branches.
* **Local correlation:** The outcome of a given static branch depends on its own recent history pattern, not just a single last outcome.

# 13) Global-history predictors: GHR + PHT

* **Global History Register (GHR):** An (n)-bit shift register of the most recent branch outcomes across the whole program. Shift in 1 for taken, 0 for not-taken.

* **Pattern History Table (PHT):** A table with (2^n) entries of 2-bit counters. Index the PHT with the **GHR** (not the PC). The selected counter predicts T or NT and is updated by saturating increment/decrement.

* **Cost:** If GHR has (n) bits, the PHT has (2^n) entries. Example: (n=5) yields 32-entry PHT.

* **Latency:** Lookup is one table read using the current GHR value. No long learning latency is implied in the access; only more entries exist.

* **Targeting:** Since the PHT is not PC-indexed, you still need a separate BTB, indexed and tagged by PC, to produce the predicted target when the direction is taken.

* **Aliasing:** A pure GHR-indexed predictor is **untagged**. Different dynamic contexts can map to the same PHT entry. Hybrids like gshare mix PC bits with GHR (XOR) to reduce destructive aliasing without tags. You can also add tags to reduce aliasing at extra cost.

# 14) Examples of correlation

* **Local:** A 4-iteration loop repeats the local pattern 1,1,1,0. A predictor that indexes by recent local history of that branch can learn and reach near 100% accuracy.
* **Global:** Caller checks `if (s1 != NULL)` then calls `f(s1)`. Inside `f`, code again checks `if (s1 == NULL)`. If the call happened, the inner check will be not-taken. The inner branch is correlated with the earlier branch in the caller. A global-history predictor can exploit this.

# 15) Putting it together: fetch flow

1. Fetch uses PC to probe the BTB.
2. In parallel, direction prediction is read:

   * PC-indexed BHT/PHT for per-PC schemes, or
   * GHR-indexed PHT for global schemes, or
   * Hybrids that combine them.
3. If direction predicts taken and BTB hits, next fetch PC becomes the BTB target. Otherwise fetch PC+4.

# 1) Two-level (correlating) predictor is still one predictor

* **Mechanism:** Keep an (n)-bit **global history register (GHR)** of recent branch outcomes (1=T, 0=NT). Index a **pattern history table (PHT)** of (2^n) 2-bit saturating counters with the GHR. The selected counter predicts T/NT and is updated by saturating inc/dec.
* **Not a second predictor:** “Two-level” refers to the two data levels (history + counters), not two separate predictors.
* **Targeting:** Direction comes from the PHT. Targets still come from a **BTB** indexed and tagged by PC.

# 2) Accuracy → CPI: fix the math

Assumptions from the text: 20% branches, mispredict penalty = 10 cycles, base CPI = 1, two cases:

* 2-bit per-PC predictor: **92%** accurate → **8%** mispredicts
* Two-level (global) predictor: **96%** accurate → **4%** mispredicts

Correct CPI model:
[
\text{CPI} ;=; 1 ;+; f_b \cdot \text{MPR} \cdot P
]
where (f_b=0.2), (P=10), MPR = mispredict rate.

* 92%: (1 + 0.2 \cdot 0.08 \cdot 10 = 1.16)
* 96%: (1 + 0.2 \cdot 0.04 \cdot 10 = 1.08)

A **4-point** accuracy gain here cuts CPI from 1.16 to 1.08 (~6.9% speedup). The transcript’s 1.11/1.07 numbers were inconsistent.

# 3) Front-end staging: BTB vs predictor

* Common practice: **BTB in fetch**, direction predictor also in fetch or very early decode for timing balance.
* A correct prediction has **lookup latency**, not a mispredict penalty. If direction is only available a stage later, the design must hide that fixed latency in the front-end, or accept a small bubble. The transcript’s “decode” placement is one possible balance, not a rule.

# 4) Interference and how to reduce it

Global history indexed PHTs can **interfere**: different PCs with the same GHR map to the same counter.

* **Bimodal (per-PC 2-bit) predictor:** robust on biased branches; indexed by PC; low pattern sensitivity.
* **Global two-level predictor:** exploits cross-branch patterns; vulnerable to interference.
* **Tournament predictor (chooser):** run **both** (e.g., per-PC predictor and global predictor) in parallel. A **meta-predictor** (another 2-bit counter table, PC-indexed) learns which sub-predictor to trust for each branch.
* **gshare:** XOR PC bits with GHR to form the PHT index. This mixes per-branch identity with global history to reduce destructive aliasing without tags.

# 5) Names and corrections

* “bodel/bimodel” → **bimodal** predictor (per-PC 2-bit counters).
* “G-Shack / G-sh” → **gshare**.
* “Samson perceptron” → **perceptron predictor** (linear classifier over history; Jiménez & Lin).
* “page” → **TAGE** (TAgged GEometric history lengths; multiple tagged tables with geometric history lengths).
* **Alpha 21264** is a well-known example of a **tournament**-style predictor (local + global + choice).
* **CBP** (Championship Branch Prediction) exists; designs are evaluated under fixed hardware budgets.

# 6) Multicycle operations: latency vs initiation interval

* **Latency (to a consumer):** cycles from producer issue until a dependent consumer can read via forwarding.
* **Initiation Interval (II):** minimum cycles between starts of **same-type** ops on a unit.

Examples:

* Integer ADD: 1-cycle exec, fully pipelined → latency 0 (to EX consumer), II = 1.
* FP ADD: 4 stages, fully pipelined → latency 3 (consumer needs result at its EX), II = 1.
* FP DIV: 25 cycles, **not** pipelined → latency 24, II = 25.

# 7) Write-back contention with multicycle units

Multiple long-latency ops can complete together. Handle with **multi-ported register files** or **write-back arbitration** (and possible stalls). The simple 5-stage “one WB per cycle” assumption breaks once you add multicycle units.

# 8) Scheduling the loop with given latencies

Context from the transcript: loop does

* `L.D` for `x[i]`
* `ADD.D F4, F0, F2` to accumulate into `F4` using `F2`
* `S.D F4, ...` consumes `F4`
* address update `R1 := R1 - 8`
* branch with a **delay slot**
  Given: load-use delay = 1 cycle; `ADD.D` latency to **store** = 2 cycles (one less than to an EX consumer, since stores need the value in MEM); branch resolved in decode → needs producer value one stage earlier → 1-cycle stall if the producer is immediately prior.

### Constraints to satisfy

* Fill the **load delay slot** after `L.D`.
* Insert **≥2 cycles** between `ADD.D producing F4` and the `S.D` that uses F4.
* Fill the **branch delay slot** with an instruction that is always executed.
* If you move the **index update** `R1 := R1 - 8` above the store, compensate the store’s addressing by **+8**.

### One valid schedule (MIPS-like)

```
loop:
    L.D    F0,   0(R1)          # load x[i]
    ADDI   R1,  R1,  -8         # index update fills load-use slot
    ADD.D  F4,  F0,  F2         # s + x[i] -> F4
    <independent integer op>    # fills 1st cycle before store
    NOP                         # 2nd cycle if nothing usable
    BNE    R1,  R0,  loop       # branch resolved in decode
    S.D    F4,   8(R1)          # delay slot; +8 undoes earlier ADDI
```

Why it works:

* The ADDI hides the load-use stall.
* Two cycles separate `ADD.D` and `S.D` (one useful op + one bubble).
* The store in the **delay slot** is always executed. Its address is corrected with `+8` because `R1` was decremented earlier.
* The branch’s decode-stage compare is not waiting on a value produced in the immediately prior EX stage here, so the extra 1-cycle branch stall mentioned in the transcript is avoided in this schedule.

If you have any truly independent work (pointer arithmetic for the next iteration, loop-invariant hoists, integer adds), use those to eliminate the `NOP`.

# 9) Global-predictor interference example clarified

The transcript’s example mixed two branches under the same GHR pattern (e.g., one alternating T/NT and one 4-iteration loop). That is exactly the interference problem a global-only index can suffer. **gshare** or a **tournament** mitigates this by injecting PC identity or by choosing per branch between predictors that exploit different structure.

# 10) Final clarifications

* A 2-bit predictor **does not** improve strict T/NT alternation; after warm-up it mispredicts every time, same as 1-bit.
* BTB entries are **tagged**; many simple BHT/PHTs are not. Tagged PHTs exist (e.g., TAGE) to reduce aliasing.
* “Two bits of history gives four entries” means the **PHT has 4 indices**; each entry still holds a **2-bit counter** in typical designs.
