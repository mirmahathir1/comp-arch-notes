**2. Consider an L1 cache which is 2-way set associative, has a capacity of 16 KB, and a block size of 64 bytes. Assume that both virtual addresses and physical addresses have 32 bits. Also, assume that the system has a page size of 4 KB.**

**(a)** (2 points) Compute the size of the index and the size of the tag in bits.

**(b)** (4 points) Illustrate, with examples, the problems that can arise if the above L1 cache were to be implemented with a VI-PT (virtually indexed physically tagged) organization?

**(c)** (4 points) Describe two ways to fix the above problems?

# Cache Organization Problem Solution

## Part (a): Index and Tag Size Calculation

**Step 1: Determine the basic parameters**
- Block size = 64 bytes = 2⁶ → **Block offset = 6 bits**
- Cache capacity = 16 KB = 16,384 bytes

**Step 2: Calculate number of sets**
$$\text{Number of sets} = \frac{\text{Cache capacity}}{\text{Block size} \times \text{Associativity}} = \frac{16{,}384}{64 \times 2} = 128 \text{ sets}$$

**Step 3: Determine index and tag sizes**
- Number of sets = 128 = 2⁷ → **Index = 7 bits**
- **Tag = 32 − 7 − 6 = 19 bits**

---

## Part (b): Problems with VI-PT Organization

**The core issue: Index bits extend beyond the page offset**

| Address Component | Bits |
|-------------------|------|
| Block offset | 0–5 (6 bits) |
| Index | 6–12 (7 bits) |
| Tag | 13–31 (19 bits) |
| Page offset | 0–11 (12 bits) |

Since the index uses bits 6–12, but the page offset only covers bits 0–11, **bit 12 of the index comes from the virtual page number**, not the page offset.

**The Aliasing/Synonym Problem:**

When two different virtual addresses map to the same physical address (common with shared memory), they may have different values in bit 12, causing them to index into *different cache sets* while referring to the *same physical data*.

**Example:**
- Process A: VA = `0x00001040` → PA = `0x0000F040`
- Process B: VA = `0x00000040` → PA = `0x0000F040` (shared page)

| | VA (Process A) | VA (Process B) |
|---|---|---|
| Bit 12 | 1 | 0 |
| Index (bits 6-12) | 64 | 0 |

Both refer to the same physical location, but they index into **different sets**.

**Consequences:**
1. **Coherence violations**: Modifications to one copy aren't reflected in the other
2. **Stale data**: One process may read outdated cached data
3. **Wasted capacity**: Same physical block stored in multiple cache locations

---

## Part (c): Two Solutions

### Solution 1: Page Coloring (Software)

The OS restricts virtual-to-physical mappings so that any virtual addresses mapping to the same physical address have **identical values in the problematic index bit(s)** (bit 12 in this case).

- Only physical pages whose bit 12 matches the virtual page's bit 12 can be assigned
- Eliminates aliasing by construction
- **Tradeoff**: Reduces OS flexibility in page allocation

### Solution 2: Increase Associativity (Hardware)

Make the cache **4-way set associative** instead of 2-way:

$$\text{Number of sets} = \frac{16{,}384}{64 \times 4} = 64 \text{ sets} = 2^6$$

Now the index only uses **6 bits** (bits 6–11), which fits entirely within the 12-bit page offset. The index becomes part of the physical address, eliminating aliasing.

- **Tradeoff**: Higher hardware complexity, power consumption, and potentially slower access time