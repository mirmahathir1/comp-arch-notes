# Answers to Selected Exam Questions

---

## Question 1: Conceptual Distinction

**Q:** A junior engineer claims: "Cache coherence and memory consistency are essentially the same thing—they both ensure that all processors see the same data." Critique this statement.

**Answer:**

The statement conflates two distinct concepts. **Cache coherence** is an *implementation* mechanism that makes caches invisible—ensuring that writes to a single memory location eventually become visible to all processors, as if caches didn't exist. **Memory consistency** is a *specification* that defines the ordering of loads and stores to *different* memory locations.

A system can be perfectly cache-coherent yet still surprise programmers because coherence says nothing about *when* or *in what order* writes to different addresses become visible. In the slides' example, even with coherent caches, P3 could see `A=1, C=0` because the store to A might overtake the store to C on the interconnect—a consistency issue, not a coherence failure.

---

## Question 2: Scenario Analysis

**Q:** Consider three processors with `X=0, Y=0, Z=0`:
```
P1:              P2:                   P3:
Z = 1;           while (X == 0);       while (Y == 0);
X = 1;           Y = 1;                print X, Z;
```

**(a) All possible outputs and explanations:**

| Output | Explanation |
|--------|-------------|
| `X=1, Z=1` | Expected case: all writes propagate in program order |
| `X=1, Z=0` | P1's store to X overtakes its store to Z (reordering within same processor) |
| `X=0, Z=1` | P2's store to Y overtakes P1's store to X on interconnect toward P3 |
| `X=0, Z=0` | Combination of both reorderings above |

**(b) Impossible under Sequential Consistency:**

`X=0, Z=0` and `X=0, Z=1` would be impossible. Under SC, P3 exiting its while loop means it saw `Y=1`, which means P2 exited its loop, which means P2 saw `X=1`. Since SC requires a global total order, P3 must also see `X=1`. Similarly, P1's writes must appear in order, so seeing `X=1` implies `Z=1` is also visible.

**(c) Mechanism to guarantee X=1, Z=1:**

Insert **memory fences** (barriers): P1 places a fence between `Z=1` and `X=1`; P2 places a fence between the while loop exit and `B=1`. This forces ordering. Alternatively, use **release-acquire semantics** on the flag variables or **atomic operations with appropriate memory ordering**.

---

## Question 3: Specification vs. Implementation

**Q:** The slides emphasize that memory consistency is a *specification* while cache coherence is an *implementation*.

**(a) Meaning and audience:**

Memory consistency is a **contract with programmers**—it defines what values reads can legally return, enabling reasoning about parallel programs. Cache coherence is a **contract with hardware designers**—it defines mechanisms (protocols like MESI) that make the memory system behave correctly. Programmers shouldn't need to know coherence details; they program against the consistency model.

**(b) Software-managed caches and consistency:**

No, removing hardware coherence would not inherently violate the memory consistency model. The consistency model is a specification of observable behavior, not a mandate for how to achieve it. Software-managed coherence (like some GPUs or the Cell processor) can still honor the consistency model—it's just a different implementation. The system would still need to provide *some* mechanism to make writes visible, but it needn't be automatic hardware protocols.

**(c) Why consistency is part of ISA:**

Because it affects program correctness. Code compiled for one consistency model may break on another. If consistency were a microarchitectural detail, the same binary could behave differently (produce different results) on different implementations of the same ISA—breaking the fundamental abstraction that ISA provides. Programmers and compilers need consistency guarantees to generate correct code.

---

## Question 4: Write Propagation and Serialization

**Q:** Cache coherence requires write propagation and write serialization. Define each and explain why both are necessary.

**(a) Definitions and necessity:**

**Write propagation:** A write to a location must eventually become visible to all processors. Without it, a processor could forever read stale data—P1 writes `X=1`, but P2 never sees it, continuing to read `X=0` indefinitely.

**Write serialization:** All processors observe writes to the same location in the same order. Without it, consider P1 writes `X=1`, then P2 writes `X=2`. Processor P3 might see the order as 1→2 (final value 2), while P4 sees 2→1 (final value 1). They'd permanently disagree on X's value despite both having "propagated" writes.

*Scenario with only propagation:* Two processors could see `X=1` then `X=2` vs. `X=2` then `X=1`, ending with different cached values.

*Scenario with only serialization:* A global order exists, but if writes don't propagate, some processors never learn about updates at all.

**(b) Does serialization guarantee propagation?**

No. Serialization only means that *if* processors see writes, they see them in the same order. It doesn't require that all processors actually receive the writes. You could theoretically serialize writes into an ordered log that some processors never read. Both properties are independently necessary.

---

## Question 7: Design Tradeoffs

**Q:** Design tradeoffs in memory consistency.

**(a) Optimizations prohibited by Sequential Consistency:**

1. **Write buffers with bypassing:** SC forbids letting a processor's own reads bypass its pending writes to different addresses, as this reorders stores and loads.

2. **Out-of-order execution of memory operations:** SC requires memory operations to appear to execute in program order. Hardware cannot speculatively complete loads before earlier stores are globally visible.

Other examples: non-blocking caches, overlapping write latencies, speculative loads.

**(b) Why vendors can't use maximally relaxed models:**

Software compatibility and programmer burden. Existing code assumes certain ordering guarantees; breaking them causes subtle, non-deterministic bugs. Weaker models require more explicit synchronization (fences everywhere), making code harder to write, port, and verify. The ecosystem of compilers, languages, and legacy software constrains architectural choices. ARM and x86 chose different tradeoffs, and this affects which code runs correctly on each.

**(c) Why language-level memory models exist:**

Languages like C++ and Java define memory models to provide **portability**—the same source code should behave consistently across different ISAs (x86, ARM, RISC-V) with different hardware consistency models. The compiler translates language-level synchronization (mutexes, atomics) into appropriate ISA-specific instructions and fences. This also enables **compiler optimizations**: the language model defines what reorderings compilers may perform, independent of hardware. Without this layer, programmers would need to write architecture-specific synchronization code.

# Answers: Sequential Consistency Exam Questions

---

## Question 1: Lamport's Definition

**Q:** Lamport's 1979 definition of Sequential Consistency contains two key requirements. A student summarizes SC as: "All processors see memory operations in the same order."

**(a)** Explain why this summary is incomplete. What critical aspect of SC does it omit?

**Answer:** The summary omits the **program order requirement**. SC requires not only a single total order visible to all processors, but also that each processor's operations appear in that total order in the sequence specified by its program. Without this, you could have a valid total order that scrambles an individual thread's operations arbitrarily.

---

**(b)** A system guarantees that all writes to a single variable are seen in the same order by all processors. Does this guarantee Sequential Consistency? Justify your answer.

**Answer:** No. This only guarantees **write serialization** (a cache coherence property), not SC. SC requires ordering across *different* variables too. The write-atomicity violation example shows P2 seeing `X=1, Y=0` while P3 sees `Y=1, X=0`—writes to different variables appear in different orders to different processors, violating SC even though writes to each individual variable are serialized.

---

**(c)** The slides note that "in practice many orderings are still valid" under SC. Explain why SC does not mandate a unique execution order, and what determines which orderings are valid.

**Answer:** SC only requires that *some* valid total order exists that is consistent with each processor's program order. Any interleaving of operations from different processors is valid, as long as per-processor program order is preserved. The nondeterminism comes from legitimate concurrency—when P1 and P2 have no dependencies, either's operation can come first in the global order.

---

## Question 2: SC-Consistent Executions

**Q:** Consider this code with initially `X=0, Y=0`:

```
P1:          P2:
Y = 1;       X = 1;
print X;     print Y;
```

**(a)** List all possible outputs under Sequential Consistency and provide a valid global ordering for each.

**Answer:**

| Output | Valid SC Ordering |
|--------|-------------------|
| P1 prints 1, P2 prints 1 | P1: Y=1 → P2: X=1 → P1: print X → P2: print Y |
| P1 prints 0, P2 prints 1 | P1: Y=1 → P1: print X → P2: X=1 → P2: print Y |
| P1 prints 1, P2 prints 0 | P2: X=1 → P2: print Y → P1: Y=1 → P1: print X |

---

**(b)** Can both processors print `0`? Prove your answer by either constructing a valid SC ordering or demonstrating that no such ordering exists.

**Answer:** No. For P1 to print `X=0`, P1's read of X must precede P2's write `X=1` in the global order. For P2 to print `Y=0`, P2's read of Y must precede P1's write `Y=1`. But program order requires P1's write before P1's read, and P2's write before P2's read. This creates a cycle: `Y=1 → read X → X=1 → read Y → Y=1`, which is impossible in any total order.

---

**(c)** The slides show that an execution with reordered operations can still be "SC" if its results match a valid SC execution. Explain the significance of this observation for hardware implementers.

**Answer:** Hardware doesn't need to literally execute operations in program order—it only needs to produce *results* (values returned by reads) that match some valid SC execution. This opens the door for optimizations like out-of-order execution, speculation, and write buffering, as long as the observable behavior remains SC-compliant. The contract is about outcomes, not mechanisms.

---

## Question 3: Write-Serialization vs. Write-Atomicity

**Q:** The slides distinguish between write-serialization and write-atomicity violations.

**(a)** Define each property and explain how they differ conceptually.

**Answer:** 

**Write-serialization:** All processors observe writes *to the same location* in the same order. It's about consistency of ordering for a single variable.

**Write-atomicity:** A write becomes visible to all processors at the same logical instant—no processor can see the write before another processor does. It's about simultaneity of visibility across processors.

Write-serialization concerns ordering; write-atomicity concerns visibility timing.

---

**(b)** In the write-serialization violation example, P2 sees `X=1` then `X=2`, while P3 sees `X=2` then `X=1`. Explain why this cannot happen in any system that maintains a single total order of writes.

**Answer:** A total order means either `X=1` comes before `X=2`, or vice versa—not both. If the global order is `X=1 → X=2`, then any processor reading both values must see 1 before 2 (or only see 2 if it reads after both). P2 and P3 observing opposite orders means no single total order exists—they're observing two incompatible histories, which is logically impossible under serialization.

---

**(c)** In the write-atomicity violation example, P2 reads `X=1, Y=0` while P3 reads `Y=1, X=0`. The writes are to *different* variables. Explain why this still violates SC.

**Answer:** SC requires a single total order of *all* operations. P2 seeing `X=1` means P1's write to X precedes P2's read in the global order. P2 seeing `Y=0` means P2's read precedes P4's write to Y. Similarly, P3's observations imply `Y=1` precedes its read, which precedes `X=1`. Combining these creates a cycle: the write to X must both precede and follow the write to Y in the global order—impossible. The write to X became visible to P2 before becoming visible to P3, violating atomicity.

---

## Question 4: Dekker's Algorithm Failure

**Q:** Consider the mutual exclusion attempt:

```
Initially flag1 = flag2 = 0

P1:                    P2:
flag1 = 1;             flag2 = 1;
if(flag2 == 0)         if(flag1 == 0)
{ /* critical */ }     { /* critical */ }
```

**(a)** Under Sequential Consistency, prove that both processors cannot simultaneously enter the critical section.

**Answer:** For both to enter, P1 must read `flag2=0` and P2 must read `flag1=0`. Under SC, there's a total order. Either `flag1=1` comes first or `flag2=1` comes first. If `flag1=1` is first, then when P2 reads flag1, it must see 1 (the write already happened in the global order), so P2 won't enter. Symmetric argument if `flag2=1` is first. In any valid SC ordering, at least one processor sees the other's flag set.

---

**(b)** Describe the precise sequence of events with write-buffering that allows both processors to enter.

**Answer:**
1. P1 executes `flag1=1`, which goes into P1's write buffer (not yet visible to memory/P2)
2. P2 executes `flag2=1`, which goes into P2's write buffer (not yet visible to memory/P1)
3. P1 reads `flag2` directly from memory/cache, sees 0 (P2's write still buffered)
4. P2 reads `flag1` directly from memory/cache, sees 0 (P1's write still buffered)
5. Both processors enter critical section
6. Later, both writes drain from buffers to memory

The write buffers allow reads to bypass pending writes to *different* addresses.

---

**(c)** A hardware designer proposes using write-through caches instead of write buffers. Evaluate this claim.

**Answer:** Write-through alone doesn't fix the problem. Write-through ensures writes eventually reach memory, but doesn't guarantee *when* relative to subsequent reads. If the write-through is non-blocking (processor continues before write completes), the same race exists. The fix requires either blocking writes (waiting for acknowledgment before proceeding) or memory fences that explicitly enforce ordering between the write and subsequent read.

---

## Question 5: Terminology Precision

**Q:** The slides distinguish between "issue," "performed w.r.t. processor X," and "globally performed."

**(a)** A store S by P1 to variable A is "performed w.r.t. P2" but not yet "globally performed." Describe a scenario where this matters.

**Answer:** P1 writes `A=1`. The write has propagated to P2 (P2 will see `A=1` on any subsequent read), but P3 hasn't received the update yet. P2 reads `A=1`; P3 reads `A=0`. This is the write-atomicity problem: the write is "performed" from P2's perspective but not globally. P2 and P3 have inconsistent views of memory at the same logical time, which SC forbids.

---

**(b)** Why does the definition of "load performed w.r.t. X" use *subsequent stores from X* rather than *all subsequent stores*?

**Answer:** A load is "performed w.r.t. X" when X has committed to the value—X's own future actions can't change what the load returns. Stores from *other* processors aren't relevant because they're not under X's control and happen asynchronously. The definition captures when the load is finalized from X's local perspective, not when it's globally consistent. Global consistency is captured separately by "globally performed."

---

**(c)** Why is "globally performed" essential for SC, whereas "performed w.r.t. X" alone is insufficient?

**Answer:** SC requires a single total order visible to all processors. "Performed w.r.t. X" only guarantees local consistency—X sees a consistent sequence. Different processors could have different "local" views that conflict with each other. "Globally performed" ensures the operation is finalized for *everyone* simultaneously, enabling the single global order SC requires. Without it, you get write-atomicity violations.

---

## Question 6: Sufficient Conditions for SC

**Q:** The slides list three sufficient conditions for SC.

**(a)** Explain why Condition 2 (wait until last operation completes globally before issuing next) guarantees program ordering is preserved.

**Answer:** If P1 issues operation A, waits until A is globally performed, then issues B, the global order must have A before B—A completed before B even began. There's no way for B to "overtake" A since B wasn't even issued until A finished. By serializing operations within each processor through global completion, program order is directly embedded into the global order.

---

**(b)** Construct a scenario where violating only Condition 3 (read completes only when matching write completes) leads to a non-SC result.

**Answer:** P1 writes `X=1`. P2 reads `X=1` from P1's cache via forwarding before P1's write is globally visible. P2 then writes `Y=1` (which completes globally). P3 sees `Y=1`, then reads `X=0` (P1's write still not global). Result: P2 saw `X=1` before P3, but P3 saw P2's subsequent `Y=1`. This violates SC—P2's read of X "happened" before P3's, yet P2's later write to Y is visible to P3 while the "earlier" X write isn't. The read completed without its source write completing.

---

**(c)** Give an example of a hardware optimization that violates these sufficient conditions yet still provides SC.

**Answer:** **In-window speculation** for loads: The processor issues load B before load A completes (violating Condition 2), executes speculatively, but commits in program order. If an invalidation arrives for B's address before commit, the speculation is squashed and replayed. This violates the *sufficient* conditions (operations issued out of order) but maintains SC because incorrect speculations are detected and corrected before becoming architecturally visible.

---

## Question 7: In-Window Speculation

**Q:** The slides describe techniques for implementing SC efficiently through speculation.

**(a)** For R→R optimization, explain the role of invalidation messages. What happens if an invalidation arrives after speculative execution but before commit?

**Answer:** Invalidation messages signal that another processor has written to a cache line. If load Y executes speculatively (before load X commits) and an invalidation for Y's address arrives before Y commits, the value Y read may be stale—another processor wrote a new value. The hardware squashes all speculative work from load Y onward and replays those operations with fresh data. This ensures the committed (visible) execution matches a valid SC ordering even though internal execution was out-of-order.

---

**(b)** For W→W optimization via write-prefetching, explain why obtaining exclusive permissions early doesn't violate SC.

**Answer:** Obtaining read-exclusive permission (via coherence protocol) is *preparation*, not *completion*. The write isn't visible to other processors until the data is actually written and the cache line can be read. By obtaining permissions for both X and Y in parallel but writing (making visible) X before Y, the observable order matches program order. The coherence traffic is an implementation detail; what matters for SC is when values become readable by others, which still happens in program order.

---

**(c)** Why is W→R speculation (read before prior write completes) more problematic than R→R?

**Answer:** With R→R, both operations only *observe* memory—squashing and replaying just re-reads values. With W→R, the write *modifies* memory. If the read executes before the write is visible and another processor reads the same location, that processor sees the old value, while the local read might see the new value (via store-to-load forwarding). The write has "escaped" to the local read before becoming globally visible. Fixing this requires tracking whether any external read could have observed the memory between the speculative read and the write's completion—much more complex than tracking invalidations.

---

## Question 8: Reasoning About Executions

**Q:** The execution shows P1's operations as `st A` then `st C`, but the original program has `C=1` before `A=1`.

**(a)** Explain precisely why this reordering is acceptable for SC compliance.

**Answer:** SC is defined by *results* (values returned by reads), not operation order. No processor in this execution reads C, so reordering C and A produces identical read values. Since there exists a valid SC execution (shown on the right: `st C → st A → ...`) that produces the same read values, the reordered execution is SC-compliant. The reordering is unobservable—it doesn't affect any read in the program.

---

**(b)** Suppose P3 also executed `print C` at the end and observed `C=0`. Could this execution still be SC-compliant?

**Answer:** No. If P3 reads `C=0` after reading `B=1`, trace the dependencies: `B=1` requires P2 saw `A=1`, which in any SC order means P1's `st A` happened. Program order requires `st C` before `st A` in P1. So in any valid SC order, `st C` precedes `st A` precedes `ld A` (by P2) precedes `st B` precedes `ld B` (by P3) precedes `ld C` (by P3). Thus P3 must see `C=1`. Observing `C=0` proves no valid SC ordering exists.

---

**(c)** Under what conditions can hardware reorder operations from the same processor without violating SC?

**Answer:** Hardware can reorder operations when the reordering is **unobservable**—when no possible execution (by any processor) could distinguish the reordered execution from a program-order execution. This happens when: (1) the operations are to different addresses with no intervening dependencies, AND (2) no other processor reads the affected locations in a way that would reveal the order. Practically, hardware speculates and verifies (via invalidations and conflict detection) rather than proving non-observability statically.

# Answers: Relaxed Memory Consistency Models

---

## Question 1: Model Comparison and Classification

**Q(a):** A programmer writes correct synchronization code for Sequential Consistency. When porting to TSO, they discover the code still works correctly. When porting to Release Consistency without modifications, it breaks. Explain what category of synchronization pattern would exhibit this behavior, and why TSO preserves correctness while RC does not.

**Answer:** The pattern is **flag-based synchronization** like `data=1; flag=1;` paired with `while(!flag); read data;`. TSO preserves W→W ordering (writes complete in program order), so `data=1` is visible before `flag=1`. RC relaxes W→W ordering, so `flag=1` could become visible before `data=1`, causing the reader to see stale data. The pattern requires explicit `release` on the flag write and `acquire` on the flag read under RC.

---

**Q(b):** The slides mention that IBM Power and NVIDIA PTX relax write atomicity in addition to all four memory orderings. Explain what "relaxing write atomicity" means and describe a scenario where this creates behaviors impossible under TSO.

**Answer:** Relaxed write atomicity means a write can become visible to different processors at different times—it's not an instantaneous global event. Example:

```
P1: X=1    P2: r1=X; r2=Y    P3: Y=1    P4: r3=Y; r4=X
```

Under Power, P2 could see `X=1, Y=0` while P4 sees `Y=1, X=0`—each write propagated to one observer before the other. Under TSO, write atomicity is preserved: if P2 sees `X=1`, then any processor that later sees P2's effects must also see `X=1`.

---

**Q(c):** Why do relaxed memory models universally preserve single-thread dependencies, even when they relax inter-thread ordering? What would break if this guarantee were removed?

**Answer:** Single-thread dependencies (like `X=2; r1=X` always reading 2) preserve the illusion of sequential execution within a thread. Removing this would break **all** programs, not just concurrent ones—basic sequential code would become unpredictable. A compiler or programmer could never reason about local behavior. This is the minimal contract that makes programming possible; inter-thread relaxations only affect shared-memory communication patterns.

---

## Question 2: TSO Deep Dive

**Q(a):** Consider with initially `A=0, B=0`:
```
P1:              P2:
A = 1;           B = 1;
r1 = A;          r3 = B;
r2 = B;          r4 = A;
```
What values must r1 and r3 read, and why?

**Answer:** r1 must read 1, and r3 must read 1. This is due to **single-thread data dependencies**: each processor reads a variable it just wrote. Even though TSO allows reads to bypass *other* pending writes in the write buffer, a processor always sees its own writes immediately (store-to-load forwarding). This is preserved in all memory models.

---

**Q(b):** Can the execution result in `r2=0` and `r4=0` simultaneously? Provide a detailed explanation of the hardware mechanism (write buffers) that enables or prevents this outcome.

**Answer:** **Yes**, this is the classic TSO anomaly. The mechanism:

1. P1 writes `A=1` into its write buffer (not yet visible to P2)
2. P2 writes `B=1` into its write buffer (not yet visible to P1)
3. P1 reads B from memory/cache, sees `B=0` (P2's write still buffered) → r2=0
4. P2 reads A from memory/cache, sees `A=0` (P1's write still buffered) → r4=0
5. Later, both write buffers drain to memory

TSO allows reads to bypass pending writes to *different* addresses, enabling both loads to execute before either store becomes globally visible.

---

**Q(c):** The slides show that TSO still makes the "flag-based synchronization" pattern work correctly. Explain why W→R reordering doesn't break this pattern, but would break Dekker's mutual exclusion algorithm.

**Answer:** In flag-based synchronization (`data=1; flag=1` / `while(!flag); read data`), the critical ordering is **W→W** (data before flag) on the producer side. TSO preserves W→W, so when the consumer sees `flag=1`, `data=1` is already visible.

Dekker's algorithm (`flag1=1; if(flag2==0) enter`) requires **W→R** ordering: the write to my flag must be visible before I read the other flag. TSO breaks this—each processor can read the other's flag (seeing 0) before its own write becomes visible, allowing both to enter the critical section.

---

## Question 3: Release Consistency Semantics

**Q(a):** The slides state three constraints for RC. Explain why the constraint "all synchronization operations must be sequentially consistent" is necessary. What could go wrong if acquires and releases could be reordered with respect to each other?

**Answer:** If synchronization operations weren't SC with respect to each other, you couldn't establish a consistent "happens-before" relationship across threads. Example: if P1's release could be reordered after P2's acquire that observes it, the acquire might not see all the writes that preceded the release. The SC ordering of synchronization points creates the "skeleton" of global ordering that gives meaning to acquire/release semantics. Without it, there's no consistent way to determine what one thread's acquire should observe from another's release.

---

**Q(b):** Consider the "allowable overlaps" diagram. Block 1 operations can move past the acquire (into block 2's region), and block 3 operations can move before the release. Explain why this doesn't violate the intuitive semantics of critical sections.

**Answer:** The critical section semantics require that operations *inside* (block 2) don't escape—they must appear between acquire and release to other threads. The constraints ensure:

- Block 1 operations moving *in* is safe: they're just completing earlier, still protected
- Block 3 operations moving *in* is safe: they're starting earlier, still protected
- Operations cannot move *out*: block 2 can't escape past acquire or release

The "roach motel" model: operations can move into the critical section but never out. Other threads always see block 2 operations as occurring atomically between synchronization points.

---

**Q(c):** Under RC, if P2's acquire succeeds after P1's release, is P2 guaranteed to see both `x=1` and `y=2`?
```
P1:                     P2:
x = 1;                  acquire(lock);
y = 2;                  r1 = y;
release(lock);          r2 = x;
                        release(lock);
```

**Answer:** **Yes**. RC's constraint states "all previous writes must complete before a release can complete." When P1 executes release(lock), both `x=1` and `y=2` are globally visible. The second constraint states "no subsequent reads can complete before a previous acquire completes." When P2's acquire completes (after observing P1's release due to SC ordering of synchronization), all of P2's subsequent reads see memory state that includes P1's pre-release writes. The acquire-release pair creates a "synchronizes-with" relationship guaranteeing visibility.

---

## Question 4: Control vs. Address Dependencies

**Q(a):** Explain why hardware can reorder the load of X before the branch resolution in the control dependency case, but cannot reorder the load through r1 in the address dependency case.

**Control dependency:**
```
P2: r1 = Y; if(r1 == 1) r2 = X;  // r2 can read 0
```

**Address dependency:**
```
P2: r1 = Y; r2 = *(r1);  // r2 must read 1
```

**Answer:** With **control dependency**, the processor knows the *address* (X) before the branch resolves—it can speculatively load X, then discard or keep the result based on the branch. The load doesn't *depend* on r1's value for knowing where to load from.

With **address dependency**, the processor cannot even issue the load until r1's value is known—the address itself comes from r1. There's a true data dependency: you can't load from `*(r1)` without first knowing r1. This is a fundamental microarchitectural constraint, not a memory model choice.

---

**Q(b):** A compiler optimization replaces `if(r1 == 1) r2 = X;` with `r2 = X; if(r1 != 1) r2 = 0;`. Under what circumstances could this transformation be problematic?

**Answer:** This transformation converts a control dependency into **no dependency**—the load of X now executes unconditionally before checking r1. Under SC or TSO, this might be acceptable (X is loaded anyway). Under RC/ARM/RISC-V, this breaks synchronization: the original code had an implicit ordering (branch depends on r1, load in branch depends on branch), but the transformed code can load X before r1 is even read. If r1 was meant to be an acquire-like flag, this defeats the synchronization. Compilers must understand the memory model to avoid such transformations on potentially-synchronizing code.

---

**Q(c):** How should a programmer fix the control dependency example to guarantee that r2 reads 1 when r1 reads 1?

**Answer:** Add explicit synchronization. Options:

1. **Make P1's write a release and P2's read an acquire:**
```
P1: X = 1; release: Y = 1;
P2: acquire: r1 = Y; if(r1 == 1) r2 = X;
```

2. **Add a fence in P2:**
```
P2: r1 = Y; if(r1 == 1) { fence; r2 = X; }
```

3. **Use an acquire load for Y in P2:**
```
P2: r1 = acquire_load(Y); if(r1 == 1) r2 = X;
```

The acquire ensures all subsequent reads (including X) see values at least as recent as what was visible when Y was read.

---

## Question 5: Implementing RC - Two Approaches

**Q(a):** In the writer-initiated approach, "invalidates need not snoop LSQ" (Load-Store Queue). Explain why this simplification is possible under RC but would be incorrect under SC or TSO.

**Answer:** Under SC/TSO, a load that has speculatively executed might be invalidated by a remote write, requiring the load to be squashed and replayed—hence invalidations must snoop the LSQ to detect this. Under RC, loads between synchronization points have **no ordering requirements** with respect to remote writes. If a speculative load reads a stale value that gets invalidated, it doesn't matter—there's no acquire that would require seeing the new value. Only at acquire points must visibility be enforced, and that's handled separately. The LSQ snooping complexity is eliminated.

---

**Q(b):** In the self-invalidation approach, a release "writes all dirty blocks to coherent lower-level cache" and an acquire "forces a miss and self-invalidates all valid blocks." Explain why this achieves correctness.

**Answer:** Correctness comes from the synchronization points:

- **Release:** Flushing dirty blocks ensures all writes before the release are visible in the shared cache/memory. Any subsequent acquire at another processor can see them.

- **Acquire:** Self-invalidating forces all subsequent reads to fetch fresh data from the coherent lower-level cache, which contains all writes from completed releases.

Between synchronization points, processors may have stale data—but that's allowed under RC. The model only guarantees visibility across acquire-release pairs, and the flush-then-invalidate sequence ensures this. No continuous coherence traffic is needed.

---

**Q(c):** The slides ask: "Is the cache still coherent?" with the self-invalidation approach, answering "Yes." Justify this answer.

**Answer:** Cache coherence requires two properties:

1. **Write propagation:** Writes eventually become visible to all processors. ✓ Satisfied because releases flush dirty data to the shared cache, and acquires fetch from it.

2. **Write serialization:** All processors see writes to the same location in the same order. ✓ Satisfied because the lower-level cache (LLC/memory) is the serialization point. All writes go through it (at release), and all reads come from it (after acquire). The shared cache provides a single consistent order.

The caches are "lazily coherent"—coherence is enforced at synchronization points rather than continuously, but the fundamental properties hold.

---

## Question 6: Barrier Types and Write Atomicity

**Q(a):** `lwsync` ensures R→R, R→W, and W→W ordering but not W→R, and doesn't ensure write atomicity. `sync` ensures all four orderings plus write atomicity. When would a programmer choose `lwsync` over `sync`?

**Answer:** `lwsync` is **cheaper** (less stalling, less coherence traffic) because it doesn't drain write buffers or wait for write atomicity. Use `lwsync` when:

- Implementing **release semantics** (all prior writes complete before release)—W→R ordering isn't needed for release
- Producer-consumer patterns where you're publishing data, not implementing mutual exclusion
- Situations where write atomicity doesn't matter (only two processors communicating directly)

Use `sync` when:
- Implementing mutual exclusion (Dekker-style) requiring W→R ordering
- Multi-processor coordination where write atomicity matters (more than two processors must agree on write order)
- Full memory fence semantics needed

---

**Q(b):** Construct a scenario with three processors where using `lwsync` (without write atomicity) produces a different outcome than using `sync`.

**Answer:**
```
Initially X = 0, Y = 0

P1:              P2:                    P3:
X = 1;           r1 = X; // sees 1     r2 = Y; // sees 1
                 lwsync;                r3 = X; // sees 0?
                 Y = 1;
```

With `lwsync`: P2 sees `X=1` and writes `Y=1`. But `X=1` might not yet be visible to P3 when P3 sees `Y=1`. Result: `r2=1, r3=0` is possible. The write to X wasn't atomic—it was visible to P2 but not yet to P3.

With `sync`: Write atomicity ensures that if P2 saw `X=1` and P3 later sees P2's `Y=1`, P3 must also see `X=1`. Result: `r2=1, r3=0` is **impossible**.

---

**Q(c):** Map release store and acquire load to barrier-based thinking. What ordering constraints does each provide, and how do they compare to `lwsync` and `sync`?

**Answer:**

**Release store** provides:
- All prior reads and writes complete before the release (R→Rel, W→Rel)
- Equivalent to `lwsync` followed by a store
- Does NOT order subsequent operations—things after release can move earlier

**Acquire load** provides:
- All subsequent reads and writes wait for the acquire (Acq→R, Acq→W)
- Equivalent to a load followed by `lwsync`
- Does NOT order prior operations—things before acquire can move later

**Key differences from barriers:**
- Release/acquire are **one-directional** (release blocks upward, acquire blocks downward)
- `lwsync` is **bidirectional** but doesn't provide W→R
- `sync` is fully bidirectional plus write atomicity
- Acquire-release pairs are more targeted and potentially more efficient than full fences

---

## Question 7: Hardware Optimization Opportunities

**Q(a):** For each hardware feature, identify which ordering relaxation enables it:

- **Write buffers (store buffers)**
- **Non-blocking caches**  
- **Out-of-order load execution**

**Answer:**

**Write buffers:** Enabled by relaxing **W→R** ordering. Writes enter the buffer and reads can bypass them (to different addresses). TSO enables this. Under SC, reads must wait for all prior writes to complete.

**Non-blocking caches:** Enabled by relaxing **R→R** ordering. A cache miss doesn't block subsequent loads to different addresses. Under SC, loads must complete in order. PSO/RMO/RC enable this.

**Out-of-order load execution:** Enabled by relaxing **R→R** (for independent loads) and eliminating LSQ snooping requirements. Under SC/TSO, speculative loads must be checked against invalidations. RC removes this requirement between synchronization points.

---

**Q(b):** Under RC with self-invalidation, "local data need not be written to lower-level upon release" and "local data need not be self-invalidated upon acquire." What constitutes "local data," and how could hardware track this?

**Answer:** **Local data** is data that is only accessed by one processor—private to a thread, not shared. Examples: stack variables, thread-local storage, private heap allocations.

**Tracking mechanisms:**

1. **Directory-based:** The coherence directory knows if a cache line has only one sharer. Lines with single owner are "local."

2. **TLB-based:** Page table entries can mark pages as private/shared. The TLB provides per-access classification. OS tracks sharing at page granularity.

3. **Compiler hints:** Special instructions or memory regions marked as thread-local bypass coherence.

Benefit: Local data skips expensive release flushes and acquire invalidations, significantly reducing synchronization overhead for private data.

---

**Q(c):** A processor designer claims: "We can implement TSO more efficiently than SC, but RC provides no additional benefit over TSO for our design." Under what assumptions might this be true or false?

**Answer:**

**Claim might be TRUE if:**
- The design has simple in-order cores where W→W and R→R orderings are naturally maintained
- Write buffers already provide the main optimization (TSO's W→R relaxation)
- The coherence protocol is snooping-based with low latency, so invalidation snooping of LSQ is cheap
- Workloads are synchronization-heavy, so RC's benefits between sync points are minimal

**Claim would be FALSE if:**
- The design has aggressive out-of-order cores that could exploit R→R and W→W relaxation
- The system is large-scale (many cores) where eliminating LSQ snooping significantly reduces complexity
- The coherence protocol could be simplified (no sharer tracking, self-invalidation) under RC
- Workloads have large regions between synchronization points where RC allows more reordering

---

## Question 8: Reasoning About Programs

**Q:** Consider with initially `X=0, Y=0, Z=0`:
```
P1:              P2:              P3:
X = 1;           while(Y==0);     while(Z==0);
Y = 1;           Z = 1;           r1 = X;
```

**Q(a):** Under Sequential Consistency, what value must r1 contain? Prove your answer.

**Answer:** r1 must be **1**. 

Proof: P3 exits its loop, so it observed `Z=1`. Under SC's total order, P2's `Z=1` preceded P3's read. P2 only writes `Z=1` after exiting its loop, meaning P2 observed `Y=1`. Therefore P1's `Y=1` preceded P2's read. P1's `X=1` precedes P1's `Y=1` (program order preserved in SC). Combining: `X=1` → `Y=1` → P2 sees Y → `Z=1` → P3 sees Z → P3 reads X. Since `X=1` is earlier in the total order than P3's read, P3 must see `X=1`.

---

**Q(b):** Under TSO, can r1 ever be 0? Explain why or why not.

**Answer:** **No**, r1 must still be 1.

TSO only relaxes W→R ordering (reads bypassing writes to *different* addresses). The critical orderings here are:

- P1: X=1 before Y=1 → **W→W preserved** under TSO ✓
- P2: Read Y before write Z → **R→W preserved** under TSO ✓  
- P3: Read Z before read X → **R→R preserved** under TSO ✓

All the orderings in the dependency chain are preserved by TSO. The "message passing" pattern through Y and Z correctly propagates the visibility of X.

---

**Q(c):** Under Release Consistency (without explicit synchronization annotations), can r1 be 0?

**Answer:** **Yes**, r1 can be 0 under RC.

RC relaxes **W→W** ordering. P1's writes `X=1` and `Y=1` can be reordered: `Y=1` could become visible before `X=1`. Execution:

1. P1's `Y=1` becomes globally visible (X=1 still in write buffer)
2. P2 sees Y=1, exits loop, writes Z=1
3. P3 sees Z=1, exits loop, reads X, sees 0 (P1's X=1 not yet visible)
4. Later P1's X=1 becomes visible

Without release on Y and acquire on Y's read, the W→W ordering isn't enforced.

---

**Q(d):** Rewrite the code with minimal RC synchronization annotations to guarantee r1=1.

**Answer:**

```
P1:                  P2:                      P3:
X = 1;               acquire: while(Y==0);   acquire: while(Z==0);
release: Y = 1;      release: Z = 1;         r1 = X;
```

The **release** on Y ensures X=1 is visible before Y=1. The **acquire** on Y ensures P2 sees X=1 when it sees Y=1. The **release** on Z ensures P2's view (including X=1) propagates. The **acquire** on Z ensures P3 sees everything P2 saw. This creates a "synchronizes-with" chain: P1 →(Y)→ P2 →(Z)→ P3.