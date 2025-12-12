# Detailed Solutions

---

## Question

**Given:**
- Clock frequency: 2 GHz
- Base CPI (non-memory): 1.0
- Load/store fraction: 30%
- L1: 1-cycle hit time, 5% miss rate
- L2: 10-cycle hit time, 20% local miss rate
- Main memory: 200 cycles

---

### Part (a): Calculate the effective CPI

**Step 1: Calculate AMAT for the two-level hierarchy**

For a multi-level cache:
$$\text{AMAT} = \text{Hit time}_{L1} + \text{MR}_{L1} \times (\text{Hit time}_{L2} + \text{MR}_{L2} \times \text{MP}_{mem})$$

$$\text{AMAT} = 1 + 0.05 \times (10 + 0.20 \times 200)$$

$$\text{AMAT} = 1 + 0.05 \times (10 + 40)$$

$$\text{AMAT} = 1 + 0.05 \times 50 = 1 + 2.5 = \boxed{3.5 \text{ cycles}}$$

**Step 2: Calculate overall CPI**

$$\text{CPI} = \text{CPI}_{others} \times \frac{\text{IC}_{others}}{\text{IC}} + \text{CPI}_{ld/st} \times \frac{\text{IC}_{ld/st}}{\text{IC}}$$

$$\text{CPI} = 1.0 \times 0.70 + 3.5 \times 0.30$$

$$\text{CPI} = 0.70 + 1.05 = \boxed{1.75}$$

---

### Part (b): Speedup if L1 miss rate reduced to 3%

**New AMAT:**
$$\text{AMAT}_{new} = 1 + 0.03 \times (10 + 40) = 1 + 1.5 = 2.5 \text{ cycles}$$

**New CPI:**
$$\text{CPI}_{new} = 1.0 \times 0.70 + 2.5 \times 0.30 = 0.70 + 0.75 = 1.45$$

**Speedup:**
$$\text{Speedup} = \frac{\text{CPI}_{old}}{\text{CPI}_{new}} = \frac{1.75}{1.45} = \boxed{1.207 \text{ (20.7\% improvement)}}$$

---

### Part (c): Equivalent miss rate with 150-cycle memory

We want to find what L1 miss rate with 150-cycle memory gives CPI = 1.45.

**Set up equation:**
$$\text{AMAT} = 1 + \text{MR}_{L1} \times (10 + 0.20 \times 150) = 1 + \text{MR}_{L1} \times 40$$

$$\text{CPI} = 0.70 + \text{AMAT} \times 0.30 = 1.45$$

**Solve:**
$$\text{AMAT} \times 0.30 = 0.75$$
$$\text{AMAT} = 2.5 \text{ cycles}$$

$$1 + \text{MR}_{L1} \times 40 = 2.5$$
$$\text{MR}_{L1} = \frac{1.5}{40} = \boxed{3.75\%}$$

**Interpretation:** With faster memory (150 cycles), you only need to reduce miss rate to 3.75% to achieve the same performance as reducing it to 3% with 200-cycle memory.

---

## Question 5 Solution

**Given:**
- Design A: Hit time = 2 cycles, Miss rate = 8%, Miss penalty = 100 cycles
- Design B: Hit time = 4 cycles, Miss rate = 4%, Miss penalty = 100 cycles

---

### Part (a): Which design has better AMAT?

$$\text{AMAT}_A = 2 + 0.08 \times 100 = 2 + 8 = \boxed{10 \text{ cycles}}$$

$$\text{AMAT}_B = 4 + 0.04 \times 100 = 4 + 4 = \boxed{8 \text{ cycles}}$$

**Design B has better AMAT** (despite higher hit time, the lower miss rate more than compensates).

---

### Part (b): Miss penalty for equal AMAT

Set $\text{AMAT}_A = \text{AMAT}_B$:

$$2 + 0.08 \times \text{MP} = 4 + 0.04 \times \text{MP}$$

$$0.08 \times \text{MP} - 0.04 \times \text{MP} = 4 - 2$$

$$0.04 \times \text{MP} = 2$$

$$\text{MP} = \boxed{50 \text{ cycles}}$$

**Verification:**
- $\text{AMAT}_A = 2 + 0.08 \times 50 = 6$ cycles
- $\text{AMAT}_B = 4 + 0.04 \times 50 = 6$ cycles ✓

**Insight:** For miss penalties < 50 cycles, Design A (faster hit time) wins. For miss penalties > 50 cycles, Design B (lower miss rate) wins.

---

### Part (c): Overall CPI calculation

Given: 40% memory instructions, $\text{CPI}_{others} = 1.2$

$$\text{CPI} = \text{CPI}_{others} \times (1 - f_{mem}) + \text{AMAT} \times f_{mem}$$

**Design A:**
$$\text{CPI}_A = 1.2 \times 0.60 + 10 \times 0.40 = 0.72 + 4.0 = \boxed{4.72}$$

**Design B:**
$$\text{CPI}_B = 1.2 \times 0.60 + 8 \times 0.40 = 0.72 + 3.2 = \boxed{3.92}$$

**Speedup of B over A:** $\frac{4.72}{3.92} = 1.20$ (20% faster)

# Cache Performance Analysis

## Problem Statement

Consider a computer system with the following characteristics:

- **CPI (Cycles Per Instruction):** 1 when all memory accesses hit in the cache
- **Data access instructions (load/store):** 50% of all instructions
- **Cache miss penalty:** 25 clock cycles
- **Cache miss rate:** 2%

**Question:** How much faster would the computer be if all instructions were cache hits?

---

## Solution

### Step 1: Compute Performance with Perfect Cache (Always Hits)

$$
\begin{aligned}
\text{CPU execution time} &= (\text{CPU clock cycles} + \text{Memory stall cycles}) \times \text{Clock cycle} \\
&= (\text{IC} \times \text{CPI} + 0) \times \text{Clock cycle} \\
&= \text{IC} \times 1.0 \times \text{Clock cycle}
\end{aligned}
$$

### Step 2: Compute Memory Stall Cycles for Real Cache

$$
\begin{aligned}
\text{Memory stall cycles} &= \text{IC} \times \frac{\text{Memory accesses}}{\text{Instruction}} \times \text{Miss rate} \times \text{Miss penalty} \\
&= \text{IC} \times (1 + 0.5) \times 0.02 \times 25 \\
&= \text{IC} \times 0.75
\end{aligned}
$$

> **Note:** The term $(1 + 0.5)$ represents **one instruction fetch** plus **0.5 data accesses** per instruction.

### Step 3: Compute Total Performance with Real Cache

$$
\begin{aligned}
\text{CPU execution time}_{\text{cache}} &= (\text{IC} \times 1.0 + \text{IC} \times 0.75) \times \text{Clock cycle} \\
&= 1.75 \times \text{IC} \times \text{Clock cycle}
\end{aligned}
$$

### Step 4: Calculate Performance Ratio

The performance ratio is the inverse of the execution times:

$$
\frac{\text{CPU execution time}_{\text{cache}}}{\text{CPU execution time}} = \frac{1.75 \times \text{IC} \times \text{Clock cycle}}{1.0 \times \text{IC} \times \text{Clock cycle}} = 1.75
$$

---

## Answer

$$\boxed{\text{The computer with no cache misses is } \mathbf{1.75} \text{ times faster.}}$$

# Answers to Cache Miss Classification Questions

---

## Question 3

**If you increase cache size while keeping associativity constant, which type(s) of misses would you expect to decrease?**

### Answer:

**Capacity misses will decrease** because a larger cache can hold more of the working set, reducing evictions caused by insufficient total space.

**Conflict misses will also decrease** because increasing cache size while maintaining the same associativity means more sets are available. With more sets, blocks are distributed across a larger number of sets, reducing contention within each individual set.

**Compulsory misses remain unchanged** because they depend only on whether a block has been accessed before—not on cache size. The first access to any block will always be a compulsory miss regardless of how large the cache is.

---

## Question 4

**A program repeatedly accesses memory addresses that map to the same cache set in a 4-way set-associative cache. If 6 unique blocks are accessed in a round-robin pattern, classify the misses after warm-up. Would changing to 8-way set-associative change your classification?**

### Answer:

**With 4-way set-associative:**

Let the blocks be A, B, C, D, E, F, all mapping to the same set.

*Warm-up phase:*
| Access | Result | Cache State (LRU order) |
|--------|--------|------------------------|
| A | Compulsory miss | {A} |
| B | Compulsory miss | {A, B} |
| C | Compulsory miss | {A, B, C} |
| D | Compulsory miss | {A, B, C, D} |
| E | Compulsory miss, evict A | {B, C, D, E} |
| F | Compulsory miss, evict B | {C, D, E, F} |

*After warm-up (steady state):*

Every subsequent access to A, B, C, D, E, F will miss. These are **conflict misses** because:
- The evictions occur because the *set* is full (only 4 ways)
- The overall cache may have plenty of empty space elsewhere
- The blocks compete for limited positions within their assigned set

**With 8-way set-associative:**

Now the set can hold all 6 blocks. After the 6 compulsory misses during warm-up, **all subsequent accesses hit**. The conflict misses are completely eliminated because the associativity now exceeds the working set size for that set.

---

## Question 8

**Is it possible for a memory access pattern to experience capacity misses in a fully associative cache but conflict misses in a larger direct-mapped cache?**

### Answer:

**Yes, this is possible.** Here's a concrete example:

**Setup:**
- Fully associative cache: 2 blocks
- Direct-mapped cache: 4 blocks (indices 0, 1, 2, 3)

**Access pattern:** Blocks X, Y, Z accessed repeatedly where:
- X maps to index 0
- Y maps to index 0
- Z maps to index 1

**In the fully associative cache (2 blocks):**

| Access | Result | Cache State |
|--------|--------|-------------|
| X | Compulsory miss | {X} |
| Y | Compulsory miss | {X, Y} |
| Z | Compulsory miss, evict X | {Y, Z} |
| X | **Capacity miss**, evict Y | {Z, X} |
| Y | **Capacity miss**, evict Z | {X, Y} |
| Z | **Capacity miss**, evict X | {Y, Z} |

The working set (3 blocks) exceeds cache capacity (2 blocks), causing capacity misses.

**In the direct-mapped cache (4 blocks):**

| Access | Result | Index 0 | Index 1 |
|--------|--------|---------|---------|
| X | Compulsory miss | X | — |
| Y | Compulsory miss (evicts X) | Y | — |
| Z | Compulsory miss | Y | Z |
| X | **Conflict miss** (evicts Y) | X | Z |
| Y | **Conflict miss** (evicts X) | Y | Z |
| Z | Hit | Y | Z |

Despite having more total capacity, X and Y conflict at index 0, causing conflict misses. Meanwhile, Z always hits after its first access.

**Key insight:** A smaller fully associative cache can have capacity misses while a larger direct-mapped cache has conflict misses for the same access pattern, because the direct-mapped cache's placement restrictions create conflicts even when total space is available.

# Answers to Selected Exam Questions

---

## Question 1

**Explain why conflict misses are virtually eliminated in the 4-way set associative cache but capacity misses remain largely unchanged. What does this reveal about the fundamental nature of these two miss types?**

### Graph Configuration and Characteristics

The figure presents two stacked area charts comparing miss rates across cache sizes ranging from 4KB to 512KB. The left chart represents a **direct-mapped cache** (1-way associativity), while the right chart represents a **4-way set associative cache**. Both charts decompose total miss rate (y-axis, ranging from 0 to 0.1 or 10%) into three components shown as stacked colored regions:

- **Cold/Compulsory misses** (bottom layer): A thin, nearly constant band
- **Capacity misses** (middle layer): A substantial region that decreases as cache size increases
- **Conflict misses** (top layer): Present significantly in the direct-mapped cache but nearly absent in the 4-way set associative cache

### Answer

**Why Conflict Misses Are Eliminated with Set Associativity:**

Conflict misses occur when multiple memory blocks compete for the same cache location, even though the overall cache has free space. In a direct-mapped cache, each memory address maps to exactly one cache line. If a program accesses addresses A and B that both map to line 7, they will continuously evict each other—even if lines 0–6 and 8–127 sit completely empty.

In a 4-way set associative cache, each memory address can reside in any of four lines within its designated set. This dramatically reduces the probability of conflicts because:

1. Four blocks can coexist in the same set before eviction becomes necessary
2. The LRU (or similar) replacement policy keeps the most useful blocks
3. Only when a fifth block maps to the same set does a true conflict occur

Since most programs exhibit working sets where fewer than four frequently-accessed blocks map to any given set, the 4-way design effectively absorbs the conflicts that plagued the direct-mapped design.

**Why Capacity Misses Remain Unchanged:**

Capacity misses occur when the total working set exceeds the cache size, regardless of organization. These misses are a function of:

- The program's working set size
- The total cache capacity (in bytes)

Changing associativity reorganizes *how* data is placed within the cache but does not change *how much* data the cache can hold. A 64KB direct-mapped cache and a 64KB 4-way set associative cache store exactly the same amount of data. Therefore, if a program needs to access 100KB of data repeatedly, both designs will experience capacity misses when older data must be evicted to make room.

**Fundamental Insight:**

This reveals a critical distinction in miss classification:

| Miss Type | Caused By | Solved By |
|-----------|-----------|-----------|
| **Conflict** | Cache organization/mapping policy | Increasing associativity |
| **Capacity** | Insufficient total cache size | Increasing cache size |

Conflict misses are an artifact of the mapping function—a "self-inflicted wound" of simpler cache designs. Capacity misses are fundamental limitations based on physical storage constraints that no organizational scheme can overcome.

---

## Question 2

**The "cold" (compulsory) miss component appears constant across all cache sizes in both graphs. Why is this the case, and under what circumstances could compulsory misses actually increase with cache size?**

### Answer

**Why Compulsory Misses Remain Constant:**

Compulsory misses (also called cold-start or first-reference misses) occur when a block is accessed for the very first time. By definition, this block cannot already be in the cache because it has never been fetched before.

The number of compulsory misses depends on:

1. **The number of unique blocks accessed** by the program
2. **The block size** (which determines how many bytes are fetched per miss)

Neither of these factors changes with cache size or associativity:

- If a program touches 10,000 unique cache blocks during execution, it will experience exactly 10,000 compulsory misses whether the cache is 4KB or 4MB
- The cache size only affects whether blocks *remain* in the cache after being fetched, not whether they need to be fetched initially
- Associativity affects placement and replacement, but a block that has never been seen must still be fetched regardless of where it will be placed

**Mathematical Perspective:**

$$\text{Compulsory Miss Rate} = \frac{\text{Number of Unique Blocks Accessed}}{\text{Total Memory Accesses}}$$

This ratio is determined entirely by program behavior and block size—cache parameters do not appear in this equation.

**Circumstances Where Compulsory Misses Could INCREASE with Cache Size:**

This seems counterintuitive, but there are several scenarios:

1. **Increased Block Size Accompanying Larger Caches:**
   
   Larger caches often use larger block sizes to exploit spatial locality efficiently. If block size increases:
   - Fewer blocks fit in the same capacity
   - But more importantly, each miss now fetches more data
   - If the program has poor spatial locality, larger blocks fetch unused bytes
   - The *number* of compulsory misses stays similar, but if we measure miss rate per byte accessed, larger blocks can increase compulsory miss *impact*

2. **Prefetching Side Effects:**
   
   Larger caches sometimes enable more aggressive prefetching. If the prefetcher speculatively fetches blocks that are never actually used:
   - These fetches technically count as compulsory misses (first access to those blocks)
   - The useful compulsory misses stay constant
   - But useless prefetched blocks add to the cold miss count

3. **Cache Pollution Leading to Working Set Expansion:**
   
   With a larger cache, programmers or compilers might use algorithms with larger working sets, assuming the cache can handle it. This increases the number of unique blocks accessed, thereby increasing compulsory misses.

4. **Measurement Artifacts with Fixed Trace Length:**
   
   If measuring miss rate over a fixed number of accesses (rather than complete program execution):
   - A small cache might evict and re-access blocks (counting as capacity/conflict misses)
   - A large cache retains everything, so each unique block is only a compulsory miss
   - The *total miss count* drops, but the *proportion* that are compulsory increases
   - In absolute terms, compulsory misses stay constant, but their relative prominence grows

---

## Question 6

**Describe a workload pattern where:**
- **(a) A direct-mapped cache would perform nearly as well as a 4-way set associative cache**
- **(b) Even a 4-way set associative cache would show significant conflict misses**

### Answer

### Part (a): Workload Where Direct-Mapped ≈ 4-Way Performance

**Characteristics of Such a Workload:**

A direct-mapped cache performs nearly as well as a set-associative cache when **conflict misses are naturally rare**. This occurs when memory access patterns don't create mapping collisions.

**Example Workload: Sequential Array Processing**

```c
// Processing a large array sequentially
float array[1000000];
float sum = 0;
for (int i = 0; i < 1000000; i++) {
    sum += array[i];
}
```

**Why Conflicts Are Minimal:**

1. **Sequential access pattern**: Consecutive memory addresses map to consecutive cache lines
2. **No revisitation before eviction**: By the time the program wraps around (if ever), all data has been processed
3. **Natural spread across cache sets**: Sequential addresses distribute evenly across all sets rather than clustering in a few sets
4. **High spatial locality**: Each cache block fetched is fully utilized before moving to the next

**Additional Conflict-Free Patterns:**

- **Small working sets**: If the entire working set fits in cache, there's no competition for lines
- **Compiler-optimized loops**: Loop tiling/blocking ensures active data maps to different sets
- **Stack-heavy computation**: Local variables on the stack typically occupy contiguous memory with good cache distribution

---

### Part (b): Workload Where 4-Way Still Has Significant Conflicts

**Characteristics of Such a Workload:**

A 4-way set associative cache experiences conflict misses when **more than four frequently-accessed blocks map to the same set**. This requires specific pathological access patterns.

**Example Workload: Strided Access with Unfortunate Alignment**

```c
// Assume 64KB cache, 64-byte blocks, 4-way set associative
// Cache has 256 sets (64KB / 64 bytes / 4 ways)
// Stride of 16KB = 256 blocks = exactly the number of sets

int matrix[8][4096];  // 8 rows, each 16KB apart in memory

// Column-major access pattern hitting same set repeatedly
for (int col = 0; col < 4096; col++) {
    for (int row = 0; row < 8; row++) {
        process(matrix[row][col]);
    }
}
```

**Why Conflicts Occur:**

1. **Stride equals (sets × block_size)**: Each row starts at an address that maps to the same cache set
2. **Eight rows compete for four ways**: With only 4 ways available, accessing 8 rows means 4 must be evicted
3. **Cyclic reuse**: The inner loop cycles through all 8 rows repeatedly, causing continuous evictions
4. **Conflict despite available capacity**: The cache could easily hold 8 × 64 = 512 bytes, but the mapping function forces all 8 blocks into one 4-entry set

**Calculating the Conflict:**

| Access | Row | Maps to Set | Cache State (4 ways) | Result |
|--------|-----|-------------|----------------------|--------|
| 1 | 0 | Set 0 | [R0, -, -, -] | Miss (compulsory) |
| 2 | 1 | Set 0 | [R0, R1, -, -] | Miss (compulsory) |
| 3 | 2 | Set 0 | [R0, R1, R2, -] | Miss (compulsory) |
| 4 | 3 | Set 0 | [R0, R1, R2, R3] | Miss (compulsory) |
| 5 | 4 | Set 0 | [R4, R1, R2, R3] | Miss (compulsory) |
| 6 | 5 | Set 0 | [R4, R5, R2, R3] | Miss (compulsory) |
| 7 | 6 | Set 0 | [R4, R5, R6, R3] | Miss (compulsory) |
| 8 | 7 | Set 0 | [R4, R5, R6, R7] | Miss (compulsory) |
| 9 | 0 | Set 0 | [R0, R5, R6, R7] | **Miss (conflict)** |
| 10 | 1 | Set 0 | [R0, R1, R6, R7] | **Miss (conflict)** |
| ... | ... | ... | ... | All subsequent = conflict |

After the first pass, **every single access is a conflict miss** because the 4-way cache cannot hold the 8-block working set that maps to one set.

**Other Pathological Patterns:**

- **Power-of-2 strided access** in scientific computing (FFT, matrix transpose)
- **Hash table collisions** where hash function aligns with cache set indexing
- **Memory allocator alignment** that places related objects at conflicting addresses
- **Multi-threaded false sharing** where different threads access data in the same set

**Mitigation Strategies:**

- Increase associativity to 8-way or higher
- Use **cache-aware algorithms** that pad arrays to avoid stride conflicts
- Employ **set index randomization** (some modern processors)
- Apply **loop transformations** (interchange, tiling) to change access patterns

---

## Question

Explain the paradox inherent in using larger cache block sizes: How can a technique that reduces one type of miss rate simultaneously increase the overall miss rate under certain workloads? Provide a concrete example of a memory access pattern where doubling the block size would degrade performance.

**Answer**

Larger block sizes reduce **compulsory (cold) misses** by exploiting spatial locality—when one word is accessed, neighboring words are brought into the cache and are likely to be used soon. However, this same technique can *increase* **capacity misses** and **conflict misses**, potentially causing a net performance loss.

### Why This Happens

**Capacity Miss Increase:**
For a fixed cache size, larger blocks mean fewer total blocks in the cache. If the working set requires more blocks than the cache can hold, evictions occur more frequently. The cache effectively "wastes" space on data that may never be used (the unused portions of large blocks).

**Conflict Miss Increase:**
In set-associative or direct-mapped caches, fewer blocks means fewer sets (or fewer blocks per set mapping). This increases the probability that two frequently-accessed addresses map to the same location, causing mutual eviction even when the cache isn't full.

### Concrete Example

Consider a direct-mapped cache of 1 KB with initial block size of 64 bytes (16 blocks), and a program that accesses these addresses in a loop:

```
Address pattern: 0x0000, 0x0400, 0x0000, 0x0400, ... (alternating)
```

These addresses are 1024 bytes apart.

**With 64-byte blocks:**
- Address 0x0000 → Block index = (0x0000 / 64) mod 16 = 0
- Address 0x0400 → Block index = (0x0400 / 64) mod 16 = (16) mod 16 = 0

Both map to block 0 → **conflict misses on every access** (thrashing).

**Now double to 128-byte blocks (8 blocks):**
- Address 0x0000 → Block index = (0x0000 / 128) mod 8 = 0
- Address 0x0400 → Block index = (0x0400 / 128) mod 8 = (8) mod 8 = 0

Still both map to block 0 → **same conflict problem**, but now each miss is *more expensive* because 128 bytes must be fetched instead of 64.

**Even worse scenario—stride access:**

```
Stride-512 access: 0, 512, 1024, 1536, 2048, ... (accessing one word at each location)
```

With 64-byte blocks: Each access touches a different block, bringing in 64 bytes but using only 4 bytes (6.25% utilization).

With 256-byte blocks: Each access brings in 256 bytes but uses only 4 bytes (1.6% utilization). The cache fills 4× faster with useless data, causing earlier evictions of potentially useful blocks.

---

## Question 5: Block Size Design for Different Workloads

You're designing a cache for a streaming media application (sequential access) versus a graph analytics workload (random access). How would your block size recommendations differ, and why? Consider the three C's (compulsory, capacity, conflict) in your analysis.


**Answer**

### Streaming Media Application (Sequential Access)

**Recommended: Larger block sizes (e.g., 128–256 bytes)**

**Analysis by the Three C's:**

| Miss Type | Impact of Large Blocks |
|-----------|----------------------|
| **Compulsory** | Significantly reduced. Sequential access exhibits excellent spatial locality—once a block is fetched, *every* word in that block will be used in order. A single cold miss amortizes over the entire block. |
| **Capacity** | Minimal concern. Streaming workloads typically have small active working sets (processing data in a pipeline fashion), so fewer blocks doesn't cause premature evictions. |
| **Conflict** | Low risk. Sequential access patterns don't jump between distant addresses that might map to the same cache set. |

**Additional Benefits:**
- Memory bandwidth is used efficiently (every byte fetched gets used)
- Modern DRAM and buses are optimized for burst transfers, making large block fetches relatively cheap
- Miss penalty is amortized over many subsequent hits

---

### Graph Analytics Workload (Random Access)

**Recommended: Smaller block sizes (e.g., 32–64 bytes)**

**Analysis by the Three C's:**

| Miss Type | Impact of Large Blocks |
|-----------|----------------------|
| **Compulsory** | Large blocks provide little benefit. Random pointer-chasing means neighbors of an accessed node are unlikely to be accessed soon. Most of each fetched block goes unused. |
| **Capacity** | Severely worsened. Graph algorithms often have large, irregular working sets (e.g., traversing millions of nodes). Fewer blocks means the cache holds fewer nodes, causing frequent evictions. |
| **Conflict** | Worsened. Random access patterns increase the probability of two active addresses mapping to the same set, and having fewer total blocks exacerbates this. |

**Additional Concerns:**
- Memory bandwidth is wasted fetching unused data
- Miss penalty is high and not amortized (only one word per block is typically used)
- Cache pollution: useless data evicts potentially useful data

---

### Summary Comparison

| Factor | Streaming (Sequential) | Graph (Random) |
|--------|----------------------|----------------|
| **Spatial locality** | High | Low |
| **Optimal block size** | Large (128–256B) | Small (32–64B) |
| **Bytes used per fetch** | ~100% | ~5–15% |
| **Capacity pressure** | Low | High |
| **Bandwidth efficiency** | High | Low |

### Design Insight

This illustrates why modern processors often use different block sizes for different cache levels (e.g., 64B for L1, larger for L3), and why some architectures support **sector caches** or **sub-block fetching**—techniques that decouple the allocation unit from the transfer unit to handle diverse access patterns more gracefully.

## Question 1: The Prefetching Paradox

**Question:** Prefetching is designed to reduce cache miss rates by bringing data into the cache ahead of time. However, prefetching can actually *increase* conflict and capacity miss rates under certain circumstances. Explain this apparent paradox: under what circumstances does prefetching increase miss rates, and what hardware mechanism can mitigate this problem?

### Answer:

**The Paradox Explained:**

Prefetching brings data into the cache *speculatively*—before it is actually needed. This creates a fundamental tension: the cache now holds both demand-fetched data (data the program has actually requested) and prefetched data (data the prefetcher *predicts* will be needed).

**Circumstances that increase miss rates:**

1. **Inaccurate prefetching:** If the prefetcher's predictions are wrong, it brings in blocks that are never used. These useless blocks occupy cache space and may evict blocks that *are* being actively used by the program.

2. **Premature prefetching:** If data is prefetched too early, it may be evicted before it is actually needed (especially in a cache with limited associativity), resulting in the data being fetched twice.

3. **Overly aggressive prefetching:** Prefetching too many blocks simultaneously can flood the cache with speculative data, evicting a large portion of the working set.

**This phenomenon is called "cache pollution"**—the contamination of the cache with data that displaces more useful blocks, thereby *increasing* conflict misses (blocks evicted due to mapping to the same set) and capacity misses (blocks evicted because the cache is full).

**Example scenario:**
Consider a 4-way set-associative cache. A program is actively using 4 blocks that all map to set 7. If the prefetcher brings in a 5th block that also maps to set 7 (but will never be used), one of the 4 useful blocks must be evicted. When that evicted block is needed again, it causes a miss that would not have occurred without prefetching.

**Mitigation: The Prefetch Buffer**

A **prefetch buffer** is a small, separate storage structure that holds prefetched blocks *outside* the main cache. The workflow is:

1. Prefetched blocks are placed in the prefetch buffer (not the cache)
2. On a cache miss, both the cache and prefetch buffer are checked
3. If the block is found in the prefetch buffer (confirming the prefetch was accurate), it is promoted to the main cache
4. If a prefetched block is never used, it is simply discarded from the buffer without ever polluting the cache

This design ensures that only *validated* prefetches (those actually used by the program) enter the cache, protecting the cache from pollution caused by inaccurate speculation.

---

## Question 2: Nonbinding Prefetch Instructions

**Question:** In software prefetching, the prefetch instruction is typically described as "nonbinding." Explain what this means and discuss the consequences if the prefetch instruction were instead a binding instruction that stalled the processor until the data arrived in the cache.

### Answer:

**Definition of Nonbinding Prefetch:**

A **nonbinding prefetch** is a hint to the memory system that a particular address is likely to be needed soon. Critically, it has the following properties:

- It does **not** stall the processor—execution continues immediately after the prefetch is issued
- It does **not** raise exceptions (e.g., if the address is invalid or causes a page fault)
- It does **not** guarantee the data will be in cache when later accessed
- The hardware may ignore the hint entirely if resources are constrained

The prefetch merely initiates a memory request in the background while the processor continues executing other instructions.

**Consequences of a Binding Prefetch:**

If prefetch were a binding instruction (like a regular load), the processor would stall until the data arrived. This would be catastrophic for performance:

1. **Defeats the purpose of prefetching:** The entire point of prefetching is to overlap memory latency with useful computation. A binding prefetch would convert a "hidden" memory access into a "visible" stall—no different from simply accessing the data when needed. The memory latency would be fully exposed, not hidden.

2. **Worse than no prefetching:** A binding prefetch issued several iterations before the data is needed would cause the processor to stall *earlier* than a regular load would. The total execution time would be the same or worse.

3. **Exception handling complexity:** If a prefetch could raise page faults or protection violations, the program's exception behavior would change based on speculative predictions. A prefetch for an address that is never actually accessed could crash the program.

4. **No graceful degradation:** With nonbinding prefetches, if the prefetch doesn't complete in time, the worst case is a normal cache miss. With binding prefetches, every prefetch carries the full latency penalty if issued too late.

**Example illustrating the difference:**

```c
// With nonbinding prefetch (correct behavior):
prefetch(A[i+16]);    // Initiates memory request, continues immediately
compute(A[i]);        // Processor keeps working
// ... later, A[i+16] is likely in cache

// With hypothetical binding prefetch (problematic):
prefetch(A[i+16]);    // STALLS for ~100 cycles waiting for data
compute(A[i]);        // Only starts after prefetch completes
// Total time = prefetch_latency + compute_time (no overlap!)
```

The nonbinding nature is essential—prefetching is fundamentally a speculative optimization that must not penalize execution when predictions are wrong or timing is imperfect.

---

## Question 3: Deriving the Prefetch Condition

**Question:** In the loop prefetching example from the lecture, the following transformation is shown:

```c
// Original                    // With prefetching
for (i=0; i<=999; i++) {       for (i=0; i<=999; i++) {
  x[i] = x[i] + s;               if (i % 16 == 0)
}                                  prefetch(x[i+32]);
                                 x[i] = x[i] + s;
                               }
```

The condition `i % 16 == 0` controls when prefetches are issued. Derive this value from first principles, given that each element of array `x` is 4 bytes and a cache block is 64 bytes. Additionally, explain why the prefetch is not issued on every iteration.

### Answer:

**Derivation of the Condition:**

**Step 1: Determine how many array elements fit in one cache block**

- Each element of `x` is 4 bytes
- Each cache block is 64 bytes
- Elements per block = 64 bytes ÷ 4 bytes/element = **16 elements**

This means that elements `x[0]` through `x[15]` all reside in the same cache block, `x[16]` through `x[31]` in the next block, and so on.

**Step 2: Determine prefetch frequency**

Since 16 consecutive elements share a cache block, we only need to prefetch once every 16 iterations. Prefetching more frequently would be redundant—issuing prefetches for `x[i+32]` when `i=0,1,2,...,15` would all target the same cache block (the one containing `x[32]`).

Therefore, the condition `i % 16 == 0` ensures we prefetch exactly once per cache block, at iterations i = 0, 16, 32, 48, ...

**Step 3: Verify the prefetch distance (why +32?)**

The prefetch targets `x[i+32]`, which is 32 elements (or 2 cache blocks) ahead. This "prefetch distance" is chosen to give the memory system enough time to fetch the block before it's needed:

- At i=0, we prefetch the block containing x[32] (needed at iteration 32)
- At i=16, we prefetch the block containing x[48] (needed at iteration 48)
- This provides a 32-iteration "lead time" for the prefetch to complete

**Why Not Prefetch Every Iteration?**

Issuing a prefetch every iteration would be wasteful and counterproductive:

1. **Redundancy:** 16 consecutive prefetches would all target the same cache block. Only the first has any effect; the other 15 are pure overhead.

2. **Instruction overhead:** Each prefetch instruction consumes:
   - Fetch bandwidth
   - Decode resources
   - Execution unit cycles
   - Instruction cache space
   
   Executing 16× more prefetches adds significant overhead for zero additional benefit.

3. **Memory system contention:** Redundant prefetch requests may queue up in the memory system, consuming buffer entries and potentially delaying other useful memory operations.

4. **Prefetch overhead metric:** The lecture explicitly mentions "prefetch overhead" as a cost of software prefetching. Minimizing unnecessary prefetches keeps this overhead low.

**Summary:**

| Parameter | Value | Derivation |
|-----------|-------|-----------|
| Elements per cache block | 16 | 64 bytes ÷ 4 bytes/element |
| Prefetch frequency | Every 16 iterations | One prefetch per cache block |
| Prefetch distance | 32 elements (2 blocks) | Enough lead time to hide memory latency |
| Condition | `i % 16 == 0` | Triggers at block boundaries |

---


## Question 10: True/False with Justification

**Question:** For each of the following statements about prefetching, indicate whether it is True or False and provide a one-sentence justification.

**(a)** Prefetching always reduces average memory access time.

**(b)** Software prefetching requires no hardware support beyond a standard cache hierarchy.

**(c)** A prefetch buffer prevents cache pollution but may increase hit latency for prefetched blocks.

**(d)** Hardware prefetching is more effective than software prefetching for traversing binary search trees.

### Answers:

#### (a) Prefetching always reduces average memory access time.

**FALSE**

**Justification:** Inaccurate or poorly-timed prefetching can *increase* average memory access time through cache pollution (evicting useful blocks), memory bandwidth contention (prefetches compete with demand requests), and increased conflict misses—all of which can outweigh any latency hiding benefits.

---

#### (b) Software prefetching requires no hardware support beyond a standard cache hierarchy.

**FALSE**

**Justification:** Software prefetching requires ISA support in the form of a dedicated prefetch instruction (e.g., `PREFETCH` in x86, `PLD` in ARM), which the hardware must recognize and execute as a non-blocking, non-faulting memory hint—capabilities not present in a minimal cache hierarchy.

---

#### (c) A prefetch buffer prevents cache pollution but may increase hit latency for prefetched blocks.

**TRUE**

**Justification:** When a prefetched block is found in the prefetch buffer rather than the L1 cache, there is additional latency to (1) detect the buffer hit, (2) transfer the block to the cache, and (3) complete the access—making buffer hits slower than direct L1 cache hits, though still much faster than main memory accesses.

---

#### (d) Hardware prefetching is more effective than software prefetching for traversing binary search trees.

**FALSE**

**Justification:** Binary search tree traversal exhibits pointer-chasing behavior where each node's address is determined by data (the child pointer) stored in the previous node—a data-dependent access pattern that hardware prefetchers cannot predict because they can only analyze address patterns, not data values loaded from memory.

**Additional context:** Software prefetching can (with difficulty) prefetch tree nodes by:
- Prefetching both children when visiting a node: `prefetch(node->left); prefetch(node->right);`
- One prefetch will be wasted (50% accuracy), but the other hides latency

Hardware prefetchers see: address 0x1000, then 0x7230, then 0x3518... an apparently random sequence with no detectable pattern. They cannot know that 0x7230 was stored inside the block at 0x1000.


# Selected Questions with Answers

---

## Question 2: Design Justification

**Question:** Based on the empirical data showing diminishing returns beyond 4-way associativity, explain why most modern L1 caches use 4-way or 8-way set associativity rather than fully associative designs.

**Answer:**

The miss rate plateau occurs because most conflict misses involve only a small number of competing blocks. Once 4-8 ways are available, the majority of conflicts are resolved—the remaining misses are primarily compulsory or capacity misses that no amount of associativity can fix.

Smaller caches show steeper improvement curves because their limited sets mean more addresses map to each set, creating more conflicts. Larger caches naturally distribute accesses across more sets, so conflicts are already rare.

This reveals that real workloads exhibit limited "conflict depth"—typically only a handful of active blocks compete for the same set at any given time. Fully associative designs pay hardware costs to solve conflicts that rarely occur in practice.

---

## Question 3: Conceptual Reasoning

**Question:** Explain why a 4KB cache benefits dramatically from increased associativity while a 512KB cache shows minimal improvement.

**Answer:**

A 512KB cache can likely hold the entire working set of many programs, meaning most misses are compulsory (first access) rather than conflicts. Adding associativity doesn't help compulsory misses.

A 4KB cache forces many active blocks to compete for few sets. With direct-mapping, two frequently-used addresses mapping to the same set cause repeated evictions. The probability of such conflicts is high when the working set exceeds cache size significantly.

Essentially: when capacity is adequate, conflicts are rare; when capacity is insufficient, conflicts dominate and associativity provides substantial relief.

---

## Question

**Question:** Which statements correctly explain why higher associativity may increase miss penalty?

**Answer:** **B, C, and E**

- **A is incorrect** – Tag comparisons affect *hit time*, not miss penalty
- **B is correct** – LRU tracking across more ways requires more state and logic
- **C is correct** – Selecting which block to evict requires evaluating more candidates
- **D is incorrect** – Associativity has no effect on compulsory misses
- **E is correct** – On a miss requiring eviction, the cache must identify and potentially write back a victim from a larger candidate pool

---

## Question

**Question:** Describe the structural difference in tag comparison between direct-mapped and 4-way set-associative caches.

**Answer:**

In a **direct-mapped cache**, the index bits select exactly one block. Only one tag comparison occurs—a single comparator checks if the stored tag matches the requested address's tag.

In a **4-way set-associative cache**, the index selects a set containing four blocks. Four tag comparisons must happen simultaneously using four parallel comparators, followed by a 4-to-1 multiplexer to select the correct data based on which comparison succeeded.

This increases access time because:
- More comparators add hardware complexity and delay
- The multiplexer selection adds an additional logic stage
- All comparisons must complete before determining hit/miss

The parallel comparison is necessary to maintain reasonable access time, but still introduces overhead compared to the single-comparison direct-mapped design.

# Exam Questions: Selected Answers

---

## Question 2: Loop Fusion (Code Analysis)

```c
// Version A
for (i = 0; i < N; i++)
    C[i] = A[i] * 2;
for (i = 0; i < N; i++)
    D[i] = C[i] + B[i];

// Version B
for (i = 0; i < N; i++) {
    C[i] = A[i] * 2;
    D[i] = C[i] + B[i];
}
```

### (a) Which version exhibits better temporal locality for array `C`? Justify your answer.

**Answer:** **Version B** exhibits better temporal locality for array `C`.

In Version B, each element `C[i]` is written and then immediately read in the very next instruction within the same loop iteration. The value of `C[i]` is still in the cache (or even in a register) when it's needed for the second computation.

In Version A, the entire array `C` is written in the first loop, and then the entire array is read in the second loop. If array `C` is larger than the cache, by the time the second loop begins reading `C[0]`, that element may have already been evicted from the cache. This means each element of `C` must be fetched from main memory twice—once for writing and once for reading—resulting in poor temporal locality.

---

### (c) Identify a scenario where loop fusion would be *illegal* (would change program semantics). Provide a brief code example.

**Answer:** Loop fusion is illegal when there are **loop-carried dependencies** that would be violated by combining the loops.

**Example:**
```c
// Original - Two separate loops
for (i = 0; i < N; i++)
    C[i] = A[i] + 1;
for (i = 0; i < N; i++)
    D[i] = C[i-1] * 2;    // Uses C[i-1], not C[i]

// ILLEGAL fusion - Changes program behavior!
for (i = 0; i < N; i++) {
    C[i] = A[i] + 1;
    D[i] = C[i-1] * 2;    // Now reads OLD value of C[i-1]!
}
```

In the original code, when the second loop executes, ALL elements of `C` have already been computed by the first loop. So `D[i] = C[i-1] * 2` uses the newly computed value of `C[i-1]`.

In the fused version, when computing `D[1]`, we need `C[0]`, which was computed in the previous iteration—this works. But the timing and values accessed are different because in iteration `i`, we're reading `C[i-1]` which was written in iteration `i-1`, not necessarily with all prior context the original code assumed.

A clearer illegal case:
```c
// Original
for (i = 0; i < N; i++)
    C[i] = A[i];
for (i = 1; i < N; i++)
    D[i] = C[i] + C[i-1];  // Needs ALL of C computed first
```

Fusing would cause `D[i]` to potentially use an uninitialized or stale `C[i-1]` in certain iterations.

---

## Question 3: Matrix Multiplication Locality (Conceptual)

Consider the standard matrix multiplication algorithm with three nested loops computing `X[i][j] = Σ Y[i][k] * Z[k][j]`, where the loops iterate over `i`, then `j`, then `k` (from outermost to innermost).

Assume row-major storage order (C-style arrays).

### (b) Which matrix suffers from the worst temporal locality across iterations of the middle loop? Explain your reasoning.

**Answer:** **Matrix Z** suffers from the worst temporal locality across iterations of the middle (j) loop.

Here's the reasoning:

- When `j` changes (middle loop advances), we access `Z[k][j]` for `k = 0, 1, 2, ...` in the inner loop
- When `j` increments to `j+1`, we access `Z[k][j+1]` for the same values of `k`

The problem is that `Z[0][j]`, `Z[1][j]`, `Z[2][j]`, etc. are accessed for a particular `j`, but then we move to column `j+1`. We won't access column `j` again until `i` increments and we cycle back through all `j` values.

This means:
- **Z's columns are accessed once per (i,j) pair, then not reused until the next value of i**
- If the matrix is large, by the time we return to reuse elements of Z (when `i` increments), those elements have been evicted from cache
- Each element of Z is accessed N times total (once for each value of `i`), but these accesses are spread far apart in time

Matrix Y has better temporal locality because `Y[i][k]` is accessed repeatedly for all values of `j` while `i` stays fixed—the same row of Y is reused across the entire middle loop.

---

### (c) A student suggests simply switching to column-major storage for all matrices. Would this solve the locality problems? Why or why not?

**Answer:** **No**, switching all matrices to column-major storage would not solve the locality problems—it would merely **shift** which matrix has poor locality.

With row-major storage:
- `Y[i][k]` has good spatial locality (consecutive `k` accesses consecutive memory)
- `Z[k][j]` has poor spatial locality (consecutive `k` accesses memory locations a row apart)

With column-major storage:
- `Z[k][j]` would now have good spatial locality (consecutive `k` accesses consecutive memory)
- `Y[i][k]` would now have poor spatial locality (consecutive `k` accesses memory locations a column apart)

**The fundamental problem** is that matrix multiplication inherently accesses one matrix by rows and another by columns. No single storage format can provide good spatial locality for both access patterns simultaneously.

This is precisely why **loop blocking/tiling** is necessary—it doesn't change the storage format but instead restructures the computation to work on small blocks that fit in cache, improving temporal locality for all matrices regardless of storage format.

---

## Question 4: Loop Blocking/Tiling (Short Answer)

### (b) What is the relationship between the "block size" chosen for tiling and the cache size? What happens if the block size is chosen poorly?

**Answer:** The block size must be chosen so that **the working set of data for one block computation fits entirely within the cache**.

For matrix multiplication with block size B:
- We need to hold portions of three matrices: a B×B block of X, a B×B block of Y, and a B×B block of Z
- This requires approximately 3B² elements to fit in cache
- Therefore: **3B² × element_size ≤ cache_size**

**If block size is too large:**
- The blocks won't fit in cache
- Cache thrashing occurs—data is evicted before it can be reused
- Performance may be *worse* than the original unblocked code due to the added loop overhead without any cache benefit

**If block size is too small:**
- The blocks fit easily in cache, so locality is good
- However, loop overhead increases (more iterations of outer blocking loops)
- We don't fully utilize the available cache capacity
- May not amortize the overhead of loading blocks

**The optimal block size** maximizes cache utilization without exceeding capacity, typically chosen so the working set fills most (but not all) of the cache, leaving room for other necessary data.

---

## Question 5: Multiple Choice

### 5.1 Which compiler optimization specifically targets improving temporal locality by reducing the time between accesses to the same memory location?

A) Array merging  
B) Loop fusion  
C) Loop blocking  
D) Both B and C  

**Answer: D) Both B and C**

**Explanation:**
- **Loop fusion** improves temporal locality by combining loops so that data written in one computation is immediately used in the next, rather than waiting until a second loop traverses the same data.
- **Loop blocking** improves temporal locality by restructuring loops to repeatedly access a small block of data that fits in cache before moving to the next block, rather than streaming through all data once per loop.
- Array merging primarily targets **spatial** locality (placing related data adjacent in memory), not temporal locality.

---

### 5.2 Array merging transforms separate arrays into an array of structures. This optimization is most beneficial when:

A) Different arrays are accessed in completely separate loops  
B) Elements at the same index across multiple arrays are accessed together  
C) Arrays are accessed in reverse order  
D) The arrays have different data types  

**Answer: B) Elements at the same index across multiple arrays are accessed together**

**Explanation:**
When code frequently accesses `array1[i]` and `array2[i]` together (same index), these elements may be far apart in memory with separate arrays. By merging into a struct:
```c
struct merged { type1 field1; type2 field2; } array[size];
```
Now `array[i].field1` and `array[i].field2` are adjacent in memory. When one is loaded into a cache line, the other comes along with it, improving spatial locality.

Option A would actually make merging *harmful*—if arrays are accessed separately, merging wastes cache space loading unused fields.

---

### 5.3 In a matrix multiplication with row-major storage, which matrix access pattern `M[k][j]` (where `k` is the loop variable) exhibits:

A) Good spatial locality because consecutive `k` values access consecutive memory locations  
B) Poor spatial locality because consecutive `k` values access memory locations far apart  
C) Good temporal locality because the same element is reused immediately  
D) The locality depends on the value of `j`  

**Answer: B) Poor spatial locality because consecutive `k` values access memory locations far apart**

**Explanation:**
In row-major storage, elements are stored row by row: `M[0][0], M[0][1], M[0][2], ..., M[1][0], M[1][1], ...`

When accessing `M[k][j]` with `k` varying (0, 1, 2, ...) and `j` fixed:
- `M[0][j]` is at position `0 × num_cols + j`
- `M[1][j]` is at position `1 × num_cols + j`
- `M[2][j]` is at position `2 × num_cols + j`

These positions are `num_cols` elements apart—an entire row's worth of memory. Consecutive accesses jump across large memory distances, resulting in poor spatial locality (likely cache misses for each access).

---

## Question 6: True/False with Justification

### (a) Loop fusion always improves program performance.

**Answer: FALSE**

**Justification:** Loop fusion may increase register pressure (more variables live simultaneously), introduce dependencies that prevent parallelization, or in some cases the original loops may have had better cache behavior if they operated on data that fit in cache separately but not together.

---

### (b) Array merging can increase memory consumption due to structure padding.

**Answer: TRUE**

**Justification:** Compilers add padding bytes to structures to satisfy alignment requirements; for example, merging `char` and `double` arrays creates structs with padding bytes between fields, increasing total memory usage compared to separate contiguous arrays.

---

### (c) Loop blocking improves spatial locality but has no effect on temporal locality.

**Answer: FALSE**

**Justification:** Loop blocking **primarily** improves temporal locality by ensuring that data loaded into the cache is reused multiple times before being evicted; it keeps the working set small enough to remain cache-resident across multiple accesses to the same elements.

---

### (d) Compiler optimizations for locality are unnecessary if the cache is large enough to hold all program data.

**Answer: TRUE**

**Justification:** If all data fits in cache, every access after the initial load is a cache hit regardless of access pattern, making locality optimizations unnecessary—though this scenario is rare for realistic working set sizes.

---

### (e) The effectiveness of loop blocking depends on knowing the cache size at compile time or choosing an appropriate block size.

**Answer: TRUE**

**Justification:** The block size must be chosen so that the working set fits in cache; a poorly chosen block size (too large or too small) can result in no benefit or even degraded performance due to loop overhead without locality gains.

---

## Question 7: Scenario Analysis

```c
float red_channel[1024][1024];
float blue_channel[1024][1024];

// Pass 1: Normalize red channel
for (int y = 0; y < 1024; y++)
    for (int x = 0; x < 1024; x++)
        red_channel[y][x] = red_channel[y][x] / 255.0;

// Pass 2: Normalize blue channel  
for (int y = 0; y < 1024; y++)
    for (int x = 0; x < 1024; x++)
        blue_channel[y][x] = blue_channel[y][x] / 255.0;

// Pass 3: Compute combined intensity
for (int y = 0; y < 1024; y++)
    for (int x = 0; x < 1024; x++)
        result[y][x] = red_channel[y][x] + blue_channel[y][x];
```

### (b) Rewrite the code (or describe the transformation) to improve cache performance.

**Answer:** Apply **loop fusion** to combine all three passes into a single loop:

```c
float red_channel[1024][1024];
float blue_channel[1024][1024];

// Fused: Normalize both channels and compute intensity in one pass
for (int y = 0; y < 1024; y++) {
    for (int x = 0; x < 1024; x++) {
        red_channel[y][x] = red_channel[y][x] / 255.0;
        blue_channel[y][x] = blue_channel[y][x] / 255.0;
        result[y][x] = red_channel[y][x] + blue_channel[y][x];
    }
}
```

**Additionally**, if the normalized values aren't needed after computing the result, we could optimize further:

```c
for (int y = 0; y < 1024; y++) {
    for (int x = 0; x < 1024; x++) {
        result[y][x] = (red_channel[y][x] + blue_channel[y][x]) / 255.0;
    }
}
```

**Array merging** could also be applied if we restructure the data:
```c
struct pixel {
    float red;
    float blue;
} channels[1024][1024];
```

---

### (c) Explain the locality improvements your transformation achieves.

**Answer:**

**Temporal Locality Improvements:**
- In the original code, `red_channel[y][x]` is accessed in Pass 1, then not accessed again until Pass 3. With a 1024×1024 array of floats (4 MB), the entire array is likely evicted from cache between passes.
- In the fused version, each `red_channel[y][x]` is read, normalized, and used for the result computation **within the same iteration**. The value stays in cache (or registers) for immediate reuse.
- The same improvement applies to `blue_channel`—accessed twice in quick succession instead of separated by millions of other memory accesses.

**Quantitative Impact:**
- Original: Each array element is loaded from memory twice (once per pass that uses it) = 3 arrays × 2 loads × 1M elements = 6 million cache misses (approximately)
- Fused: Each element loaded once, reused immediately = 3 arrays × 1 load × 1M elements = 3 million cache misses (approximately 50% reduction)

**Spatial Locality:**
- Spatial locality remains good in both versions since arrays are traversed in row-major order matching storage layout
- The fused version maintains this while adding the temporal locality benefits

# Selected Questions with Short Answers

---

**Question 1(a): Explain what happens during a memory access in each configuration (series vs. parallel) when there is an L1 miss but a victim cache hit.**

**Answer:** In **series configuration**, the victim cache is checked only after an L1 miss is confirmed, adding latency but consuming less energy. In **parallel configuration**, both L1 and victim cache are checked simultaneously—if L1 misses but victim cache hits, the data is available immediately without additional lookup delay, but this consumes more power since both caches are accessed on every request.

---

**Question 1(c): Given that victim caches are typically fully-associative with 8-32 entries, explain why this design choice makes sense in terms of what type of cache misses the victim cache is intended to address.**

**Answer:** Victim caches specifically target **conflict misses**—blocks evicted from L1 due to limited associativity, not lack of capacity. Since conflict misses involve blocks that were recently evicted and may be reused soon, only a small number of entries (8-32) is needed. Full associativity is essential because the evicted blocks can map to any L1 set, so the victim cache must be able to match any address without restriction.

---

**Question 2(a): Explain why this reordering of memory operations could lead to incorrect program behavior if not handled properly. Provide a specific scenario with a code sequence demonstrating the problem.**

**Answer:** If a write to address X is sitting in the write buffer and a subsequent read to address X bypasses it to go to memory, the read would get stale data instead of the newly written value. For example:
```
sw 512(r0), r3   // write value to address 512, goes to write buffer
...
lw r2, 512(0)    // read from address 512—without checking buffer, gets old value
```
The read must check the write buffer first and return the pending write's value if addresses match.

---

**Question 3(b): For a cache with 64-byte blocks and 4-byte words, describe a scenario where critical word first provides a significant advantage over early restart.**

**Answer:** If the processor requests the **last word** (word 15) of a 64-byte block, early restart must wait for words 0-14 to transfer before word 15 arrives and can be forwarded. Critical word first requests word 15 from memory first, delivers it immediately to the processor, then fills the remaining words. The processor saves nearly the entire block transfer time.

---

**Question 3(c): Under what access pattern would neither technique provide any benefit? Explain your reasoning.**

**Answer:** When the processor accesses the **first word** of the cache block. With early restart, the first word arrives first anyway during sequential transfer. With critical word first, requesting the first word first is identical to normal sequential filling. Both techniques only help when the needed word is not at the beginning of the transfer sequence.

---

**Question 4(c): The designer notices that some benchmarks show minimal improvement even with "hit under 64 misses" while others show substantial gains. What characteristics of a program's memory access pattern would explain why non-blocking caches provide little benefit for certain workloads?**

**Answer:** Programs with **strong data dependencies** between memory accesses see little benefit—if each load's result is needed before the next memory operation can be issued, the processor cannot generate multiple outstanding misses to overlap. Additionally, programs with **very high hit rates** have few misses to overlap, and **sequential/blocking algorithms** that must complete one phase before starting another cannot exploit memory-level parallelism.

---

**Question 5(a): Explain the difference between local miss rate and global miss rate. Which metric is more meaningful for evaluating overall system performance, and why?**

**Answer:** **Local miss rate** = misses at a cache level divided by accesses to that cache level. **Global miss rate** = misses at a cache level divided by total CPU memory requests. Global miss rate is more meaningful for system performance because it reflects what fraction of all memory operations actually go to main memory, representing the true "pain" felt by the processor across the entire cache hierarchy.

---

**Question 5(b): Why do L2 caches typically have much higher local miss rates than L1 caches? Your answer should address what types of memory accesses actually reach the L2 cache.**

**Answer:** L2 only sees requests that **missed in L1**—the "easy" accesses with high temporal and spatial locality are already filtered out. L2 receives the hardest-to-satisfy requests: those with poor locality, large working set accesses, and conflict misses. This filtered stream naturally has a lower hit rate than the original request stream seen by L1.

---

**Question 5(c): Looking at the relationship between L2 size and miss rates: explain why the global miss rate remains relatively stable across different L2 sizes while the local miss rate decreases significantly as L2 size increases.**

**Answer:** Global miss rate stays stable because L1 already captures most accesses—the global miss rate is dominated by L1's miss rate (which is constant regardless of L2 size). Local miss rate drops because a larger L2 can hold more of the secondary working set, converting L2 misses to hits. But since L2 accesses are already a small fraction of total requests, these improvements have limited impact on the global rate.

---

**Question 7(c): How do non-blocking caches and out-of-order execution complement each other? Why would a non-blocking cache provide less benefit in an in-order processor?**

**Answer:** Out-of-order processors can **continue executing independent instructions** while cache misses are pending, generating multiple memory requests that a non-blocking cache can service concurrently. In-order processors **stall at the first cache miss**, unable to issue subsequent independent memory operations. Without the ability to generate multiple outstanding requests, in-order processors cannot exploit the non-blocking cache's ability to handle overlapped misses.

# Exam Questions with Answers

---

**Question 1: Design Tradeoffs**

A processor design team is debating whether to use a 4-way set associative L1 cache versus a direct-mapped L1 cache of the same total size. From a *cache hit time* perspective only, explain why the direct-mapped cache would be faster. Your answer should address both the physical layout implications and the tag comparison process.

**Answer:**

The direct-mapped cache is faster for two key reasons. First, smaller and simpler caches are physically more compact with shorter wire spans, and since wires are slow, this reduces signal propagation delay. Second, a direct-mapped cache only requires one tag comparison versus four in a 4-way associative cache. Crucially, in a direct-mapped cache the single tag comparison can happen in parallel with the data fetch since there's only one candidate location—the processor can speculatively read the data while simultaneously verifying the tag. A set associative cache must wait for tag comparison to complete before selecting which block's data to output.

---

**Question 3: Scenario-Based Reasoning**

Consider a system running two processes, A and B, that both have a variable stored at virtual address 0x1000 in their respective address spaces. Process A's variable maps to physical address 0x5000, while Process B's variable maps to physical address 0x9000.

Explain what problem would occur if this system used a purely virtual address cache and the operating system performed a context switch from Process A to Process B without any special handling. What mechanism must the operating system or hardware implement to ensure correctness?

**Answer:**

Process B would incorrectly receive Process A's data. When Process B accesses virtual address 0x1000, the cache sees a hit (same virtual address) and returns the cached data from Process A's physical location 0x5000 instead of Process B's correct data at 0x9000. This is called the homonym or aliasing problem—identical virtual addresses from different processes referring to different physical data.

To fix this, the system must either flush the cache on every context switch (costly), or tag each cache entry with an Address Space Identifier (ASID) that distinguishes which process owns each entry.

---

**Question: Workload Characteristics Favoring Physical Address Caches**

Under what workload characteristics would a physical address cache potentially outperform a virtual address cache despite its longer hit time?

**Answer:**

Physical address caches outperform virtual address caches in workloads with frequent context switches or extensive memory sharing. Virtual caches require cache flushes or ASID overhead on each context switch, so systems running many concurrent processes (like servers) suffer significant penalties. Additionally, shared memory between processes creates the synonym problem in virtual caches (multiple virtual addresses pointing to the same physical location), requiring expensive coherence mechanisms. The consistent, predictable behavior of physical caches wins when context switch overhead or sharing complexity exceeds the cost of address translation on every access.