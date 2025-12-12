**1. State whether the following assertions are true or false. Justify your answer with reasons.**

**(a)** (2 points) A direct-mapped cache does not need to store tags with each cache line, since every memory block is mapped to exactly one cache line

**(b)** (2 points) A hardware prefetcher can possibly cause the miss-rate to increase.

**(c)** (2 points) It is possible for multiple virtual addresses to map to the same physical address.

**(d)** (2 points) It is possible for the same virtual address to map to multiple physical addresses.

# Cache and Virtual Memory Assertions

## (a) FALSE

**Assertion:** A direct-mapped cache does not need to store tags with each cache line, since every memory block is mapped to exactly one cache line.

**Reasoning:** While it's true that each memory block maps to exactly one cache line, the **converse is not true**—many different memory blocks can map to the *same* cache line. The cache line index is determined by a subset of the address bits, so all addresses sharing those index bits compete for the same line.

Tags are essential to identify *which* of the many possible blocks is currently occupying that cache line. Without tags, on a cache access, you'd have no way to verify whether the stored data corresponds to the requested address.

---

## (b) TRUE

**Assertion:** A hardware prefetcher can possibly cause the miss-rate to increase.

**Reasoning:** Although prefetchers aim to reduce miss rates, they can inadvertently *increase* them through:

- **Cache pollution:** Prefetched blocks that are never used can evict useful data, causing additional demand misses
- **Incorrect predictions:** If the prefetcher mispredicts access patterns, it wastes cache capacity
- **Thrashing:** In limited-capacity caches, aggressive prefetching can evict blocks that are still needed

---

## (c) TRUE

**Assertion:** It is possible for multiple virtual addresses to map to the same physical address.

**Reasoning:** This is known as **aliasing** and is commonly used for:

- **Shared memory** between processes (IPC)
- **Memory-mapped files** accessed by multiple processes
- **Copy-on-write (COW)** implementations (e.g., after `fork()`)
- **Shared libraries** loaded at different virtual addresses across processes

---

## (d) TRUE (with context)

**Assertion:** It is possible for the same virtual address to map to multiple physical addresses.

**Reasoning:** 
- **Across processes:** Each process has its own page table, so virtual address `0x1000` in Process A maps to a *different* physical address than `0x1000` in Process B. This is fundamental to process isolation.
- **Over time:** Within a single process, a virtual address can map to different physical addresses due to page swapping, page migration, or OS memory management.

*Note:* Within a **single process at a single instant**, a virtual address maps to exactly one physical address (or is invalid). The "multiple mappings" occur across different address spaces or time.

