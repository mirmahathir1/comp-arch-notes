**4. For the following questions, consider a shared memory quadcore processor with private L1 caches, a shared L2 cache, and shared memory. Assume that the cache line size is 32 bytes and size of each integer variable is 64 bits. Also, assume that the caches are kept coherent using a directory-based cache coherence protocol with the directory integrated with the shared L2.**

Let us assume we are running a program whose objective is to compute the number of prime numbers in the first N natural numbers, where N is the input to the program. Let us assume that the program is parallelised such that each thread is responsible for finding primes in its share of the workload. For the purpose of this question, we are only interested in ensuring that the count of the number of primes is maintained (you may ignore all other issues pertaining to how the problem is parallelised – e.g. load balancing).

The method that is used to maintain the count is as follows. An array count[4] is allocated in shared memory; assume that the array is laid out contiguously in memory. Each processor maintains and increments individual counts using only loads and stores (and without using read-modify-write instructions), and later the individual counts are summed up by one of the processors (not shown).

```
P0              P1              P2              P3
--              --              --              --
...             ...             ...             ...
...             ...             ...             ...
// if prime     // if prime     // if prime     // if prime
count[1]++;     count[2]++;     count[3]++;     count[4]++;
...             ...             ...             ...
```

**(a)** (6 points) Is the method proposed above correct? That is, does it compute the correct value of count? If it is correct (incorrect), please explain why it is correct (incorrect) by describing how the proposed method interacts with the coherence protocol.

**(b)** (4 points) Explain, in detail, how you can further optimize the method so that it computes the count more efficiently without compromising on correctness? Your explanation should again describe how your optimized method interacts with the coherence protocol.

---

# Analysis of Shared Memory Prime Counter

## Part (a): Correctness Analysis

**Yes, the method is correct**, but it's inefficient.

### Why it's correct:

**Key observation about memory layout:**
- Integer size = 64 bits = 8 bytes
- Cache line size = 32 bytes
- Therefore, all 4 elements of `count[4]` fit in a **single cache line** (4 × 8 = 32 bytes)

**Correctness reasoning:**

Each processor writes to a **distinct memory location** (count[1], count[2], count[3], count[4]). Even though `count[i]++` is a non-atomic read-modify-write sequence:
```
LOAD  count[i] → register
ADD   1 → register  
STORE register → count[i]
```

There is **no data race** because no two processors ever access the same array element. The directory-based coherence protocol guarantees:

1. **Write serialization**: When P0 wants to write to count[1], it sends a request to the directory in L2
2. **Invalidation**: The directory invalidates all other L1 copies of that cache line
3. **Exclusive access**: P0 receives the line in Modified/Exclusive state and performs the write
4. **Visibility**: Before any final summation, all writes become visible through the coherence protocol

Since each `count[i]` has exactly one writer, all increments are preserved correctly.

---

## Part (b): Optimization via Padding

### The Problem: False Sharing

Although correct, the current method suffers from **false sharing**. All four count variables share one cache line, causing:
- Constant invalidation traffic between cores
- Cache line "ping-pongs" between L1 caches
- Each write by one processor invalidates the others' copies

### The Solution: Pad the Array

Ensure each counter resides on its **own cache line**:

```c
// Instead of: int count[4]
// Use padding to separate elements by 32 bytes:

struct padded_counter {
    long count;           // 8 bytes
    char padding[24];     // 24 bytes padding
} count[4];               // Each element = 32 bytes = 1 cache line
```

Or equivalently:
```c
long count[16];  // Use only count[0], count[4], count[8], count[12]
```

### How This Interacts with Coherence:

| Aspect | Before (False Sharing) | After (Padded) |
|--------|----------------------|----------------|
| Cache lines | 1 shared line | 4 separate lines |
| State per core | Constantly invalidated | Stays in Modified state |
| Directory traffic | High (invalidations on every write) | Minimal (one initial fetch per core) |
| Performance | Poor due to coherence overhead | Near-optimal parallelism |

**With padding:**
1. Each processor fetches its own cache line once (Shared → Exclusive/Modified)
2. All subsequent increments hit in the local L1 with no coherence traffic
3. Only during final summation does inter-core communication occur

This eliminates the false sharing bottleneck while maintaining correctness.