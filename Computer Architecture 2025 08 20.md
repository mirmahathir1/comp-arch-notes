# Dynamic vs. static power and energy

* **Dynamic power**


  `P_dyn ∝ C * V^2 * f`


  where `C` is effective switched capacitance, `V` is supply voltage, `f` is clock frequency, and `alpha` in `[0,1]` is the activity factor.

* **Dynamic energy for a fixed amount of work**

  Equivalently, with `N_cycles` cycles and CPI constant:
  `E_dyn ∝ P_dyn * T`

  `= C * V^2 * f) * (N_cycles / f)` 
  
  `= C * V^2 * N_cycles`

  Frequency cancels for a CPU-bound workload at fixed `V`.

* **Static (leakage) power**
  Static energy over runtime `T`: `E_static = P_static * T`.

# Why frequency affects power but not dynamic energy (common assumptions)

* If CPI and voltage are unchanged and the workload is CPU-bound: `T = N_cycles / f`. Increasing `f` raises `P_dyn` linearly but shortens `T` proportionally, so `E_dyn` is unchanged.
* This cancellation does not include static energy. Slower frequency lengthens `T` and therefore increases `E_static`.

# DVFS, DFS, and overclocking

* **DVFS**: change both `V` and `f` together. Big lever on power (`∝ V^2 * f`) and often on energy via `V^2`.
* **DFS**: change only `f` at fixed `V`. Cuts power, leaves dynamic energy roughly unchanged for CPU-bound work, and can increase total energy due to higher leakage energy.
* **Overclocking**: increase `f`, often with a voltage bump to keep timing. More performance, more power, and usually more total energy.

# Worked mini-example: 15% voltage drop with a matched 15% frequency drop

**Question:** Some microprocessors today are designed to have adjustable voltage, so a 15% reduction in voltage may result in a 15% reduction in frequency. What would be the impact on dynamic energy and on dynamic power?

Let `V' = 0.85 * V`, `f' = 0.85 * f`.

* **Dynamic energy scaling**: `E_dyn' / E_dyn = (V'/V)^2 = 0.85 * 0.85 = 0.72`
  Dynamic energy falls to **72%** of the original.

* **Dynamic power scaling**: `P_dyn' / P_dyn = (V'/V)^2 * (f'/f) = 0.85 * 0.85 * 0.85 = 0.61`
  Dynamic power falls to **61%** of the original.

# When DFS alone can still save energy

* If the workload is not CPU-bound (I/O-bound or memory-bound), reducing `f` might barely increase `T`. Then `P_dyn` drops while `T` changes little, so total energy can drop.
* If static power is large, a strategy of race to idle often wins: run fast at a given `V` to finish sooner, then enter deep idle states to curb leakage energy.

# Clock gating vs. power gating

* **Clock gating**: disable clocks to idle units. Cuts dynamic power in those sequential elements by preventing unnecessary switching. Does not eliminate leakage.
* **Power gating**: cut supply to idle blocks. Reduces static power dramatically but has wake-up latency and state-retention costs.

# Dennard scaling and leakage (terminology and causality)

* It is **Dennard** scaling. Historically, voltage and dimensions scaled to hold power density roughly constant.
* Scaling hit limits because threshold voltage `V_th` could not keep dropping without exponential growth in subthreshold and gate leakage. As a result, supply voltage stopped scaling aggressively, and leakage became a significant share of total power.

# Numerical exercise

**Given:** Processor A at 3 GHz. `P_dyn = 80 W`. `P_static = 20 W`. Program time `T0 = 20 s`.

## 1) Frequency scaled down by 20% (DFS). Voltage unchanged.

* `f' = 0.8 * f`. Assume CPU-bound and CPI unchanged.
* New dynamic power: `P_dyn' = 0.8 * 80 = 64 W`.
* New runtime: `T' = T0 / 0.8 = 25 s`.
* Dynamic energy: `E_dyn' = 64 * 25 = 1600 J`. Original `E_dyn = 80 * 20 = 1600 J`. **No change.**
* Static energy: `E_static' = 20 * 25 = 500 J`. Original `= 400 J`. **Increases.**
* **Total energy:** `E_tot' = 1600 + 500 = 2100 J` vs. `E_tot = 2000 J`. **Worse** with DFS alone.

## 2) Frequency and voltage both scaled down by 20% (DVFS).

* `f' = 0.8 * f`, `V' = 0.8 * V`.
* Dynamic power scale: `0.8 * 0.8^2 = 0.512`. So `P_dyn' = 0.512 * 80 = 40.96 W`.
* Runtime: `T' = 25 s`.
* Dynamic energy: `E_dyn' = 40.96 * 25 = 1024 J`. **Reduced** from 1600 J.
* Static power model: if `P_static = I_leak * V` with `I_leak` fixed, then `P_static' = 0.8 * 20 = 16 W` and `E_static' = 16 * 25 = 400 J`, which matches the original 400 J because `0.8 * 1.25 = 1`. In real silicon, `I_leak` often also drops with lower `V` and temperature, so static energy can improve further.
* **Total energy (idealized):** `E_tot' = 1024 + 400 = 1424 J`. **Better** than 2000 J.

# Practical constraints and stability

* Raising `f` at fixed `V` eventually violates timing and causes instability. Overclocking typically needs a voltage increase and better cooling.
* Lowering `V` too far at a given `f` causes timing errors. DVFS controllers pick safe `(V, f)` pairs.
* CPI and memory behavior can shift with `(V, f)` because of different stall or compute overlaps, so the simple linear models are first-order guides, not guarantees.

# 1) Device scaling, supply voltage, and threshold voltage

* **Why lowering `VDD` affects max frequency:** CMOS gate delay rises as `VDD` approaches `V_th` (threshold). A first-order model:
  `f_max ∝ ((VDD - V_th)^gamma) / VDD`, with `gamma` in `[1,2]`. Lower `VDD` implies lower `f_max` unless you also lower `V_th`.
* **Why not keep `f` constant while lowering `VDD`:** You can only undervolt within the existing timing margin. Past that margin, paths violate setup time and errors appear. Practical DVFS uses validated `(V, f)` pairs.
* **Why Dennard scaling hit limits:** As dimensions shrank, `V_th` could not scale proportionally without exploding leakage. Result: `VDD` stopped dropping quickly. Leakage and variability rose. The dominant constraints are leakage, variability, and reliability.

# 2) Static (leakage) power: what actually scales

* **Definition:** `P_static = I_leak * VDD`.
* **Misconception to fix:** Static power is not `∝ VDD^2`. The `V^2` term belongs to dynamic power.
* **How leakage changes:** `I_leak` depends strongly on `V_th`, `VDD`, process, and temperature. Lowering `VDD` often reduces `I_leak` somewhat, but not by a simple square law. First-order teaching problems usually hold `I_leak` constant to isolate effects.
* **Clock vs power gating:**
  Clock gating stops toggling and cuts dynamic power only.
  Power gating removes supply to a block and cuts static power at the cost of wake-up latency and state retention.

# Processor Power Scaling and Energy Consumption

## Problem Statement

Processor-A runs at **3 GHz**, consuming:

* **Dynamic power** = 80 W
* **Static power** = 20 W
* **Execution time** = 20 s

We need to calculate total **energy consumption** under two scenarios:

1. **Only frequency scaled down by 20%**
2. **Both frequency and voltage scaled down by 20%**

---

## Case 1: Frequency scaled down by 20%

* **New frequency** = 3 × 0.8 = 2.4 GHz
* **Dynamic power** = 80 × 0.8 = 64 W
* **Static power** = 20 W (unchanged)
* **Execution time** = 20 ÷ 0.8 = 25 s

**New total power** = 64 + 20 = **84 W**
**New energy** = 84 × 25 = **2100 J**

**Explanation:**
Dynamic power is proportional to frequency. Static power does not change with frequency. Lower frequency increases execution time (since time ∝ 1/f). Total energy is power × time.

---

## Case 2: Frequency and voltage scaled down by 20%

* **New frequency** = 2.4 GHz
* **Voltage scaling**: V → 0.8V. Dynamic power ∝ V² × f.
  → Dynamic power = 80 × 0.8 × 0.64 = **41 W**
* **Static power** = 20 × 0.8 = **16 W** (static ∝ V)
* **Execution time** = 25 s

**New total power** = 41 + 16 = **57 W**
**New energy** = 57 × 25 = **1425 J**

**Explanation:**
Scaling both voltage and frequency reduces dynamic power significantly (since power ∝ V²f). Static power also reduces because it depends linearly on voltage. Although runtime increases, the large drop in power yields lower overall energy.

---

## Final Results

* **Frequency only scaled (20%)** → **2100 J**
* **Frequency + Voltage scaled (20%)** → **1425 J**

Voltage scaling reduces energy more effectively than frequency scaling alone.


# 4) Compiler constant propagation and precompute

* **Constant propagation** replaces expressions whose operands are compile-time constants with their values.
* **Constant folding** computes those values at compile time.
* **Strength reduction** replaces costly ops with cheaper ones, for example multiply by power-of-two to shift.
* These lower dynamic energy by cutting activity `alpha` and sometimes instruction count.

# 5) Latency, throughput, bandwidth

* **Definitions:** Latency is time per task. Throughput or bandwidth is tasks per unit time.
* **Not generally reciprocal:** `Throughput != 1 / Latency` except for a single-server no-overlap system.
* **Little’s Law:** `Concurrency = Throughput * Latency`, so `Throughput = Concurrency / Latency`.
* **Pipelining example:** `k`-stage pipeline with cycle time `tau`. Latency `= k * tau`. Throughput `= 1 / tau` once full, not `1 / (k * tau)`.
* **Memory example:** Read latency might be 80 ns, but with many banks or queues the sustained bandwidth can be high because many requests overlap.
* **Terminology:** Bandwidth and throughput are interchangeable. Latency aligns with execution time.

# 6) Benchmarks and misleading proxies

* Run your workload when possible. Proxies can mislead.
* Clock frequency alone is a poor predictor. IPC or CPI, memory behavior, microarchitecture, and ISA all matter.
* MIPS depends on ISA semantics; not comparable across ISAs and even within one ISA it rewards trivial instructions.
* FLOPS or peak GFLOPS are theoretical maxima; real applications are limited by memory and communication.

# 7) Speedup language and formulas

* “A is `n×` faster than B” means `Speedup S = T_B / T_A = n`.
* “A is `m%` faster than B” means `m = ((T_B - T_A) / T_B) * 100%`.
  Equivalently, `T_A = (1 - m/100) * T_B` and `S = 1 / (1 - m/100)`.

# 8) Parallelism at multiple levels

* **System or core level:** Multicore, manycore, accelerators.
* **Thread or ILP level:** Out-of-order issue, superscalar, speculation, vector or SIMD.
* **Pipeline level:** Overlap stages to raise throughput.
* **Circuit level:** Carry-lookahead or Kogge-Stone adders increase bit-level parallelism versus ripple-carry.

# 9) Locality and caches

* **Temporal locality:** Recently used items likely reused soon.
* **Spatial locality:** Nearby items likely accessed next.
* Caches exploit both to avoid slow main memory for the common case.

# 10) Focus on the common case: Amdahl’s Law

* **Model:** Fraction `f` of time benefits from enhancement with speedup `s`. New time:
  `T_new = T_old * ((1 - f) + f/s)`
  **Speedup:** `S = 1 / ((1 - f) + f/s)`
* **Example:** `f = 0.10`, `s = 2`.
  `S = 1 / (0.9 + 0.1/2) = 1 / 0.95 = 1.0526` about **5.26%**.
* **Consequence:** Optimize what dominates time. For scaled-up problems, Gustafson’s perspective applies, but Amdahl governs fixed-size speedup.

# 11) Overclocking and stability facts

* Frequency-only overclocking works until timing margin is exhausted. Past that, errors or crashes occur.
* Practical overclocking usually increases `VDD` to regain margin, which raises dynamic power (`∝ V^2 * f`) and often leakage; better cooling becomes necessary.



# 12) Quick check: power vs. energy with time change

**Question:** Processor A consumes on average 20% more power than processor B. Processor A on average needs 70% of the time that processor B needs. What is the energy of A relative to B?

**Answer:** Using `E = P * T`, `E_A / E_B = 1.2 * 0.7 = 0.84`. Processor A uses **84%** of B’s energy (16% less).

# Question

Branch instructions take 2 cycles; all other instructions take 1 cycle (compare takes 1).
CPU A uses a separate compare + branch. CPU B has a single branch.
CPU A’s clock is 1.25× CPU B. On CPU A, 20% of instructions are branches (and another 20% are compares). Which CPU is faster?

# Answer

**CPU A is faster by 25%.**

## CPU A

* 20% branches → 2 cycles
* 20% compares → 1 cycle
* 60% other → 1 cycle
* Average CPI = (0.2 × 2) + (0.2 × 1) + (0.6 × 1) = **1.2**

## CPU B

* 20% branches → 2 cycles
* 80% other → 1 cycle
* Average CPI = (0.2 × 2) + (0.8 × 1) = **1.2**

## Performance Comparison

Execution time ∝ CPI ÷ frequency

* CPU A time = 1.2 ÷ (1.25 × fB)
* CPU B time = 1.2 ÷ fB

Ratio = (1.2 / 1.25 fB) ÷ (1.2 / fB) = 1 ÷ 1.25 = **0.8**

So CPU A uses 80% of CPU B’s time → **1.25× faster**.
