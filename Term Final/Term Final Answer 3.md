**3.** (10 points) Consider a uniform memory access (UMA) shared memory multiprocessor with one level of writeback/write-allocate private cache that is connected to memory via a bus (no shared cache). Assume that all of the processors are kept coherent using a bus-based MSI snooping protocol. Further, assume the following:

- Each of the processors are running at 100 million instructions per second (MIPS) on some workload with the following mix: 50% ALU, 20% loads, 10% stores, and 20% branches.
- Instruction cache miss rate is 0%.
- 5% of the loads miss in the local cache.
- 94% of the stores hit in the local cache in M state, 2% of the stores find the block in the local cache in S state, and 4% of the stores find the block in I state.
- Each bus transaction requires 32 bytes.
- Whenever a new cache block is brought into the cache, an old cache block has to be evicted. Assume that every such eviction requires a separate bus transaction.

If the machine supports a 250 MB/s bus, how many processors can it accommodate before the bus becomes saturated on this workload?


# Solving the Bus Saturation Problem

I need to calculate the bus bandwidth consumption per processor, then determine how many processors the 250 MB/s bus can support.

## Step 1: Determine Instruction Rates per Processor

At 100 MIPS with the given instruction mix:
- **ALU:** 100M × 50% = 50 million ALU ops/sec
- **Loads:** 100M × 20% = **20 million loads/sec**
- **Stores:** 100M × 10% = **10 million stores/sec**
- **Branches:** 100M × 20% = 20 million branches/sec

Only loads and stores generate bus traffic, so we focus on those.

## Step 2: Calculate Bus Transactions from Loads

**Load misses:**
- Miss rate: 5% of loads
- Load misses per second: 20,000,000 × 0.05 = **1,000,000 misses/sec**

Each load miss generates:
- **1 BusRd transaction** to fetch the block (32 bytes)
- **1 eviction transaction** to writeback the evicted block (32 bytes)

**Load miss transactions:** 1,000,000 × 2 = **2,000,000 transactions/sec**

## Step 3: Calculate Bus Transactions from Stores

**Store breakdown (10,000,000 stores/sec):**

| State | Percentage | Rate | Transactions per Store | Transactions/sec |
|-------|-----------|------|------------------------|------------------|
| M (hit) | 94% | 9,400,000/sec | 0 | 0 |
| S (hit) | 2% | 200,000/sec | 1 (BusUpgr) | 200,000 |
| I (miss) | 4% | 400,000/sec | 2 (BusRdX + eviction) | 800,000 |

**Detailed breakdown:**

- **M state (94%):** 10,000,000 × 0.94 = 9,400,000 stores/sec
  - Block already in Modified state → **no bus transaction needed**
  - Transactions: **0**

- **S state (2%):** 10,000,000 × 0.02 = 200,000 stores/sec
  - Block in Shared state → need BusUpgr to invalidate other copies
  - Each BusUpgr requires 32 bytes (per problem statement)
  - Transactions: 200,000 × 1 = **200,000 transactions/sec**

- **I state (4%):** 10,000,000 × 0.04 = 400,000 stores/sec
  - Block not in cache → write-allocate policy requires fetching block first
  - Each store miss generates:
    - 1 BusRdX transaction to fetch block with exclusive ownership (32 bytes)
    - 1 eviction transaction for the displaced block (32 bytes)
  - Transactions: 400,000 × 2 = **800,000 transactions/sec**

**Total store transactions:** 0 + 200,000 + 800,000 = **1,000,000 transactions/sec**

## Step 4: Calculate Total Bus Bandwidth per Processor

**Total transactions per processor:**
$$\text{Total transactions} = 2,000,000 + 1,000,000 = 3,000,000 \text{ transactions/sec}$$

**Total bandwidth per processor:**
$$\text{Bandwidth} = 3,000,000 \times 32 \text{ bytes} = 96,000,000 \text{ bytes/sec} = \boxed{96 \text{ MB/sec}}$$

### Bandwidth Summary Table

| Source | Transactions/sec | Bytes/sec |
|--------|------------------|-----------|
| Load misses (BusRd) | 1,000,000 | 32 MB/s |
| Load evictions | 1,000,000 | 32 MB/s |
| Store upgrades (BusUpgr) | 200,000 | 6.4 MB/s |
| Store misses (BusRdX) | 400,000 | 12.8 MB/s |
| Store evictions | 400,000 | 12.8 MB/s |
| **Total** | **3,000,000** | **96 MB/s** |

## Step 5: Calculate Maximum Number of Processors

$$\text{Number of processors} = \frac{250 \text{ MB/s}}{96 \text{ MB/s per processor}} = 2.604$$

**Verification:**
- 2 processors: 2 × 96 = 192 MB/s < 250 MB/s ✓
- 3 processors: 3 × 96 = 288 MB/s > 250 MB/s ✗

Since we cannot have a fractional number of processors, we take the floor:

$$\left\lfloor 2.604 \right\rfloor = 2 \text{ processors}$$

## Final Answer

The bus can accommodate **2 processors** before becoming saturated.

Each processor consumes **96 MB/s** of bus bandwidth, and with a bus capacity of 250 MB/s:
- 2 processors use 192 MB/s (76.8% utilization) ✓
- 3 processors would require 288 MB/s (exceeds capacity) ✗