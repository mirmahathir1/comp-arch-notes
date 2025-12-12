# Lecture 8

---

## Question
**Explain the fundamental trade-off between miss rate and access time as you move from direct-mapped to fully associative caches. Why can't we simply use fully associative caches everywhere?**

**Answer:**

**The Trade-off:**
- **Direct-mapped** caches have the fastest access time because the block location is determined by a simple calculation—only one location needs to be checked. However, this rigidity means multiple blocks competing for the same location causes frequent evictions, resulting in higher miss rates.

- **Fully associative** caches achieve the lowest miss rate because any block can go anywhere, eliminating conflict misses entirely. However, to find a block, we must search every cache entry simultaneously, requiring comparators for every tag. This increases access time, hardware complexity, and power consumption significantly.

**Why not fully associative everywhere?**
1. **Hardware cost:** Requires one comparator per cache line (expensive for large caches)
2. **Power consumption:** All tag comparisons happen in parallel, consuming substantial energy
3. **Access time:** The comparison and selection logic adds latency, which is critical for L1 caches where speed is paramount

---

## Question
**Two blocks A and B map to the same location in a direct-mapped cache and are accessed alternately in a loop. Describe the phenomenon that occurs and explain how a 2-way set-associative cache would resolve it.**

**Answer:**

**The Phenomenon: Thrashing (or Ping-Pong Effect)**

In a direct-mapped cache:
- Access A → A loaded into the single location
- Access B → B evicts A (same location)
- Access A → A evicts B (miss)
- Access B → B evicts A (miss)
- This continues indefinitely, causing a **100% miss rate** despite the cache having space

**How 2-way set-associative resolves it:**

With 2-way associativity, each set contains 2 blocks. When A and B map to the same set:
- Access A → A loaded into way 0
- Access B → B loaded into way 1 (A remains)
- Access A → Hit (A is still present)
- Access B → Hit (B is still present)

Both blocks coexist in the same set, eliminating the conflict entirely and achieving a **near 0% miss rate** after initial loads.

---

## Question

**Consider a system where miss penalty is extremely high but access time requirements are relaxed. Which associativity scheme would you recommend and why?**

**Answer:**

**Recommendation: Fully Associative Cache**

**Reasoning:**

1. **High miss penalty** means each cache miss is extremely costly (e.g., accessing slow main memory or disk). Minimizing miss rate becomes the top priority.

2. **Relaxed access time** means we can tolerate the longer lookup time required to search all cache entries.

3. Fully associative provides the **lowest possible miss rate** by:
   - Eliminating all conflict misses
   - Allowing optimal replacement policy decisions (any block can be evicted)
   - Maximizing effective cache utilization

**Real-world example:** This scenario resembles a Translation Lookaside Buffer (TLB) or disk caches, where the penalty for a miss (page table walk or disk access) is so severe that the overhead of full associativity is justified.

## Answers

---

### Question 1
**Explain why the tag comprises the "high-order bits" of the memory address rather than the low-order bits. What would happen if we used low-order bits as the tag instead?**

**Answer:**

**Why high-order bits:**
- The low-order bits represent the **offset within a block** (which byte within the block)
- The middle bits represent the **index** (which set the block maps to)
- The high-order bits are what **uniquely distinguish** blocks that map to the same set

**If we used low-order bits as the tag:**
- Low-order bits change most frequently as we access sequential memory addresses
- Adjacent memory blocks would have nearly identical tags, causing massive conflicts
- Spatial locality would be destroyed—sequential accesses would map to the same set with the same "tag," making it impossible to distinguish between different blocks
- The cache would be functionally useless since we couldn't differentiate blocks within the same set

**Example:** Addresses `0x1000`, `0x1004`, `0x1008` differ only in low bits. Using these as tags would make them indistinguishable, while their high-order bits are identical (so they should share a set, not a tag).

---

### Question 2
**In a set-associative cache, tag comparisons happen in parallel within a set. Explain why this parallel comparison is necessary and how the number of comparators relates to the associativity.**

**Answer:**

**Why parallel comparison is necessary:**
- In an m-way set-associative cache, the requested block could be in any of the m ways within the indexed set
- Sequential comparison would require m cycles in the worst case, dramatically increasing access time
- Parallel comparison allows hit/miss determination in a single cycle, maintaining performance

**Relationship to associativity:**
- An **m-way** set-associative cache requires exactly **m comparators** per set access
- Each comparator simultaneously compares the incoming tag with one of the m stored tags
- The outputs feed into an m-input OR gate (for hit detection) and an m-input multiplexer (for data selection)

**Example:**
| Associativity | Comparators Needed |
|---------------|-------------------|
| Direct-mapped (1-way) | 1 |
| 2-way | 2 |
| 4-way | 4 |
| Fully associative (n blocks) | n |

This is why fully associative caches are impractical for large caches—thousands of parallel comparators would be needed.

---

### Question 3
**Why is the index not included as part of the tag? Wouldn't storing more bits improve identification accuracy?**

**Answer:**

**Why index is excluded from the tag:**

1. **Redundancy:** The index bits are already implicitly known from *where* the block is stored. If a block is found in set 5, we already know its index bits represent "5"—storing them again wastes space.

2. **Storage efficiency:** Including index bits would increase tag size by log₂(number of sets) bits per cache line. For a cache with thousands of lines, this adds significant overhead.

3. **No accuracy improvement:** Within a given set, all blocks share the same index bits. The tag only needs to distinguish between blocks *that map to the same set*—these blocks already have identical index bits by definition.

**Example:**
Consider a cache with 256 sets (8 index bits). Two addresses mapping to set 100:
- Address A: `Tag_A | 01100100 | offset`
- Address B: `Tag_B | 01100100 | offset`

Both have index `01100100`. Storing this in the tag is wasteful since we know any block in set 100 has this index. Only `Tag_A` vs `Tag_B` matters for identification.

---

### Question 4
**A 32-bit system has a 128KB, 8-way set-associative cache with 32-byte blocks. Calculate the offset bits, index bits, tag bits, and total number of sets.**

**Answer:**

**Step 1: Offset bits**
- Block size = 32 bytes = 2⁵ bytes
- **Offset bits = 5**

**Step 2: Total number of blocks**
- Cache size = 128KB = 128 × 1024 = 131,072 bytes
- Number of blocks = 131,072 ÷ 32 = 4,096 blocks

**Step 3: Number of sets**
- Associativity = 8-way (8 blocks per set)
- **Number of sets = 4,096 ÷ 8 = 512 sets**

**Step 4: Index bits**
- Number of sets = 512 = 2⁹
- **Index bits = 9**

**Step 5: Tag bits**
- Total address bits = 32
- **Tag bits = 32 - 9 - 5 = 18 bits**

**Summary:**
| Field | Bits |
|-------|------|
| Tag | 18 |
| Index | 9 |
| Offset | 5 |
| **Total** | **32** |

---

### Question 5
**Two different cache configurations both have 256 sets. Cache A is direct-mapped with 64B blocks. Cache B is 4-way set-associative with 32B blocks. For a 32-bit address space, calculate the tag size for each cache and explain why they differ.**

**Answer:**

**Cache A (Direct-mapped, 64B blocks, 256 sets):**
- Offset bits = log₂(64) = 6 bits
- Index bits = log₂(256) = 8 bits
- **Tag bits = 32 - 8 - 6 = 18 bits**
- Total cache size = 256 × 1 × 64 = 16KB

**Cache B (4-way, 32B blocks, 256 sets):**
- Offset bits = log₂(32) = 5 bits
- Index bits = log₂(256) = 8 bits
- **Tag bits = 32 - 8 - 5 = 19 bits**
- Total cache size = 256 × 4 × 32 = 32KB

**Why they differ:**

The tag sizes differ because of the **block size**, not the associativity:
- Cache A has larger blocks (64B), requiring more offset bits (6), leaving fewer bits for the tag
- Cache B has smaller blocks (32B), requiring fewer offset bits (5), leaving more bits for the tag

**Key insight:** The number of sets determines index bits. Since both have 256 sets, both have 8 index bits. The remaining difference comes entirely from block size affecting offset bits.

---

### Question 6
**A cache designer wants to reduce tag storage overhead. Analyze the effect of doubling the block size on: (a) number of tag bits, (b) total number of tags stored, and (c) overall tag storage. Is this an effective strategy?**

**Answer:**

Assume a fixed cache size C, block size B, and associativity A.

**Original configuration:**
- Number of blocks = C / B
- Number of tags = C / B
- Tag bits = Address bits - log₂(sets) - log₂(B)

**After doubling block size (2B):**

**(a) Number of tag bits:**
- Offset increases by 1 bit (log₂(2B) = log₂(B) + 1)
- Number of sets halves, so index decreases by 1 bit
- **Net change: Tag bits remain the same** (−1 + 1 = 0)

**(b) Total number of tags stored:**
- Number of blocks = C / 2B = half as many blocks
- **Number of tags is halved**

**(c) Overall tag storage:**
- Total tag storage = (number of tags) × (tag bits)
- = (C / 2B) × (same tag bits)
- **Total tag storage is halved**

**Is this effective?**

**Yes, for reducing tag overhead**, but with significant trade-offs:
- **Increased miss penalty:** Larger blocks take longer to transfer
- **Wasted bandwidth:** If spatial locality is poor, unused bytes are fetched
- **Potential increase in conflict misses:** Fewer blocks means less flexibility

This strategy is effective only when spatial locality is high and bandwidth is abundant.

---

### Question 7
**Given address `0xABCD1234` in a system with a 2-way set-associative 32KB cache and 64B blocks, decompose the address into tag, index, and offset fields. Show your work.**

**Answer:**

**Step 1: Calculate field sizes**

- Block size = 64B = 2⁶ → **Offset = 6 bits**
- Number of blocks = 32KB ÷ 64B = 512 blocks
- Number of sets = 512 ÷ 2 = 256 sets = 2⁸ → **Index = 8 bits**
- **Tag = 32 - 8 - 6 = 18 bits**

**Step 2: Convert address to binary**

`0xABCD1234` in binary:
```
A    B    C    D    1    2    3    4
1010 1011 1100 1101 0001 0010 0011 0100
```

**Step 3: Partition the bits**

| Tag (18 bits) | Index (8 bits) | Offset (6 bits) |
|---------------|----------------|-----------------|
| `10 1011 1100 1101 0001` | `0010 0011` | `01 0100` |

**Step 4: Convert to hexadecimal**

- **Tag:** `10 1011 1100 1101 0001` = `0x2BCD1` (18 bits)
- **Index:** `0010 0011` = `0x23` = **35 in decimal**
- **Offset:** `01 0100` = `0x14` = **20 in decimal**

**Final Answer:**
| Field | Binary | Hex/Decimal |
|-------|--------|-------------|
| Tag | `10 1011 1100 1101 0001` | 0x2BCD1 |
| Index | `0010 0011` | 0x23 (set 35) |
| Offset | `01 0100` | 0x14 (byte 20) |

---

### Question 8
**Explain why increasing associativity (while keeping cache size constant) reduces the number of index bits but increases tag comparison hardware. What fundamental trade-off does this represent?**

**Answer:**

**Why index bits decrease:**

- Number of sets = (Cache size) ÷ (Block size × Associativity)
- As associativity increases, number of sets decreases
- Fewer sets means fewer index bits needed

**Example (64KB cache, 64B blocks):**

| Associativity | Sets | Index Bits |
|---------------|------|------------|
| 1-way (direct) | 1024 | 10 |
| 2-way | 512 | 9 |
| 4-way | 256 | 8 |
| 8-way | 128 | 7 |
| Fully associative | 1 | 0 |

**Why tag comparison hardware increases:**

- Within each set, we need one comparator per way
- Higher associativity = more ways = more parallel comparators
- Also requires wider multiplexers to select the correct data

**The fundamental trade-off:**

This represents the **miss rate vs. access time/complexity trade-off:**

| Aspect | Low Associativity | High Associativity |
|--------|-------------------|-------------------|
| Index bits | More | Fewer |
| Tag bits | Fewer | More |
| Comparators | Fewer | More |
| Access time | Faster | Slower |
| Power consumption | Lower | Higher |
| Miss rate | Higher | Lower |
| Hardware cost | Lower | Higher |

**Key insight:** We're trading **hardware simplicity and speed** for **flexibility and lower miss rate**. The "sweet spot" depends on cache level—L1 favors speed (low associativity), while L3 favors hit rate (high associativity).

## Question

A system has the following cache configuration:
- Cache size: 32 KB
- Associativity: 2-way set-associative
- Block (line) size: 64 bytes
- Address space: 32 bits

For the memory address `0x000249F0`, decompose it into its tag, index, and offset fields. Show all calculations and clearly identify each field in binary and its decimal/hexadecimal value.

---

## Answer

### Step 1: Calculate the Offset Field

The offset identifies which byte within a 64-byte block is being accessed.

- Block size = 64 bytes = 2⁶
- **Offset bits = 6 bits**

### Step 2: Calculate the Index Field

The index identifies which set the block maps to.

- Total cache size = 32 KB = 32,768 bytes
- Number of lines (blocks) = 32,768 ÷ 64 = 512 lines
- Number of sets = 512 lines ÷ 2 ways = 256 sets = 2⁸
- **Index bits = 8 bits**

### Step 3: Calculate the Tag Field

The tag uniquely identifies the block within a set.

- Tag bits = Total address bits − Index bits − Offset bits
- **Tag bits = 32 − 8 − 6 = 18 bits**

### Step 4: Decompose the Address

Convert `0x000249F0` to binary:

```
0x   0    0    0    2    4    9    F    0
   0000 0000 0000 0010 0100 1001 1111 0000
```

Partition into fields (Tag: 18 bits | Index: 8 bits | Offset: 6 bits):

```
Tag (18 bits)           | Index (8 bits) | Offset (6 bits)
0000 0000 0000 0010 01  | 00 1001 11     | 11 0000
```

### Step 5: Interpret Each Field

| Field | Binary | Value |
|-------|--------|-------|
| **Offset** | `11 0000` | 0x30 = **48** (byte 48 within the block) |
| **Index** | `0010 0111` | 0x27 = **39** (set 39) |
| **Tag** | `00 0000 0000 0010 01` | 0x00009 |

### Summary

For address `0x000249F0`:

| Field | Size | Binary | Interpretation |
|-------|------|--------|----------------|
| Tag | 18 bits | `00 0000 0000 0010 01` | Unique block identifier |
| Index | 8 bits | `0010 0111` | Maps to **set 39** |
| Offset | 6 bits | `11 0000` | Accesses **byte 48** within block |

The cache controller will use the index (39) to locate the correct set, then compare the tag against both ways in that set to determine a hit or miss. On a hit, the offset (48) selects the specific byte from the 64-byte block.

## Answers

---

### Question 2
**The LRU replacement policy works well in practice "because of the principle of locality." Explain specifically how temporal locality justifies the LRU approach. Provide a memory access pattern where LRU would perform optimally.**

**Answer:**

**How temporal locality justifies LRU:**

Temporal locality states that if a memory location is accessed, it is likely to be accessed again in the near future. The contrapositive is equally important: if a memory location has not been accessed for a long time, it is less likely to be accessed in the near future.

LRU exploits this principle by:
- Keeping recently accessed blocks in the cache (they are likely to be reused soon)
- Evicting blocks that haven't been used for the longest time (they are least likely to be needed)

**The logic chain:**
1. Recent access → high probability of future access → keep in cache
2. Distant past access → low probability of future access → safe to evict

**Access pattern where LRU performs optimally:**

Consider a 2-way set-associative cache set and this access sequence:
```
A, B, A, B, A, B, C, A, B, A, B
```

**LRU behavior:**
| Access | Cache State | Result | Evicted |
|--------|-------------|--------|---------|
| A | [A, -] | Miss | - |
| B | [A, B] | Miss | - |
| A | [B, A] | Hit | - |
| B | [A, B] | Hit | - |
| A | [B, A] | Hit | - |
| B | [A, B] | Hit | - |
| C | [B, C] | Miss | A (LRU) |
| A | [C, A] | Miss | B (LRU) |
| B | [A, B] | Miss | C (LRU) |
| A | [B, A] | Hit | - |
| B | [A, B] | Hit | - |

LRU achieves **6 hits out of 11 accesses** in this pattern with strong temporal locality (repeated A, B accesses).

**Key insight:** LRU performs optimally when the working set (frequently accessed blocks) fits within the cache and access patterns exhibit strong temporal locality—exactly the conditions predicted by real program behavior.

---

### Question 6
**Provide a memory access sequence where random replacement would outperform LRU. Explain why LRU fails in this scenario.**

**Answer:**

**Scenario: Cyclic access pattern larger than cache capacity**

Consider a 4-way set-associative cache (one set with 4 blocks) and this repeating access sequence:
```
A, B, C, D, E, A, B, C, D, E, A, B, C, D, E, ...
```

**Why LRU fails completely:**

| Access | Cache State (MRU→LRU) | Result | Evicted |
|--------|----------------------|--------|---------|
| A | [A, -, -, -] | Miss | - |
| B | [B, A, -, -] | Miss | - |
| C | [C, B, A, -] | Miss | - |
| D | [D, C, B, A] | Miss | - |
| E | [E, D, C, B] | Miss | A (LRU) |
| A | [A, E, D, C] | Miss | B (LRU) |
| B | [B, A, E, D] | Miss | C (LRU) |
| C | [C, B, A, E] | Miss | D (LRU) |
| D | [D, C, B, A] | Miss | E (LRU) |
| E | [E, D, C, B] | Miss | A (LRU) |

**LRU achieves 0% hit rate!** Every access evicts exactly the block needed next.

**Why random can outperform LRU:**

With random replacement, there's a 1/4 (25%) chance of evicting any particular block. When block E needs to be loaded:
- Random might evict B, C, or D instead of A
- If A survives, the next access to A is a hit

Over many iterations, random achieves approximately **20-25% hit rate** compared to LRU's **0% hit rate**.

**The fundamental problem:**

LRU assumes "not recently used = not needed soon," but cyclic patterns violate this assumption. The least-recently-used block is actually the *next* block needed. This is called **LRU thrashing** or the **Bélády anomaly scenario**.

**Real-world occurrence:** This pattern appears in:
- Streaming through large arrays
- Circular buffer processing
- Database scans exceeding cache size

---

### Question 7
**Compare the storage overhead required for LRU versus NRU in an 8-way set-associative cache. Why is NRU considered a practical compromise between LRU and random replacement?**

**Answer:**

**LRU Storage Overhead:**

True LRU requires tracking the complete ordering of all 8 blocks from most-recently to least-recently used.

Method 1: Counter per block
- Each block needs a counter to store its position (0-7)
- Bits per block = log₂(8) = 3 bits
- **Total per set = 8 × 3 = 24 bits**
- On every access, multiple counters must be updated

Method 2: Comparison matrix
- Track pairwise recency relationships
- Need C(8,2) = 28 bits per set
- **Total per set = 28 bits**

**NRU Storage Overhead:**

NRU only needs to identify the most-recently-used (MRU) block, not the complete ordering.

- Simply store the index of the MRU block
- Bits needed = log₂(8) = 3 bits
- **Total per set = 3 bits**

Alternatively, use a 1-bit "referenced" flag per block:
- **Total per set = 8 bits**
- Clear all bits periodically; set bit on access

**Comparison:**

| Metric | LRU | NRU |
|--------|-----|-----|
| Storage per set | 24-28 bits | 3-8 bits |
| Update complexity | Update multiple entries | Update 1 entry |
| Victim selection | Deterministic (oldest) | Random among non-MRU |
| Miss rate | Lowest | Slightly higher |

**Why NRU is a practical compromise:**

1. **Much lower storage overhead:** 3-8 bits vs 24-28 bits per set (70-90% reduction)

2. **Simpler update logic:** Only track one block (MRU) rather than maintaining complete ordering

3. **Performance close to LRU:** By protecting the MRU block, NRU preserves the most critical aspect of temporal locality—the most recently used block is most likely to be reused

4. **Better than random:** Random might evict the MRU block (probability 1/8), which is almost certainly a mistake. NRU guarantees the MRU block survives.

5. **Scales well:** As associativity increases, LRU overhead grows dramatically while NRU remains minimal

**Key insight:** Most of LRU's benefit comes from protecting recently-used blocks. NRU captures this benefit at a fraction of the cost by protecting just the *most* recent block.

---

### Question 10
**Some modern processors use "pseudo-LRU" instead of true LRU for set-associative caches. Why might a processor designer choose an approximation of LRU rather than true LRU? What trade-offs are involved?**

**Answer:**

**Why processors use pseudo-LRU:**

**1. Storage overhead of true LRU is prohibitive**

For an n-way cache, true LRU requires:
- log₂(n!) bits to encode all possible orderings, or
- n × log₂(n) bits using counters

| Associativity | True LRU Bits | Pseudo-LRU Bits |
|---------------|---------------|-----------------|
| 4-way | 8-12 bits | 3 bits |
| 8-way | 24-28 bits | 7 bits |
| 16-way | 64+ bits | 15 bits |

For large L3 caches with thousands of sets, this difference translates to significant silicon area.

**2. Update complexity**

True LRU requires updating multiple entries on every cache access:
- All blocks more recent than the accessed block must be demoted
- This creates a complex cascade of updates on the critical path

Pseudo-LRU (tree-based) requires updating only log₂(n) bits along a single path.

**3. Access time impact**

Cache access is time-critical, especially for L1 caches. The complex logic for true LRU can increase access latency, negating the benefit of slightly better hit rates.

**Common pseudo-LRU implementation (Tree-PLRU):**

For a 4-way cache, use 3 bits organized as a binary tree:

```
        [B0]
       /    \
    [B1]    [B2]
    /  \    /  \
   W0  W1  W2  W3
```

- Each bit points toward the "less recently used" subtree
- On access: update bits along path to point away from accessed block
- On eviction: follow bits to find victim

**Trade-offs involved:**

| Aspect | True LRU | Pseudo-LRU |
|--------|----------|------------|
| **Miss rate** | Optimal among recency-based | 1-2% higher miss rate |
| **Storage** | O(n log n) bits | O(n) bits |
| **Update speed** | O(n) operations | O(log n) operations |
| **Implementation** | Complex | Simple |
| **Power consumption** | Higher | Lower |

**When pseudo-LRU diverges from true LRU:**

Pseudo-LRU may not evict the actual LRU block. Example with tree-PLRU:
- Access sequence: W0, W1, W2, W3, W0, W1
- True LRU victim: W2
- Tree-PLRU victim: might be W3 (depending on bit states)

**Why this trade-off is acceptable:**

1. The performance gap between true LRU and pseudo-LRU is typically small (1-2% miss rate increase)

2. The hardware savings enable larger caches or higher associativity, which often compensates for the slight accuracy loss

3. Memory access patterns in real programs are often "good enough" that approximate LRU captures most benefits

**Real-world usage:**
- Intel processors use tree-based pseudo-LRU for L1/L2 caches
- Some designs use adaptive policies that switch between PLRU variants based on workload characteristics

### Question

A system architect is designing a shared-memory multiprocessor. Which write policy is more naturally suited for maintaining consistency across processor caches, and why?

A) Write-back, because it reduces bus traffic
B) Write-through, because the lower level always reflects current data
C) Write-back, because it is faster
D) Write-through, because it generates more traffic for the coherency protocol to monitor

**Correct Answer: B**

> Write-through, because the lower level always reflects current data

## Explanation

In a shared-memory multiprocessor, multiple processors may need to access the same memory locations. The fundamental coherency challenge is ensuring that when Processor A writes to address X, Processor B sees the updated value.

**Why write-through helps:**
- Every write immediately updates the lower level (shared memory)
- The shared memory always contains the "true" current value
- Other processors consulting main memory will see consistent data

**Why write-back complicates things:**
- Modified data may exist *only* in one processor's private cache
- Main memory holds stale data until eviction occurs
- If Processor B reads from main memory, it gets an outdated value
- Requires additional hardware mechanisms (snooping, directory protocols) to detect and handle this situation

## Why the other options are wrong

| Option | Problem |
|--------|---------|
| A | Reduced traffic is a write-back *advantage*, but it doesn't help coherency—it actually makes it harder |
| C | Speed is also a write-back advantage, irrelevant to coherency |
| D | More traffic is a *side effect*, not the *reason* write-through aids coherency |

Note: Modern multiprocessors typically use write-back with sophisticated coherency protocols (like MESI) to get the performance benefits while still maintaining consistency—but write-through is "more naturally suited" as the question asks.

# Answer Key

---

## Question 1

**Explain why write-allocate is typically paired with write-back, and write-no-allocate is typically paired with write-through. What would be the consequences of using write-allocate with write-through instead?**

**Answer:**

**Write-allocate + write-back pairing:**
- Write-allocate brings the block into cache expecting future accesses (locality)
- Write-back allows multiple writes to accumulate in the cache before a single write to memory
- This minimizes memory traffic since repeated writes to the same block only update the cache

**Write-no-allocate + write-through pairing:**
- Write-no-allocate assumes no future accesses, so the block isn't cached
- Write-through immediately sends writes to memory anyway
- Since data goes to memory immediately, there's little benefit to caching a block we don't expect to reuse

**Consequences of write-allocate + write-through:**
- On a write miss, you'd fetch the entire block into cache (memory read)
- Then immediately write the modified data to memory (memory write)
- Every subsequent write to that block still goes to memory
- **Result:** You pay the cost of allocation (fetching the block) but don't benefit from reduced memory traffic, making this the worst of both worlds

---

## Question 2

**A system designer argues that write-no-allocate "wastes" cache space less than write-allocate. Under what circumstances is this claim valid, and when might it actually be false?**

**Answer:**

**Claim is valid when:**
- The program performs streaming writes (writing data sequentially that is never read again)
- Write-heavy workloads with poor temporal locality
- Large data structure initialization
- Write-allocate would evict useful blocks to make room for blocks that are never reused, polluting the cache

**Claim is false when:**
- Writes exhibit strong temporal or spatial locality
- A written block will soon be read or written again
- With write-no-allocate, subsequent accesses to the same block would miss, requiring additional memory accesses
- The "wasted" space in write-allocate actually saves memory traffic and improves performance

**Key insight:** Cache space is only "wasted" if the allocated block provides no future benefit. If locality exists, write-allocate uses cache space *efficiently*, not wastefully.

---

## Question 3

**Consider a program that initializes a large array by writing to each element exactly once, then never accesses the array again. Compare the performance and memory traffic implications of write-allocate versus write-no-allocate for this workload.**

**Answer:**

**Write-allocate (with write-back):**

For each cache block:
1. Write miss → fetch entire block from memory (READ)
2. Modify word(s) in cache
3. Eventually evict dirty block → write entire block to memory (WRITE)

*Memory traffic:* 2 full block transfers per block (read + write)

*Additional overhead:* Evicts potentially useful data from cache (cache pollution)

**Write-no-allocate (with write-through):**

For each write:
1. Write miss → write only the modified word directly to memory (WRITE)
2. No block fetch, no cache allocation

*Memory traffic:* Only the actual data written (one word per write)

**Comparison:**

| Metric | Write-Allocate | Write-No-Allocate |
|--------|----------------|-------------------|
| Memory reads | 1 block per miss | 0 |
| Memory writes | 1 block per eviction | 1 word per write |
| Cache pollution | High | None |
| Total traffic | ~2× array size | ~1× array size |

**Conclusion:** Write-no-allocate is significantly better for this streaming write pattern—less memory traffic and no cache pollution.

---

## Question 4

**Why does write-allocate assume temporal or spatial locality will benefit future accesses? Describe a memory access pattern where this assumption fails and quantify the overhead introduced.**

**Answer:**

**Why locality is assumed:**
- Fetching an entire block is expensive (memory latency + bandwidth)
- This cost is only justified if:
  - **Temporal locality:** The same address is accessed again soon
  - **Spatial locality:** Nearby addresses in the same block are accessed soon
- Without locality, the fetched block provides no cache hits before eviction

**Access pattern where assumption fails:**

*Strided writes with stride ≥ block size*

Example: Block size = 64 bytes, array of 8-byte integers
```
for (i = 0; i < N; i += 8)
    A[i] = value;  // Stride of 64 bytes
```

Each write touches a different cache block, and no block is ever accessed twice.

**Overhead quantification:**

Let:
- B = block size (bytes)
- W = word size written (bytes)

Per write miss with write-allocate:
- Fetch entire block: B bytes read
- Write back dirty block: B bytes written
- Useful data written: W bytes

**Overhead ratio** = (B + B) / W = 2B/W

If B = 64 bytes and W = 8 bytes:
- Overhead = 128/8 = **16×** more memory traffic than necessary

With write-no-allocate, only 8 bytes would be written per access.

---

## Question 6

**You observe that a program's cache miss rate improves when switching from write-no-allocate to write-allocate, yet overall performance decreases. Propose two possible explanations for this counterintuitive result.**

**Answer:**

**Explanation 1: Cache pollution degrading read performance**

- Write-allocate reduces write misses by bringing written blocks into cache
- However, these blocks evict other blocks that were being used for reads
- If the program is read-heavy, the loss of read hits outweighs the write miss reduction
- Reads are often on the critical path (processor stalls waiting for data), while writes can be buffered
- **Net effect:** Fewer total misses, but more performance-critical read misses → slower execution

**Explanation 2: Increased memory bandwidth saturation**

- Write-allocate requires fetching blocks on write misses (read-for-ownership)
- Even though miss rate decreases, memory traffic increases substantially
- Memory bus becomes saturated, increasing average memory access latency
- This increased latency affects all memory operations, including the hits
- **Net effect:** Better hit rate, but each miss (and potentially each access) takes longer due to bandwidth contention

## Question
Consider a processor with a three-level cache hierarchy: L1 (32 KB), L2 (256 KB), and L3 (8 MB). A system architect is deciding between inclusive and exclusive cache policies.

---

**Question:** Calculate the maximum amount of unique data that can be cached under each policy. Explain the difference.

**Answer:**

| Policy | Calculation | Maximum Unique Data |
|--------|-------------|---------------------|
| **Inclusive** | Only L3 counts (L1, L2 are subsets) | **8 MB** |
| **Exclusive** | L1 + L2 + L3 = 32 KB + 256 KB + 8 MB | **8.28 MB** |

**Explanation:**
- **Inclusive:** Every block in L1 and L2 must also exist in L3, so the unique capacity is limited to L3's size (8 MB). The L1 and L2 capacity is "wasted" as redundant copies.
- **Exclusive:** Each block exists in exactly one level, so capacities are additive, maximizing aggregate storage.

---

**Question:** In a multiprocessor system with cache coherence, explain why an inclusive L3 cache simplifies the coherence protocol. What specific problem would arise with an exclusive cache when another processor issues a snoop request for a block?

**Answer:**

**Why Inclusive Simplifies Coherence:**
- The L3 acts as a **snoop filter**
- To check if Processor A has a block, other processors only need to query A's L3
- If the block is **not in L3**, it's guaranteed **not in L1 or L2** either
- Single lookup point eliminates searching multiple cache levels

**Problem with Exclusive Caches:**
- A requested block could reside in **any one level** (L1, L2, or L3)
- Snoop requests must either:
  - Search **all three levels** (slow, high bandwidth)
  - Maintain **additional tracking structures** (hardware overhead)
- Without inclusion, there's no single point that guarantees complete visibility into the cache hierarchy

---

**Question:** An exclusive cache hierarchy requires uniform block sizes across all levels. Explain why this constraint exists and what complications would arise if L1 used 32-byte blocks while L2 used 64-byte blocks in an exclusive design.

**Answer:**

**Why Uniform Block Sizes Are Required:**

Exclusive caches move blocks between levels on evictions and fills. Block size mismatches create fundamental problems:

| Scenario | Problem |
|----------|---------|
| **L1 eviction → L2** | A 32-byte block evicted from L1 cannot fill a 64-byte L2 slot. What occupies the remaining 32 bytes? |
| **L2 fill → L1** | A 64-byte block from L2 must split into two 32-byte L1 blocks. Does the other half stay in L2? This **violates exclusivity**. |

**Specific Complications:**

1. **Partial block residency:** Half a block in L2, half in L1 breaks the "one level only" rule
2. **Fragmentation:** Managing sub-block tracking adds significant hardware complexity
3. **Eviction ambiguity:** When evicting from L1, the system cannot cleanly transfer ownership to L2
4. **Coherence complexity:** Snoops must now track partial blocks across levels

**Inclusive caches avoid this** because redundant copies are allowed—L2 can simply keep its full 64-byte block while L1 holds a 32-byte subset.