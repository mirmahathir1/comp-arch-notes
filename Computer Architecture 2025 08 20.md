Here is a corrected and expanded version, organized by topic.

# Dynamic vs. static power and energy

* **Dynamic power**
  (P_{\text{dyn}} = \alpha,C,V^2,f)
  where (C) is effective switched capacitance, (V) is supply voltage, (f) is clock frequency, and (\alpha) (0–1) is the activity factor.

* **Dynamic energy for a fixed amount of work**
  (E_{\text{dyn}} = \alpha,C,V^2,N_{\text{switches}}).
  Equivalently, with (N_{\text{cycles}}) cycles and CPI constant,
  (E_{\text{dyn}} = P_{\text{dyn}} \cdot T = (\alpha C V^2 f)\cdot(N_{\text{cycles}}/f)=\alpha C V^2 N_{\text{cycles}}).
  Frequency cancels for a CPU-bound workload at fixed (V).

* **Static (leakage) power**
  (P_{\text{static}} = I_{\text{leak}} \cdot V).
  Static energy over runtime (T): (E_{\text{static}}=P_{\text{static}}\cdot T).

# Why frequency affects power but not dynamic energy (under common assumptions)

* If CPI and voltage are unchanged and the workload is CPU-bound:
  (T = N_{\text{cycles}}/f). Increasing (f) raises (P_{\text{dyn}}) linearly but shortens (T) proportionally, so (E_{\text{dyn}}) is unchanged.
* This cancellation does **not** include static energy. Slower frequency lengthens (T) and therefore increases (E_{\text{static}}).

# Voltage–frequency relationship (why DVFS is coupled)

* Maximum safe frequency rises with voltage because gate delay falls as (V) increases. A common model is the alpha-power law:
  (f_{\max} \propto \dfrac{(V - V_{th})^{\gamma}}{V}) with (\gamma \in [1,2]).
  Hence, lowering (V) typically requires lowering (f) to meet timing; raising (V) allows higher (f).

# DVFS, DFS, and overclocking

* **DVFS**: change **both** (V) and (f) together. Big lever on **power** ((\propto V^2 f)) and often on **energy** via (V^2).
* **DFS**: change **only** (f) at fixed (V). Cuts **power**, leaves **dynamic energy** roughly unchanged for CPU-bound work, and can increase **total energy** due to higher leakage energy.
* **Overclocking**: increase (f), often with a voltage bump to keep timing. More performance, more power, and usually more total energy.

# Worked mini-example: 15% voltage drop with a matched 15% frequency drop

Let (V' = 0.85V), (f' = 0.85f).

* **Dynamic energy scaling**: (E'*{\text{dyn}}/E*{\text{dyn}} = (V'/V)^2 = 0.85^2 = 0.7225).
  Dynamic energy falls to **72.25%** of the original.

* **Dynamic power scaling**: (P'*{\text{dyn}}/P*{\text{dyn}} = (V'/V)^2 (f'/f) = 0.85^2 \cdot 0.85 = 0.614125).
  Dynamic power falls to **61.41%** of the original.

# When DFS alone can still save energy

* If the workload is **not CPU-bound** (I/O-bound or memory-bound), reducing (f) might barely increase (T). Then (P_{\text{dyn}}) drops while (T) changes little, so total energy can drop.
* If **static power is large**, a strategy of “**race to idle**” often wins: run fast at a given (V) to finish sooner, then enter deep idle states to curb leakage energy.

# Clock gating vs. power gating

* **Clock gating**: disable clocks to idle units (e.g., FPU during integer-only code). This **cuts dynamic power** in those sequential elements by preventing unnecessary switching. It does **not** eliminate leakage.
* **Power gating**: cut supply to idle blocks. This **reduces static power** dramatically but has wake-up latency and state-retention costs.

# Dennard scaling and leakage (terminology and causality)

* It is **Dennard** scaling. Historically, voltage and dimensions scaled to hold power density roughly constant.
* Scaling hit limits because threshold voltage (V_{th}) could not keep dropping without **exponential** growth in subthreshold and gate leakage. As a result, supply voltage stopped scaling aggressively, and leakage became a significant share of total power.

# Numerical exercise

**Given:** Processor A at 3 GHz. (P_{\text{dyn}}=80) W. (P_{\text{static}}=20) W. Program time (T_0=20) s.

## (1) Frequency scaled down by 20% (DFS). Voltage unchanged.

* (f' = 0.8f). Assume CPU-bound and CPI unchanged.
* New dynamic power: (P'_{\text{dyn}} = 0.8 \cdot 80 = 64) W.
* New runtime: (T' = T_0/0.8 = 25) s.
* Dynamic energy: (E'*{\text{dyn}} = 64 \cdot 25 = 1600) J. Original (E*{\text{dyn}} = 80 \cdot 20 = 1600) J. **No change.**
* Static energy: (E'_{\text{static}} = 20 \cdot 25 = 500) J. Original (= 400) J. **Increases.**
* **Total energy:** (E'*{\text{tot}} = 1600 + 500 = 2100) J vs. (E*{\text{tot}} = 2000) J. **Worse** with DFS alone.

## (2) Frequency and voltage both scaled down by 20% (DVFS).

* (f' = 0.8f), (V' = 0.8V).

* Dynamic power scale: (0.8 \cdot 0.8^2 = 0.512). So (P'_{\text{dyn}} = 0.512 \cdot 80 = 40.96) W.

* Runtime: (T' = 25) s.

* Dynamic energy: (E'_{\text{dyn}} = 40.96 \cdot 25 = 1024) J. **Reduced** from 1600 J.

* Static power model: if we take (P_{\text{static}} = I_{\text{leak}} V) with **(I_{\text{leak}}) fixed**, then (P'*{\text{static}} = 0.8 \cdot 20 = 16) W and
  (E'*{\text{static}} = 16 \cdot 25 = 400) J, which **matches** the original 400 J because (0.8 \times 1.25 = 1).
  In real silicon, (I_{\text{leak}}) often also drops with lower (V) and temperature, so static energy can improve further.

* **Total energy (idealized):** (E'_{\text{tot}} = 1024 + 400 = 1424) J. **Better** than 2000 J.

# Practical constraints and stability

* Raising (f) at fixed (V) eventually violates timing and causes instability. Overclocking typically needs a voltage increase and better cooling.
* Lowering (V) too far at a given (f) causes timing errors. DVFS controllers pick safe ((V,f)) pairs.
* CPI and memory behavior can shift with ((V,f)) because of different stall/compute overlaps, so the simple linear models are first-order guides, not guarantees.

Here is a corrected and expanded version, organized by topic.

# 1) Device scaling, supply voltage, and threshold voltage

* **Why lowering (V_{DD}) affects max frequency:** CMOS gate delay rises as (V_{DD}) approaches (V_{th}) (threshold). A common first-order model:
  [
  f_{\max} \propto \frac{(V_{DD}-V_{th})^{\gamma}}{V_{DD}},\quad \gamma\in[1,2].
  ]
  Lower (V_{DD}) ⇒ lower (f_{\max}) unless you also lower (V_{th}).

* **Why not just keep (f) constant while lowering (V_{DD}):** You can only undervolt within the existing **timing margin**. Past that margin, paths violate setup time and errors appear. Practical DVFS uses validated ((V,f)) pairs.

* **Why Dennard scaling hit limits:** As dimensions shrank, (V_{th}) could not scale proportionally without exploding leakage (subthreshold and gate). Result: (V_{DD}) stopped dropping quickly. Leakage and variability rose. “Quantum effects” exist (e.g., tunneling through thin oxides) but the dominant constraints are leakage, variability, and reliability, not hand-wavy “quantum weirdness.”

# 2) Static (leakage) power: what actually scales

* **Definition:** (P_{\text{static}}=I_{\text{leak}}\cdot V_{DD}).

* **Misconception to fix:** Static power is **not** (\propto V_{DD}^2). The (V^2) term belongs to **dynamic** power.

* **How leakage changes:** (I_{\text{leak}}) depends strongly on (V_{th}), (V_{DD}), process, and temperature. Lowering (V_{DD}) often reduces (I_{\text{leak}}) somewhat, but not by a simple square law. First-order teaching problems usually hold (I_{\text{leak}}) constant to isolate effects.

* **Clock vs power gating:**

  * Clock gating stops toggling ⇒ **cuts dynamic** power only.
  * Power gating removes supply to a block ⇒ **cuts static** power (plus dynamic) at the cost of wake-up latency and state retention.

# 3) Revisiting the two energy problems (unit-correct and numerically consistent)

**Given:** (P_{\text{dyn,0}}=80\text{ W}), (P_{\text{stat,0}}=20\text{ W}), (f_0=3\text{ GHz}), CPU-bound, (T_0=20\text{ s}).
Original totals: (P_0=100\text{ W}), (E_{\text{dyn,0}}=80\cdot20=1600\text{ J}), (E_{\text{stat,0}}=20\cdot20=400\text{ J}), (E_0=2000\text{ J}).

## (a) DFS only: frequency ↓20% (to (0.8f_0)), voltage unchanged

* (P'_{\text{dyn}} = 0.8\cdot 80 = 64\text{ W}).
* (P'*{\text{stat}} = 20\text{ W}) (unchanged at fixed (V*{DD})).
* Runtime (T' = T_0/0.8 = 25\text{ s}) (CPU-bound).
* Energies: (E'*{\text{dyn}}=64\cdot25=1600\text{ J}) (**unchanged**), (E'*{\text{stat}}=20\cdot25=500\text{ J}) (**worse**).
* **Total:** (E' = 2100\text{ J}). Units are **joules**, not “watt-seconds” in prose and certainly not “watts.”

## (b) DVFS: frequency ↓20% and voltage ↓20% ((f'=0.8f_0,\ V'=0.8V_0))

* Dynamic power scale: (V'^2f'/V_0^2f_0=0.8^2\cdot0.8=0.512).
  (P'_{\text{dyn}}=0.512\cdot80=40.96\text{ W}).
* Runtime (T'=25\text{ s}).
* (E'_{\text{dyn}}=40.96\cdot25=1024\text{ J}) (**down** from 1600 J).
* If (I_{\text{leak}}) is held constant (teaching assumption): (P'*{\text{stat}}=0.8\cdot20=16\text{ W}).
  (E'*{\text{stat}}=16\cdot25=400\text{ J}) (same as baseline because (0.8\times1.25=1)).
* **Total:** (E'=1024+400=1424\text{ J}). In real chips, leakage often also drops with (V) and temperature, so static energy can improve further.

# 4) Compiler constant propagation and “precompute”

* **Constant propagation** replaces expressions whose operands are compile-time constants with their values.
* **Constant folding** computes those values at compile time.
* **Strength reduction** replaces costly ops with cheaper ones (e.g., multiply by power-of-two → shift).
* These remove dynamic switching for the optimized region, indirectly lowering dynamic energy by cutting activity ((\alpha)) and sometimes instruction count.

# 5) Latency, throughput, bandwidth: independence and the correct relations

* **Definitions:**
  Latency = time per task. Throughput/bandwidth = tasks per unit time.

* **Not generally reciprocal:** ( \text{Throughput} \neq 1/\text{Latency} ) except for a single-server, no-overlap system.

* **General relation (Little’s Law):**
  [
  \text{Concurrency (in-flight)} = \text{Throughput} \times \text{Latency}.
  ]
  Thus, (\text{Throughput} = \text{Concurrency}/\text{Latency}).

* **Pipelining example:** k-stage pipeline with cycle time (\tau):
  Latency (=k\tau). Throughput (=1/\tau) once full. Not (1/(k\tau)).

* **Memory example:** Read latency might be 80 ns, but with many banks/ports/queues the sustained bandwidth can be gigabytes per second because many requests overlap.

* **Terminology:** “Bandwidth” and “throughput” are used interchangeably; “latency” aligns with “execution time.”

# 6) Benchmarks and misleading proxies

* **Run your workload** when possible. Proxies can mislead.
* **Clock frequency** alone is a poor predictor. IPC/CPI, memory behavior, microarchitecture, and ISA all matter.
* **MIPS** depends on ISA semantics; not comparable across ISAs and even within one ISA it rewards trivial instructions.
* **FLOPS/peak GFLOPS** are theoretical maxima under ideal kernels; real applications are limited by memory and communication.

# 7) Speedup language and formulas

* **“A is (n\times) faster than B”** means ( \text{Speedup }S = T_B/T_A = n).

* **“A is (m%) faster than B”** means
  [
  m = \frac{T_B - T_A}{T_B}\times 100%.
  ]
  Equivalently, (T_A=(1-m/100),T_B) and (S=1/(1-m/100)).

# 8) Parallelism at multiple levels

* **System-/core-level:** Multicore, manycore, accelerators.
* **Thread/ILP-level:** Out-of-order issue, superscalar, speculation, vector/SIMD.
* **Pipeline-level:** Overlap stages to raise throughput.
* **Circuit-level:** Ripple-carry vs carry-lookahead/carry-select/Kogge-Stone adders increase bit-level parallelism.

# 9) Locality and caches (why they work)

* **Temporal locality:** Recently used items likely reused soon.
* **Spatial locality:** Nearby items likely accessed next.
* Caches exploit both to trade a small fast store for most accesses, avoiding slow main memory for the common case.

# 10) “Focus on the common case” formalized: Amdahl’s Law

* **Model:** Fraction (f) of time benefits from enhancement with speedup (s). New time:
  [
  T_{\text{new}} = T_{\text{old}}\left[(1-f)+\frac{f}{s}\right].
  ]
  **Speedup:**
  [
  S = \frac{T_{\text{old}}}{T_{\text{new}}} = \frac{1}{(1-f)+f/s}.
  ]

* **Example corrected:** (f=0.10,\ s=2).
  (S=1/(0.9+0.1/2)=1/0.95=1.0526). About **5.26%** overall. Big local gains yield modest global gains if (f) is small.

* **Consequence:** Optimize what dominates time. Otherwise, diminishing returns.
  (For scaled-up problems, Gustafson’s perspective applies, but Amdahl governs fixed-size speedup.)

# 11) Overclocking and stability facts

* **Frequency-only overclocking** works until timing margin is exhausted. Past that, errors or crashes occur.
* **Practical overclocking** usually increases (V_{DD}) to regain margin, which raises dynamic power ((\propto V^2 f)) and often leakage; better cooling becomes necessary.

# 12) Notational and naming corrections

* **“Mo’s law” → Moore’s Law.**
* **“Andal’s law” → Amdahl’s Law.**
* **“floatingoint” → floating-point.**
* **“microp processors” → microprocessors.**
* **BIOS** spelling as used, though many systems expose firmware settings via UEFI.

# 13) Quick consistency checks for future problems

* Always track **units**: power in watts, time in seconds, energy in joules.
* For DFS at fixed (V): expect (E_{\text{dyn}}) unchanged, (E_{\text{static}}) up, total energy often worse for CPU-bound code.
* For DVFS: dynamic energy scales with (V^2); total energy usually improves, subject to leakage behavior and performance targets.
* Use **Little’s Law** to reason about latency vs throughput.
