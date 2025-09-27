# Administrative notes

* Due date: Tuesday after Labor Day. Automatic 1-day extension makes the late deadline Wednesday.
* Platforms: Gradescope and Canvas.
* Help: TAs in office hours. Use Piazza for Q&A and discussion. Do not post full answers and ask “is this correct?”
* Talk: AI accelerators talk in the School of Computing. Topic is accelerator design in the cloud context. TPUs are a public example of vendor hardware.

# ISA vs microarchitecture

* ISA defines what instructions mean. Microarchitecture defines how the hardware executes them.
* This course focuses on microarchitecture, but ISA semantics must be precise because the microarchitecture must implement them exactly.

# RISC vs CISC

**Terminology and scope**

* CISC = Complex Instruction Set Computer. RISC = Reduced Instruction Set Computer.
* “Complex” refers to instruction semantics and addressing modes, not simply the count of opcodes.
* Modern ISAs all have many instructions. The key difference is whether single instructions perform multi-step work and expose rich addressing modes.

**Example**

* CISC-style x86 example, register plus memory operand in one instruction:

```asm
add r1, [r0 + offset]    ; r1 <- r1 + MEM[r0 + offset]
```

This decodes to: compute effective address, read memory, add to r1, write r1.

* RISC-style sequence doing the same work:

```asm
r2 <- MEM[r0 + offset]   ; load
r1 <- r1 + r2            ; add
```

**Why CISC existed and still exists**

* Early chips had few transistors and few exposed registers. Memory vs ALU latency gap was smaller, so memory-operand arithmetic was attractive.
* Backward compatibility matters. You cannot break decades of binaries.

**Micro-ops and translation**

* Modern x86 decoders translate CISC instructions into internal micro-operations (μops) that look RISC-like. This eases pipelining and scheduling.
* There is overhead. x86 front ends are complex and power-hungry. Vendors mitigate with decoded-μop caching:

  * Pentium 4 used a trace cache holding decoded μop traces.
  * Later Intel cores added a μop cache that supplies decoded μops directly when hits occur.

**Apple ARM transition nuance**

* Apple’s 2020 shift to ARM64 did not “avoid translation” in the hardware sense. ARM cores also decode to μops.
* Performance gains came from high-IPC cores, large caches, aggressive prefetching, fast DVFS, and unified memory. Decode simplicity helps power, but it was not the only lever.

**Status of the “debate”**

* Not resolved in the absolute. Domain-specific accelerators add complex domain opcodes. Example: matrix-multiply instructions.
* Even so, accelerators still have load and store or equivalent data-movement ops. Complex domain ops do not remove the need to fetch data.

# Instruction classes

## 1) Data processing

* Integer arithmetic: add, sub, mul, div.
* Bitwise Boolean: and, or, xor, not.
* Shifts and rotates.

**Right shifts precisely**

* Logical right shift (SRL): fill vacated MSB with 0.
* Arithmetic right shift (SRA): replicate the original sign bit into the MSB to preserve two’s-complement sign.

Example with fixed 4-bit registers:

* Value 1011₂ = −5 in two’s complement.

  * SRL by 1 → 0101₂. MSB filled with 0. LSB 1 is discarded.
  * SRA by 1 → 1101₂. MSB copied from original MSB 1. LSB 1 is discarded.
* In all fixed-width shifts you lose the bit shifted out. Register width stays constant.

Notes:

* Left shift is the same for logical and arithmetic in two’s complement on most ISAs. Both insert 0 in the LSB.
* Rotates move bits around without insertion. No sign semantics.

## 2) Data movement

* Register-to-register moves.
* Loads and stores are the only instructions that access memory in RISC ISAs.

**Load semantics**

* Load reads from memory and writes a register.

```
r1 <- MEM[r2 + imm]    ; effective address = r2 + imm
```

* Variants by width: byte, halfword, word, doubleword.
* Sign vs zero extension applies when the loaded width is smaller than the destination register width:

  * Signed load extends the top bit of the loaded value.
  * Unsigned load fills upper bits with 0.

**Store semantics**

* Store reads a register and writes memory.

```
MEM[r2 + imm] <- r1
```

* Stores do not sign-extend. They copy the specified low bytes from the register to memory.

**“Word” terminology correction**

* Word size is ISA-specific. Do not assume word == register width.

  * MIPS32: word = 32 bits. MIPS64: “word” often still 32 bits and 64 bits are “doublewords.”
  * AArch64: W registers are 32-bit, X registers are 64-bit.
  * RISC-V RV64: LW sign-extends 32→64, LWU zero-extends.
* Always check the ISA manual.

## 3) Control flow

* Conditional branches, unconditional jumps, calls/returns, traps.

**Minimal comparisons on RISC**

* Classic MIPS uses `slt`/`sltu` to set a register to 1 if a relation holds, then uses `beq`/`bne` with zero to branch.

Patterns:

```asm
# if r1 >= r2 goto L
slt t0, r1, r2          # t0 = 1 if r1 < r2
beq t0, $zero, L        # branch if NOT (r1 < r2)

# if r1 > r2 goto L
slt t0, r2, r1          # t0 = 1 if r2 < r1
bne t0, $zero, L        # branch if r1 > r2

# if r1 == r2 goto L
beq r1, r2, L

# if r1 != r2 goto L
bne r1, r2, L
```

* There is no need for a dedicated “set equal” instruction.

# Addressing modes

**Base+displacement (displacement addressing)**

* EffectiveAddress = BaseRegister + ImmediateOffset.
* Dominant in RISC. Simple for hardware. Flexible for compilers.

**Arrays**

* If `A` is the base address of an int array with 4-byte elements and you want `A[i]`:

```
addr = A + i*4
load r, [addr]
```

* Compilers either compute `i*element_size` with shifts or use scaled addressing when the ISA provides it.

**Scaled indexed addressing (CISC-style richness)**

* x86 effective address:

```
EA = base + index*scale + displacement
; scale ∈ {1,2,4,8}
```

* Useful for `base + i*stride + const`.
* Hardware computes the EA in one instruction, but compilers can synthesize the same with RISC sequences if needed.

**What x86 does NOT do**

* No “double indirection in one load” that reads a pointer from memory and then automatically dereferences it again. That requires two instructions:

  1. load ptr
  2. load [ptr]

**PC-relative branches**

* The branch target is computed as `Target = PC_relative_base + offset`.
* The base is ISA-defined. Many RISCs use the address of the instruction after the branch as the base.
* Offsets are in bytes or in instruction words depending on ISA. Example: classic MIPS encodes word offsets and the hardware shifts left by 2.

**Stacks and local variables**

* Local variables are addressed as an offset from a frame pointer or stack pointer.
* Base register = FP or SP. Offset = compiler-chosen displacement for the variable.

# Pipeline implications

* Very complex instructions bloat certain stages and make balanced pipelines hard.
* Translating CISC to μops gives the core a uniform internal ISA that pipelines and reorders well.
* Decode overhead exists. Vendors hide it with μop caches and wide front ends. The benefit depends on hit rates and code locality.
* Software dynamic binary translation (DBT) amortizes translation by caching translated blocks in a code cache. Hot loops benefit. Irregular control flow benefits less.

# Historical corrections

* Moore’s law, not “Mo’s law.”
* Systolic arrays were popularized in the late 1970s, not the 1960s.
* TPUs are Tensor Processing Units. They incorporate systolic arrays and also provide memory operations and data movement instructions.

# Worked examples

**1) Shift example, 8-bit register**

* x = 1110 0110₂ = −26.

```
SRL x,1 → 0111 0011  ; zero into MSB, drop LSB 0 → result 115
SRA x,1 → 1111 0011  ; sign 1 into MSB, drop LSB 0 → result −13
```

**2) Load variants on a 64-bit register file**

* Load byte signed:

```
rb <- sign_extend_8_to_64( MEM[addr] )
```

* Load byte unsigned:

```
rb <- zero_extend_8_to_64( MEM[addr] )
```

* Load word to 64-bit register depends on ISA:

  * RV64 `LW` sign-extends. `LWU` zero-extends.
  * AArch64 `LDR Wt,[...]` zero-extends into Xt.

**3) Array element with base+index**

* RISC sequence without scaled addressing:

```asm
t0 <- i
t0 <- t0 << 2          ; multiply by 4
t0 <- t0 + A           ; effective address
r  <- MEM[t0]          ; load A[i]
```

* x86 with scaled index:

```asm
mov r, [A + i*4]
```

# Common pitfalls fixed from the transcript

* “Gradescope and cannons” → Gradescope and Canvas.
* “Padza/PZA/piaza” → Piazza.
* “CIS vs risk” → CISC vs RISC.
* “microode” → microcode.
* Equality is handled with `beq`/`bne`; there is no dedicated “set equal” in classic MIPS.
* Word size is ISA-specific. Do not assert “word = 64 bits” on 64-bit ISAs.
* Right-shift behavior clarified. Bits shifted out are lost. MSB fill depends on arithmetic vs logical.
* x86 addressing does not perform double dereference in one load.
* Accelerators still have loads and stores even if they add matrix ops.
* Systolic arrays date to the late 1970s, not the 1960s.
* Moore’s law spelling corrected.

# Arithmetic right shift (correct)

* Two’s-complement arithmetic right shift (SRA) copies the original sign bit into the new MSB. Bits shift right. The LSB falls off and is lost.
* Effect: roughly division by 2 for signed integers, rounding toward −∞ for negatives. It is **not** defined as “exact division” by the ISA; it is a sign-propagating shift.

Example, 4-bit registers:

* 1011₂ (−5) >>ₐ 1 → 1101₂ (−3).
* 1011₂ >>ₗ 1 (logical) → 0101₂ (+5).
* Register width is fixed. A shifted-out bit is discarded. The inserted bit is 0 for SRL, sign for SRA.

# Control flow classes (precise)

* **Conditional branches**: branch if a relation holds. Dominant in real code.
* **Unconditional control transfers**: jump, call, return.
* **Exceptions/interrupts**: control transfers triggered by events or faults.

  * **TLB specifics**: TLB = *Translation Lookaside Buffer*.

    * Some ISAs (classic MIPS) use **software-managed TLBs**; a TLB miss raises an exception to the OS handler.
    * Others (x86, AArch64) are **hardware-walked**; a TLB miss triggers a hardware page-table walk without an OS trap. Only a true page fault (not present/permission) traps.

# How ISAs implement conditional branches

There are three widespread models. Know what each ISA actually does.

1. **Condition codes / flags**

   * Arithmetic/logical ops set bits in a flags register (e.g., ZF, SF, CF, OF in x86; NZCV in ARM).
   * Branches test flags: JZ/JNZ, B.EQ/B.NE, etc.
   * Pros: comparison “free” piggybacks on ALU ops.
   * Cons: flags are implicit and get clobbered by later instructions; scheduling is tricky.

2. **Explicit compare into a GPR, then branch on register**

   * Compare writes 0/1 or a relation code into a general register; a branch tests that register.
   * **MIPS** pattern: `slt/sltu` (or immediates) produce 0/1, then `beq/bne` with `$zero`. No dedicated condition register.
   * Pros: explicit dataflow, easier rename/schedule.
   * Cons: one extra instruction in many cases.

3. **Compare-and-branch opcodes**

   * The branch instruction performs the comparison internally.
   * **RISC-V**: `BEQ, BNE, BLT, BGE, BLTU, BGEU` compare registers and branch. No flags.
   * **PowerPC** has a separate condition register (CR) that branches test; conceptually similar but with a CR file.

Notes and corrections:

* “Set less than or equal” is not a classic MIPS instruction. Use `slt`/`sltu` (and swap operands or invert) plus `beq/bne`.
* RISC-V’s branches are not “giant” in any problematic sense; they’re simple compare-and-branch encodings used because branches are frequent.

# MIPS instruction formats (corrected and detailed)

All encodings are 32 bits wide.

**R-type**:
`[ op(6)=0 ][ rs(5) ][ rt(5) ][ rd(5) ][ shamt(5) ][ funct(6) ]`

* Example `add rd, rs, rt`: `op=0`, `funct=0x20`.
* Shifts (e.g., `sll rd, rt, shamt`) use `shamt`; still R-type.

**I-type**:
`[ op(6) ][ rs(5) ][ rt(5) ][ imm(16) ]`

* Loads/stores: base=rs, dest/source=rt, 16-bit immediate (sign-extended in MIPS).
* Example `lw rt, imm(rs)`; example `addi rt, rs, imm` (traps on signed overflow), `addiu` avoids overflow trap.

**J-type**:
`[ op(6) ][ target(26) ]`

* The **jump target is not the full 32-bit address** in the instruction. Hardware forms:
  `target_address = { (PC+4)[31:28], target, 2'b00 }`
  i.e., take high 4 bits from `PC+4`, append 26 target bits, then `<<2` for 4-byte alignment.
* Reason for “×4”: instructions are 4-byte aligned; the encoding omits the low 2 zero bits.

# Decode + register read in one pipeline stage

Problem: you must **decode** to know which fields are sources, yet you want to **read** sources in the same cycle.

Solution used in simple pipelines (e.g., textbook MIPS-like):

* Optimistically read **both** `rs` and `rt` from the register file every cycle while the decoder determines the true role of each field.
* Also extract the immediate bits in parallel.
* The control logic then selects which operands are relevant for the current opcode.
  Trade-off: a little extra read work to collapse two conceptual steps into one pipeline stage.

# Five classic pipeline stages (with the right resource split)

1. **IF**: instruction fetch and `PC+4` computation. Typically hits the **L1 I-cache**; data accesses use **L1 D-cache** to avoid resource conflicts.
2. **ID**: decode, register fetch, immediate formation, early hazard checks.
3. **EX**: ALU work. Also effective-address add for loads/stores. Also branch comparison and target add for simple RISCs.
4. **MEM**: data cache access for loads/stores.
5. **WB**: write register results.

Critical constraints:

* Each stage needs independent hardware. No “single stool” contention between stages.
* Deepening a pipeline can raise **throughput** in theory but raises **latency**, **hazard cost**, and **branch penalties**. Real speedup << number of stages.

# Superscalar vs pipelining (history and accuracy)

* **Pipelining** overlaps different stages of different instructions using one copy of each stage.
* **Superscalar** issues multiple instructions per cycle by replicating or widening front-end/execute resources (multiple decoders, ALUs, LS units).
* Commercial superscalar cores appeared **in the early 1990s** (e.g., IBM RS/6000 1990, Intel Pentium 1993, DEC Alpha 1992). Not “late ’90s/2000s.”
* Modern cores are both pipelined and superscalar.

# Precise terminology and typo fixes

* **TLB** = *Translation Lookaside Buffer*, not “lucasite.”
* **Control flow**, not “control show.”
* **Superscalar**, not “supercala.”
* **RISC-V**, not “risk five.”
* **MIPS**, not “NIP/MIP.”
* Flags/condition codes are **not** used by MIPS or RISC-V. ARM and x86 use flags. PowerPC uses a condition-register file.

# Worked micro-examples

**1) MIPS branch using SLT pattern (no flags):**

```asm
# if (r1 >= r2) goto L
slt  t0, r1, r2     # t0=1 if r1<r2
beq  t0, $zero, L   # branch when NOT(r1<r2)
```

**2) RISC-V compare-and-branch:**

```asm
# if (x1 >= x2) goto L
bge  x1, x2, L
```

**3) Load and store (MIPS I-type):**

```asm
lw   rt, imm(rs)    # rt <- MEM[rs + signext(imm)]
sw   rt, imm(rs)    # MEM[rs + signext(imm)] <- rt
```

**4) Jump target formation (J-type):**

* Encoding carries `target[25:0]`. Hardware computes `{(PC+4)[31:28], target, 2’b00}`.

**5) SRA vs SRL effect on negatives:**

* `0xFFFF_FFFB` (−5) >>ₐ 1 → `0xFFFF_FFFD` (−3).
* `0xFFFF_FFFB` >>ₗ 1 → `0x7FFF_FFFD` (large positive).

# Pipeline hazard awareness preview

* **Data hazards**: read-after-write, write-after-write, write-after-read. Mitigate with bypassing and interlocks.
* **Control hazards**: branches; mitigate with prediction and target precompute.
* **Structural hazards**: resource conflicts; avoid via split I/D caches and adequate ports.
