
# Question 1 (Class 12: Oct 15)

Which cache uses larger set associativity? L1 or L2?

### Short answer

Typically, **L2 caches use *larger* set associativity than L1 caches**.

More generally (using your terminology):

* “Higher level” cache (closest to CPU, e.g., L1) → **lower associativity**
* “Lower level” cache (farther from CPU, e.g., L2, L3) → **higher associativity**

---

### Why associativity changes across levels

Think about what each level is optimizing for.

#### 1. L1 cache: speed is everything

L1 is on the critical path of almost every load/store. Every extra gate here shows up directly as extra CPU cycle time.

* Higher associativity means:

  * More tag comparators in parallel
  * Larger multiplexers to pick the right way
  * More wires and longer critical paths
* That increases:

  * **Access latency**
  * **Energy per access**
  * **Design complexity / frequency constraints**

So for L1, designers accept a bit **higher miss rate** in exchange for **very low hit time**.
Result: L1 is small and only modestly associative (e.g., 2–8 way).

#### 2. L2 (and below): capacity & miss rate matter more

L2 is already slower than L1, so adding a **little more latency** to its hit time is less painful. But:

* L2 is much larger.
* It sees a much larger and more diverse working set.
* It is the last line of defense before going to main memory, which is extremely slow.

Large caches are more prone to **conflict misses** if they are low-associative, because many different addresses can map to the same set.

Increasing associativity in L2:

* Reduces **conflict misses** significantly.
* Improves **overall miss rate** of the whole hierarchy.
* The added hit latency is acceptable because L2 hits are much less frequent than L1 hits and are off the core’s most timing-critical path.

So for lower-level caches (farther from the CPU), the design trade-off shifts:

> Willing to pay more in hit time and energy per access in order to reduce miss rate.

Result: L2/L3 often have **higher associativity** than L1 (e.g., 8–16 way).

---

### Intuition summary

* **Closer to CPU (L1):**

  * Primary goal: **minimum hit latency**
  * Constraint: very tight timing → **lower associativity**

* **Farther from CPU (L2, L3):**

  * Primary goal: **lower miss rate / better effective memory latency**
  * Can tolerate a bit more hit latency → **higher associativity**

That is why “higher level” (closest) caches usually have lower associativity, while “lower level” (farther) caches have higher associativity.


## Question 2 (Class 13: Octt 15)

Consider two memory addresses, 245 and 295 (in decimal). These are distinct addresses that share the same hundreds digit but differ in their tens digit. Suppose we want to map memory addresses to cache lines. One proposal is to use the hundreds digit of the address to determine the cache line; another proposal is to use the tens digit.

Which digit (the hundreds or the tens digit) should be used to determine the cache-line mapping, and why?

---

**Formal Answer**

We should use the **tens digit** (a lower-order digit) rather than the **hundreds digit** (a higher-order digit) to determine the cache-line mapping.

In cache design, the index into the cache (or cache set) is normally derived from the **low-order bits** of the address (just above the block/word offset). The reasons are:

1. **Better distribution of addresses across cache lines.**
   Lower-order digits (or bits) change more frequently between nearby addresses. Using the tens digit means that addresses like 245 and 295, which are distinct and may be accessed independently, can map to different cache lines. If we used the hundreds digit, both 245 and 295 would map to the same cache line, increasing the likelihood of conflict misses.

2. **Alignment with spatial locality and block structure.**
   Programs frequently access contiguous or nearby memory addresses. Using lower-order address bits (or digits) to index the cache ensures that contiguous memory regions are spread more evenly across the cache, which better exploits spatial locality and reduces systematic conflicts.

3. **Hardware practice.**
   Real cache implementations index by low-order address bits for exactly these reasons: to avoid pathological aliasing of many distinct addresses into the same few cache lines.

Therefore, using the **tens digit** is the more appropriate choice for determining cache-line mapping in this setting.

### Question 3 (Class 13: Oct 15)

In a direct-mapped cache there is no need for a replacement policy, since each memory block has exactly one possible cache line it can occupy.
So why do we still need tag bits in a direct-mapped cache?

---

### Answer

Even in a direct-mapped cache, tag bits are necessary to distinguish **which** memory block is currently stored in a given cache line.

Key points:

1. **Many memory blocks share the same index**
   In a direct-mapped cache, the address is typically divided as:

   * **Tag** | **Index** | **Block offset**
     The index selects a single cache line, but many different memory blocks can map to that same index (they just have different tag bits).

2. **The index alone is not enough**
   When you access an address:

   * You use the **index** bits to choose a cache line.
   * You then compare the **tag** bits of the address with the **stored tag** in that line.
   * If the tags match (and the line is valid), it is a **hit**; otherwise, it is a **miss**.

   Without tag bits, you could not tell whether the data currently in that cache line corresponds to:

   * The block you are requesting now, or
   * Some other block that previously mapped to the same index.

3. **Replacement policy vs. tags**

   * A **replacement policy** is about deciding *which* line to evict when there are multiple choices (e.g., in set-associative or fully associative caches). In a direct-mapped cache, there is only *one* possible line per index, so no policy is needed.
   * **Tags** are about **correctness**, not choice: they ensure the cache can detect whether the data in the selected line actually belongs to the requested address.

So, even though a direct-mapped cache does not need a replacement policy, it still needs tags to correctly detect hits and misses.

# Question 4 (Class 14, Oct 20)

**Among compulsory misses, capacity misses, and conflict misses, which ones are affected if cache block size is increased?**

---

# Answer

When cache block size is increased, the following cache misses are affected:

## 1. Compulsory Misses (Cold Start Misses) - **DECREASE**

Compulsory misses are **reduced** when block size increases. This is because larger blocks exploit spatial locality more effectively—when a block is fetched, more neighboring data comes with it. Since programs typically access data with spatial locality, fetching larger blocks means fewer initial misses are needed to bring in the data that will be accessed.

## 2. Capacity Misses - **MAY INCREASE**

Capacity misses can **increase** with larger block sizes. When block size increases while total cache size remains constant, the total number of blocks the cache can hold decreases. This means the cache can store fewer distinct memory regions, potentially causing more data to be evicted and leading to more capacity misses.

For example:
- A 64 KB cache with 64-byte blocks can hold 1,024 blocks
- The same 64 KB cache with 128-byte blocks can only hold 512 blocks

## 3. Conflict Misses - **MAY INCREASE**

Conflict misses can **increase** with larger block sizes, particularly in set-associative or direct-mapped caches. With fewer total blocks in the cache, there are fewer sets available, which increases the likelihood that multiple memory addresses will map to the same set and compete for the limited positions, causing more conflicts.

---

## Summary

- **Compulsory misses**: Decrease (better spatial locality exploitation)
- **Capacity misses**: Increase (fewer blocks fit in cache)
- **Conflict misses**: Increase (fewer sets available)

The optimal block size represents a trade-off between these competing effects.

# Question 5

In software prefetching, why do modern processors implement dedicated prefetch instructions (like `PREFETCH` on x86 or `PLD` on ARM) rather than simply using regular load instructions to bring data into cache?

## Answer

### The Core Problem: Register Binding

When you use a regular load instruction for prefetching, **you must bind the loaded data to an architectural register**. This creates several significant problems:

#### 1. **Register Pressure**
- Registers are a scarce resource in most ISAs (16-32 general-purpose registers typically)
- If you perform a load just for prefetching, you consume a register that could be used for actual computation
- In tight loops or complex code, available registers may already be exhausted
- The compiler must choose between using registers for prefetch loads versus actual program data

#### 2. **Register Lifetime Management**
- Once you load data into a register for prefetching, you must maintain that register until the actual use
- This extends register live ranges unnecessarily
- The compiler/programmer must track which "prefetch registers" hold what data
- If the prefetch is far ahead of actual use, you tie up registers for extended periods

#### 3. **False Dependencies**
- Loading into a register creates a data dependency in the instruction stream
- Subsequent instructions that use that register must wait for the load to complete
- This defeats the purpose of prefetching if you're trying to hide latency
- Creates artificial serialization points in the program

### Advantages of Non-Binding Prefetch Instructions

Dedicated prefetch instructions solve these problems by being **non-binding**:

#### **No Register Allocation**
- Prefetch instructions specify only a memory address, not a destination register
- No register resources are consumed
- Frees the compiler/programmer from managing prefetch-related register allocation

#### **Truly Non-Blocking**
- The processor can issue the prefetch and immediately continue
- No artificial dependencies are created in the instruction stream
- Enables aggressive out-of-order execution

#### **Hint-Based Semantics**
- Prefetch instructions are architectural *hints*, not mandatory operations
- The processor can ignore them if resources are constrained
- No architectural state changes, so no correctness implications if skipped
- Can be safely speculated without rollback concerns

### Problems in Multithreaded Contexts

The register-binding problem becomes even more severe in multithreaded scenarios:

#### **Context Switch Overhead**
- If loads are used for prefetching, the loaded values occupy registers
- During context switches, all register state must be saved/restored
- This increases context switch latency unnecessarily for data that's only needed for prefetching
- Prefetch instructions don't contribute to architectural state, so nothing extra to save

#### **Thread-Local Register Constraints**
- Each thread has its own register set (or shares a limited pool)
- Using loads for prefetching exacerbates per-thread register pressure
- In SMT (Simultaneous Multithreading), threads already compete for execution resources
- Wasting registers on prefetch loads degrades performance for all threads

#### **Speculation and Thread Synchronization**
- With regular loads, incorrect speculation requires architectural rollback
- The loaded register values must be properly managed during rollback
- Prefetch instructions can be freely speculated without rollback concerns
- This is particularly valuable near synchronization points where speculation is common

#### **Cache Pollution Coordination**
- Multiple threads using load-based prefetching may inadvertently evict each other's data
- Prefetch instructions can be given lower priority in cache replacement policies
- Some implementations allow prefetches to allocate in different cache levels or with different coherence states

### Additional Benefits

**Semantic Clarity**: The intent is explicit—this is a performance hint, not a correctness requirement

**Flexibility**: Processors can implement prefetch instructions with varying degrees of sophistication (different cache levels, prefetch distances, etc.) without changing the ISA

**Power Efficiency**: The processor can choose to ignore prefetches when power-constrained without affecting correctness

## Conclusion 

Dedicated prefetch instructions separate the **performance optimization** concern (bringing data closer) from the **correctness** concern (architectural state management). This separation is essential for effective software prefetching, particularly in resource-constrained environments like multithreaded processors where register pressure and context switch overhead are critical concerns.

# Question 6

**In certain benchmarks, non-blocking caches show no performance improvement over blocking caches. Why might this be the case?**

## Answer

Non-blocking caches are designed to improve performance by allowing the processor to continue executing independent instructions while a cache miss is being serviced. However, this benefit disappears when the workload exhibits **high memory-level dependence**—specifically in cases involving extensive pointer chasing.

**Pointer chasing** occurs in workloads that traverse linked data structures such as linked lists, trees, or graphs. In these scenarios, each memory access depends directly on the value returned by the previous access. For example, to find the next node in a linked list, the processor must first retrieve the current node's pointer field.

When a cache miss occurs in such workloads:

1. The processor cannot determine the next memory address until the current data arrives
2. There are no independent memory operations that can be overlapped with the outstanding miss
3. The processor must stall regardless of whether the cache is blocking or non-blocking

In essence, non-blocking caches provide benefit only when there is **memory-level parallelism (MLP)**—multiple independent memory accesses that can be serviced concurrently. When the workload is serialized by data dependencies, the theoretical advantage of allowing continued execution becomes moot, since there is simply no useful work the processor can perform while waiting for the miss to resolve.

# Question 7

**When examining a graph of L2 local miss rate versus L2 cache size, we often observe no improvement in miss rate at smaller cache sizes. Why does this flat region occur?**

## Answer

The L2 cache serves as a **victim cache** for the L1—it captures data that was evicted from the L1 but may still be part of the application's working set. For the L2 to provide any benefit, it must be large enough to hold the portion of the working set that exceeds the L1's capacity.

At smaller L2 sizes, the cache cannot accommodate this overflow. Consider a workload with a 256 KB working set and a 32 KB L1 cache. The remaining 224 KB must fit in the L2 for the application to experience improved hit rates. If the L2 is only 64 KB or 128 KB, it will suffer from severe capacity misses just as the L1 did—data will be evicted before it can be reused.

This explains the **flat region** at the beginning of the graph:

1. The L2 is too small to capture temporal locality from the spillover working set
2. Nearly every access that missed in L1 also misses in L2
3. The local miss rate remains high regardless of small increases in L2 size

Only once the L2 reaches a **threshold size**—large enough to contain the working set delta between what L1 can hold and what the application needs—do we begin to see the local miss rate decrease. Beyond this point, further increases in cache size yield diminishing returns until the entire working set fits, at which point the miss rate drops to near zero (limited only by compulsory misses).