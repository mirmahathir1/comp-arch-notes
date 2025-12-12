## Selected Exam Questions with Answers

---

**Question 4(a):** Consider the producer-consumer code. Explain why this code could produce incorrect results even on a system with perfect cache coherence.

**Answer:** Cache coherence only guarantees that writes eventually become visible to other processors, but it doesn't guarantee *when* or *in what order*. The processor or compiler may reorder the producer's instructions, executing `flag = 1` before `data = 10` completes. Similarly, the consumer might speculatively read `data` before observing `flag = 1`. Thus the consumer could see `flag = 1` but still read a stale value of `data`. This is a memory consistency issue, not a coherence issue—coherence ensures visibility, but consistency defines ordering guarantees.

---

**Question 5(b):** Cache coherence protocols must ensure both "write propagation" and "write serialization." Which property is more challenging to implement in a NUMA system with hundreds of cores? Justify your answer.

**Answer:** Write serialization is more challenging. Write propagation only requires that a write eventually reaches all caches—this can be done through point-to-point messages or hierarchical broadcasts. Write serialization, however, requires that *all processors observe writes to the same location in the same order*, demanding global agreement. In a large NUMA system, establishing this total order requires coordination across physically distant nodes with varying latencies, often necessitating a centralized ordering point or complex distributed protocols that become bottlenecks at scale.

---

**Question 8(b):** The slides mention memory fences as hardware support for synchronization. When would a programmer need to use a memory fence even when using proper locking?

**Answer:** Memory fences are needed when communicating between threads *without* locks, such as in lock-free algorithms or the flag-based synchronization pattern shown in the slides. Even with proper locking, fences may be needed when accessing shared data outside the critical section (e.g., double-checked locking patterns) or when implementing the locks themselves. Additionally, if code reads shared variables to make decisions about whether to acquire a lock, fences ensure those reads reflect current values. Lock implementations typically include implicit fences, but DIY synchronization requires explicit ones.



# Short Answers to Selected Questions

---

## Question 1

Consider the producer-consumer code pattern where the producer sets data = 10 followed by flag = 1, and the consumer spins on while (!flag) {} before reading data.

**Part A: Explain why, without memory fences, the consumer might observe `flag = 1` but still read a stale value of `data`. Identify at least two distinct hardware/software mechanisms that could cause this reordering.**

Two mechanisms that can cause this problem:

1. **Store buffer reordering**: The producer's CPU may have both writes sitting in its store buffer, but they drain to cache/memory out of order. The `flag = 1` write may become visible before `data = 10`.

2. **Compiler reordering**: The compiler may reorder the stores to `data` and `flag` since they appear independent, placing the flag write before the data write in the generated code.

Additionally, on the consumer side, speculative loads or load reordering could cause the read of `data` to execute before the read of `flag` completes, fetching a stale cached value.

---

**Part B: Where must memory fences be placed in both the producer and consumer to guarantee correctness? Explain what each fence prevents.**

**Producer**: A fence must be placed between `data = 10` and `flag = 1`. This fence ensures all prior stores (including `data = 10`) are visible to other processors before the store to `flag` becomes visible. It prevents store-store reordering.

**Consumer**: A fence must be placed between `while (!flag) {}` and `x = data * y`. This fence ensures the load of `flag` completes and is observed before the load of `data` occurs. It prevents load-load reordering and ensures the consumer doesn't speculatively read stale data.

---

## Question 2

The slide shows four processors P1-P4 where P1 writes X=1 and P4 writes X=2. Processor P2 observes X=1 then X=2, while P3 observes X=2 then X=1.

**Part A: Explain precisely why this observation pattern violates cache coherence, even though each processor eventually sees both writes.**

Write serialization requires that all processors observe writes to the same location in the same order. If P2 sees X=1 then X=2, but P3 sees X=2 then X=1, there is no single global ordering of the writes that both processors agree upon. Coherence doesn't just require eventual visibility—it requires a consistent total order of writes that all observers agree on. This violation means the system cannot establish a coherent "history" for location X.

---

**Part B: Describe how a coherence protocol enforcing SWMR (Single-Writer-Multiple-Reader) would prevent this scenario.**

SWMR ensures only one processor can write to a location at any given time. Before P1 can write X=1, it must obtain exclusive ownership, invalidating or blocking all other copies. Before P4 can write X=2, it must similarly obtain exclusive ownership, which requires waiting for P1's write to complete and propagate. This serializes the writes—either X=1 happens globally before X=2, or vice versa. Since writes are serialized at the point of obtaining exclusivity, all processors observe them in that same serial order.

---

**Part C: Could this violation occur if all four processors shared a single L2 cache? Why or why not?**

No, this violation could not occur with a shared L2 cache. The shared L2 acts as a natural serialization point—all writes to X must pass through this single cache, which imposes a total ordering on the writes. Whichever write reaches the L2 first becomes globally ordered first. All processors accessing X through this shared L2 will observe writes in the order they were serialized at the L2, preventing any disagreement about write ordering.

---

## Question 3

**Part A: A program has one producer thread repeatedly writing to a shared variable and eight consumer threads that each read the variable once after all writes complete. Which coherence protocol (invalidate or update) would generate more coherence traffic?**

**Update generates more traffic.** With update, every write by the producer sends the new value to all 8 consumer caches, even though they don't need intermediate values—they only read once at the end. If the producer writes N times, update sends 8N update messages.

With invalidate, the first write invalidates all copies (8 messages), and subsequent writes require no coherence traffic since no one has re-cached the data. When consumers finally read, they each fetch the final value (8 misses). Total: 8 + 8 = 16 messages regardless of N.

---

**Part B: Now consider a different scenario: one thread writes to a variable 100 times, and another thread reads it once at the end. Analyze the traffic for both protocols.**

**Invalidate**: First write sends 1 invalidation. Remaining 99 writes generate no traffic (writer has exclusive ownership). Final read causes 1 cache miss. **Total: 2 coherence events.**

**Update**: Each of the 100 writes sends an update to the other cache (even though it's not reading). **Total: 100 update messages**, and 99 of them are completely wasted since only the final value matters.

Invalidate is dramatically more efficient for this write-heavy, read-light pattern.

---

**Part C: Explain why invalidate protocols are typically paired with write-back caches while update protocols pair with write-through caches. What fundamental design philosophy connects each pairing?**

**Invalidate + Write-back philosophy**: Minimize communication until absolutely necessary. Write-back delays writing to memory; invalidate delays sharing new data with others. Both assume other caches/memory don't need immediate updates. The writer accumulates changes locally and only communicates when someone else requests the data.

**Update + Write-through philosophy**: Propagate changes immediately and everywhere. Write-through sends every write to memory; update sends every write to other caches. Both keep all copies current at all times. Memory and all caches always have the latest value, enabling simple reads directly from memory.

The pairing reflects a consistent choice between lazy (defer communication) versus eager (immediate propagation) coherence strategies.

---

## Question 4

**Part B: A student claims: "SWMR guarantees that if processor P1 writes to address A, no other processor can have a cached copy of A." Is this statement correct? If not, provide a precise correction.**

The statement is **incorrect**. 

**Correction**: SWMR guarantees that if processor P1 is writing to address A, no other processor can have a *valid* cached copy of A. Other processors may still have the cache line present, but it must be in the Invalid state. The key distinction is that SWMR allows either: (1) a single writer with no valid copies elsewhere, OR (2) multiple readers with no writer. During the writer epoch, other caches' copies are invalidated but not necessarily evicted—they remain as invalid entries that will miss on access.

---

## Question 6

**Part A: Explain why software coherence using `writeback` and `self-invalidate` instructions requires "conservatism of static analysis." Give a concrete example where the compiler might insert unnecessary coherence operations.**

Static analysis cannot always determine at compile time whether memory will actually be shared or which addresses will be accessed. The compiler must conservatively assume the worst case.

**Example**: Consider a function that takes a pointer parameter. The compiler cannot know if multiple threads will pass pointers to the same memory or different memory. It must insert writeback/invalidate instructions assuming the pointer could reference shared data, even if at runtime the memory is thread-private. Similarly, in a loop over an array, the compiler might not know the array bounds overlap with another thread's working set, so it conservatively invalidates the entire array range even if only a small portion is actually shared.

---

**Part B: What fundamental advantage does hardware coherence provide over software coherence in terms of correctness guarantees?**

Hardware coherence provides **transparent, automatic correctness** without programmer or compiler involvement. The hardware tracks sharing at runtime and takes coherence actions precisely when needed—only for data that is actually shared, and only at the exact moment conflicts occur. This eliminates the possibility of programmer error (forgetting a writeback) and removes the conservatism of static analysis (unnecessary invalidations). Correctness is guaranteed regardless of sharing patterns, pointer aliasing, or dynamic behavior that cannot be predicted at compile time.

---

**Part C: Despite the advantages of hardware coherence, some systems (particularly GPUs and accelerators) use software-managed coherence. Propose two reasons why a system designer might make this choice.**

1. **Reduced hardware complexity and power**: Hardware coherence requires snooping logic, coherence directories, additional state bits per cache line, and complex protocol state machines. For systems with hundreds or thousands of cores (like GPUs), this overhead is prohibitive. Software coherence shifts this burden to the programmer/compiler, allowing simpler, more power-efficient cache designs.

2. **Predictable performance and control**: Hardware coherence can cause unpredictable latency spikes when coherence traffic occurs. In throughput-oriented architectures, programmers often prefer explicit control over when data movement happens. Software coherence lets programmers schedule data transfers to hide latency and avoid coherence traffic during critical computation phases, enabling more predictable performance optimization.

---

## Question 8

You are designing a cache coherence protocol for a system where most shared data is read-mostly (written rarely, read frequently by many processors).

**Part A: Would you choose an invalidate or update-based protocol? Justify your choice based on expected traffic patterns.**

**Update protocol** would be better for read-mostly data. 

When data is written rarely and read frequently by many processors, update ensures all readers maintain valid cached copies after each infrequent write. With invalidate, each write would invalidate all reader copies, causing N cache misses when N readers next access the data. For read-mostly patterns, these miss penalties occur frequently (many reads) due to rare writes.

With update, the rare writes broadcast the new value once, and all subsequent reads hit locally. The update traffic is proportional to write frequency (low), while invalidate's miss penalty is proportional to read frequency × number of readers (high).

---

**Part B: Your system also has some data that exhibits migratory sharing (repeatedly written by different processors in turn). How does this pattern affect your protocol choice?**

Migratory sharing strongly favors **invalidate protocol**, complicating the choice.

With migratory data, processor P1 writes, then P2 writes, then P3 writes, etc. Under update, each write broadcasts to all other caches, most of which will never read the value before it's overwritten—wasted traffic. Under invalidate, each write only needs to invalidate the previous writer's copy (one message), and no cache miss occurs for writers because they obtain exclusive access directly.

This creates a tension: update is better for read-mostly data, invalidate is better for migratory data. A simple single-protocol choice requires accepting suboptimal performance for one pattern.

---

## Question 9

**Part A: Explain the relationship between cache coherence and memory consistency. Can a system be coherent but have a weak consistency model? Can it be consistent but incoherent?**

Coherence governs the ordering of accesses to a **single memory location**—ensuring all processors see a consistent sequence of values for each address. Consistency governs the ordering of accesses across **multiple memory locations**—defining what orderings between operations to different addresses are legal.

**Coherent but weakly consistent**: Yes, this is common. Most modern systems are coherent (single-location ordering is guaranteed) but use relaxed consistency (cross-location ordering is relaxed). Coherence is essentially a prerequisite—you can build any consistency model on top of a coherent system.

**Consistent but incoherent**: No, this is not meaningful. Without coherence, you cannot guarantee even single-location ordering, which undermines any consistency model. Consistency assumes coherence as a foundation.

---

**Part B: The producer-consumer example uses fences for synchronization. If the system had sequential consistency, would the fences still be necessary? Explain.**

**No, fences would not be necessary.** Sequential consistency guarantees that all memory operations appear to execute in some total order that respects each processor's program order. This means:

- The producer's `data = 10` must appear before `flag = 1` in the global order (program order preserved)
- The consumer's `flag` read must appear before the `data` read (program order preserved)
- If the consumer sees `flag = 1`, the producer's flag write already occurred, which means `data = 10` already occurred before it

The program would be correct without fences because SC preserves all program orderings automatically. Fences exist precisely because real systems use relaxed consistency to enable hardware optimizations.

---

**Part C: Why do modern processors typically implement relaxed consistency models despite the programming complexity this creates?**

Relaxed consistency enables critical performance optimizations:

1. **Store buffers**: Writes can be buffered and committed out of order, hiding memory latency and allowing the processor to continue executing. SC would require stalling until each store completes globally.

2. **Load speculation**: Processors can execute loads early, out of program order, improving instruction-level parallelism. SC would require waiting for all prior operations.

3. **Write combining**: Multiple writes to nearby addresses can be combined into single bus transactions, reducing memory traffic.

4. **Cache hierarchy flexibility**: Different levels of cache can have different views temporarily, without expensive synchronization on every access.

The performance cost of strict ordering (stalls, serialization, reduced parallelism) outweighs the programming complexity, especially since most code is either single-threaded or uses synchronization libraries that correctly employ fences. The hardware provides fast common-case execution while offering fence instructions for the rare cases requiring ordering guarantees.

# Cache Coherence Protocols - Exam Questions with Answers

---

## Question 1: Protocol State Analysis

Consider a system with two processors P0 and P1 using the MSI cache coherence protocol. Both caches are initially empty (all lines Invalid). The following sequence of operations occurs on memory address X:

1. P0 reads X
2. P1 reads X  
3. P0 writes X
4. P1 reads X

For each step, describe the state of address X in both P0's and P1's caches, and identify what bus transactions (if any) are generated. Explain why the protocol must perform each state transition.

**Answer:**

| Step | Operation | P0 State | P1 State | Bus Transaction |
|------|-----------|----------|----------|-----------------|
| Initial | - | Invalid | Invalid | - |
| 1 | P0 reads X | Shared | Invalid | CPU read miss; data fetched from memory |
| 2 | P1 reads X | Shared | Shared | CPU read miss; data fetched from memory (or P0) |
| 3 | P0 writes X | Modified | Invalid | CPU write upgrade; P1 sees remote write miss, invalidates |
| 4 | P1 reads X | Shared | Shared | CPU read miss; P0 must supply data and transition to Shared |

Step 3 requires invalidation because write-invalidate protocols ensure only one writer exists. Step 4 forces P0 to downgrade because P1's read means the line is now shared.

---

## Question 2: MESI vs MSI Trade-offs

A programmer notices that their single-threaded application runs faster on a system using the MESI protocol compared to MSI, even though no data is shared between processors.

**(a) Explain the specific scenario in which MESI provides a performance advantage over MSI for single-threaded code.**

**Answer:** When a processor reads data and later writes to it (load-modify-store pattern), MSI requires a bus transaction for the write because the line is in Shared state and must broadcast an upgrade. In MESI, if no other processor has the data, the line enters Exclusive state on the read. The subsequent write transitions silently to Modified without any bus traffic, saving bandwidth and latency.

**(b) What additional hardware mechanism does MESI require that MSI does not? Why is this mechanism necessary?**

**Answer:** MESI requires either a "shared line" signal on the bus (where all processors indicate whether they have a copy) or a directory/controller that tracks which caches hold copies. This is necessary to distinguish between "no sharing" (→Exclusive) and "sharing" (→Shared) during a read miss.

**(c) Describe a workload pattern where MESI provides no advantage over MSI.**

**Answer:** Read-only shared data accessed by multiple processors. Since other caches always have copies, reads will enter Shared state in both protocols. The Exclusive state is never reached, so MESI's optimization provides no benefit.

---

## Question 3: False Sharing Scenario

Two threads execute concurrently on different processors. Thread A repeatedly increments variable `counter_A` while Thread B repeatedly increments variable `counter_B`. A performance analyst observes severe cache thrashing despite the threads accessing completely independent variables.

**(a) Explain how this situation can occur and what this phenomenon is called.**

**Answer:** This is called **false sharing**. It occurs when `counter_A` and `counter_B` reside on the same cache line. Each write invalidates the entire line in the other processor's cache, even though they're accessing different words. The coherence protocol cannot distinguish word-level accesses—it operates at cache line granularity.

**(b) Describe two different approaches to eliminate this problem—one involving software changes and one involving hardware design choices. Discuss the trade-offs of each approach.**

**Answer:** 
- **Software:** Pad data structures so each variable occupies its own cache line (e.g., add dummy bytes). Trade-off: wastes memory and requires programmer awareness.
- **Hardware:** Use smaller cache lines. Trade-off: reduces spatial locality benefits for sequential access patterns and increases tag overhead.

**(c) If the cache line size were reduced to a single word, would any coherence misses remain? Explain what type of sharing would still exist.**

**Answer:** Only **true sharing** misses would remain—cases where processors actually read and write the same word. False sharing is completely eliminated because each word is its own coherence unit.

---

## Question 4: Miss Classification Ambiguity

Processor P0 has cache line A in the Modified state. Due to capacity pressure, P0 evicts line A (writing it back to memory). Later, processor P1 writes to line A, and subsequently P0 attempts to read line A, experiencing a cache miss.

**(a) Is P0's miss a capacity miss, a coherence miss, or both? Justify your answer.**

**Answer:** It is **both**. The line was evicted due to capacity limits (capacity miss), but even if P0 had infinite cache, P1's write would have invalidated P0's copy anyway (coherence miss). Neither condition alone is sufficient to explain the miss.

**(b) Would increasing the cache size alone eliminate this miss? Would using a different coherence protocol eliminate this miss? Explain your reasoning.**

**Answer:** Increasing cache size would prevent the capacity eviction, but P1's write would still invalidate P0's copy—miss remains. Changing protocols wouldn't help either; all invalidation-based protocols must invalidate on remote writes to maintain coherence. The miss persists because fixing one cause doesn't fix the other.

**(c) Propose a methodology for how a performance analyst should handle such ambiguous cases when optimizing a parallel application.**

**Answer:** Analyze both causes independently: simulate with infinite cache to isolate coherence misses, and simulate single-threaded to isolate capacity misses. If a miss appears in both scenarios, it's the intersection. Optimization should address whichever cause dominates or is easier to fix for the specific workload.

---

## Question 5: Data Access Pattern Identification

For each scenario below, identify the most appropriate data access pattern classification and describe what cache behavior you would expect under MESI:

**(a) A global configuration structure that is initialized once at program startup and never modified thereafter, accessed by all threads.**

**Answer:** **Read-only shared.** After initialization, all caches hold the line in Shared state. No coherence misses occur during execution since no invalidations happen. Initial cold misses only.

**(b) A task queue where a master thread inserts work items and worker threads remove them.**

**Answer:** **Producer-consumer.** The queue data migrates between Modified (producer writing) and Invalid/Shared (consumers reading). Frequent coherence traffic as ownership transfers. High coherence miss rate on the queue metadata.

**(c) A lock variable used to protect a critical section, acquired and released by different threads in sequence.**

**Answer:** **Migratory.** The lock moves between processors, each taking exclusive ownership (Modified) temporarily. Each acquisition causes a coherence miss to invalidate the previous holder. Classic ping-pong pattern.

**(d) A thread-local buffer that only its owning thread ever accesses.**

**Answer:** **Private.** The line enters Exclusive on first read (no sharers), transitions to Modified on write, and stays there. No coherence traffic after the initial cold miss. Ideal cache behavior.

---

## Question 6: Protocol Design Reasoning

**(a) Why doesn't the Exclusive state simply transition to Shared when a remote read occurs, similar to how Modified transitions to Shared?**

**Answer:** It actually does—when another processor reads, Exclusive must transition to Shared (or Invalid, depending on implementation) because the "exclusive" property (only copy) is no longer true. The key difference from Modified is that Exclusive data is clean, so no writeback is needed—just a state change.

**(b) A designer proposes adding a fifth state "Owned" (O) that indicates the cache has a modified copy but other caches may have Shared copies. What problem does this MOESI extension solve, and in what scenario would it provide benefit?**

**Answer:** MOESI solves the problem of unnecessary memory writebacks. When a Modified line is read by another processor, MESI requires either a writeback to memory or a cache-to-cache transfer with the original going to Shared (clean). With Owned, the original cache keeps responsibility for the dirty data while sharing it—avoiding the memory writeback. Benefits scenarios with frequent read-sharing of recently-written data.

---

## Question 7: Coherence Protocol Behavior Trace

Trace the state of Y in each cache through this sequence under MESI:

1. P0 reads Y (no other cache has Y)
2. P1 reads Y
3. P2 reads Y
4. P1 writes Y
5. P0 reads Y

**Answer:**

| Step | Operation | P0 | P1 | P2 | Bus Traffic | Data Source |
|------|-----------|----|----|----|----|-------------|
| 1 | P0 reads Y | **Exclusive** | I | I | Read miss | Memory |
| 2 | P1 reads Y | Shared | **Shared** | I | Read miss; P0 sees snoop | P0 or Memory |
| 3 | P2 reads Y | Shared | Shared | **Shared** | Read miss | Memory or cache |
| 4 | P1 writes Y | **Invalid** | **Modified** | **Invalid** | Write upgrade/invalidate | - (has data) |
| 5 | P0 reads Y | **Shared** | **Shared** | I | Read miss | P1 (supplies data, downgrades) |

Key observations: Step 1 achieves Exclusive because no sharers exist. Steps 2-3 force Shared state. Step 4 invalidates all other copies. Step 5 is a coherence miss caused by P1's write.


