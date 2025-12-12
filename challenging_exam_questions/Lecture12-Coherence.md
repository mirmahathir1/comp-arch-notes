# Exam Questions & Answers: Snooping Coherence Protocols

---

## Question 1: Cache Controller Conflict

**Question:** In a snooping-based coherence system, explain why a conflict arises when both the processor and the bus-side controller need to access the cache simultaneously. Describe two hardware solutions to this problem and discuss the tradeoffs of each approach.

**Answer:** A conflict arises because the processor needs to access the cache for its own load/store operations while simultaneously the bus-side controller must check cache tags and state to respond to snooped transactions from other processors. Both need to read (and potentially modify) the tag and state arrays at the same time.

Two solutions exist:

1. **Dual-ported modules** for the tag and state array — allows simultaneous access but increases hardware cost and complexity.

2. **Duplicate tag and state arrays** — one copy for the processor, one for the bus-side controller. This is simpler but requires keeping both copies consistent whenever either is modified, adding synchronization overhead.

---

## Question 2: Race Condition Scenario

**Question:** Consider two processors P1 and P2, each with a cache line for block X in the Shared (S) state. Both processors simultaneously attempt to write to block X.

**(a)** Explain the sequence of events that creates a race condition in this scenario.

**Answer:** Both processors issue upgrade transactions (S→M) that must wait for bus access. The upgrade transaction waits for the bus, then sends invalidation messages to other sharers. Since both are trying to upgrade simultaneously, they race for bus arbitration.

**(b)** Describe what the "losing" processor must do after the "winning" processor's transaction completes. Why can't it simply proceed with its original upgrade request?

**Answer:** The losing processor's cache line gets invalidated by the winner's transaction. It cannot simply issue an upgrade because it no longer has the data (state is now Invalid, not Shared). It must instead issue a full write miss request to obtain the data from the winner's cache.

**(c)** What mechanism allows the system to determine which processor "wins"?

**Answer:** Bus arbitration determines the winner. The bus provides a serialization point where any transaction A is ordered either before or after transaction B, establishing a clear winner.

---

## Question 3: Transient States

**(a)** Explain why transient states are necessary in snooping protocols, even when the bus itself provides atomicity.

**Answer:** State transitions are not instantaneous and require multiple steps. A read-miss transaction involves waiting for the bus, waiting for bus-side controllers to check caches, and waiting for data response. During this multi-step process, conflicting requests may arrive that must be handled correctly. Transient states track that a transition is in progress.

**(b)** In the Extended MESI protocol diagram, identify what the state "I→M" represents and under what circumstances a cache line would enter this state.

**Answer:** "I→M" is a transient state entered when a CPU issues a write miss from the Invalid state. The cache line remains in this transient state while waiting for the bus to be granted. Once bus access is granted, it transitions to the stable Modified state.

**(c)** What event causes a transition out of the "S→M" transient state back to the Shared stable state (via "conflict")?

**Answer:** A conflict occurs when another processor wins bus arbitration for the same cache line. The losing processor's upgrade request is preempted, and since its line was invalidated by the winner, it must return to handle this conflict (effectively restarting with a write miss from Invalid state).

---

## Question 4: Memory Response Timing

**Question:** In a snooping system with write-back caches, memory cannot immediately respond to a read miss request on the bus. Explain the "Wired OR" mechanism that determines when memory should provide data versus when another cache will supply it. Why is this coordination necessary?

**Answer:** Memory waits until the inhibit signal is deasserted, then checks the Wired OR of two signals: "sharers" and "modified." If this Wired OR is false (no cache has the line in Shared or Modified state), memory provides the data. If true, another cache has the most up-to-date copy and will supply it.

This coordination is necessary because with write-back caches, memory may hold stale data. If a cache has a Modified copy, that cache—not memory—must supply the current data to maintain coherence.

---

## Question 5: Design Analysis

**Question:** A system architect proposes eliminating transient states to simplify the coherence protocol, arguing that bus atomicity is sufficient for correctness. Construct a specific counterexample scenario demonstrating why this would lead to incorrect behavior or protocol deadlock.

**Answer:** Consider P1 initiating a read miss from Invalid state. Without transient states, P1 might assume it will reach Shared state. While waiting for data, P2 issues a write that invalidates P1's line. If P1's controller isn't tracking that it's mid-transition, it might:

1. Incorrectly ignore the invalidation (thinking it's still Invalid)
2. Complete its read and enter Shared state, missing the invalidation entirely
3. Result in P1 having a stale Shared copy while P2 has Modified—violating coherence

Transient states (like I→S,E) allow the controller to properly handle such intervening bus transactions and take corrective action, such as restarting the transaction or updating the target state based on observed conflicts.

# Selected Exam Questions with Answers

---

## Question 1: Inclusion Property Violation Scenario

**Question:** Consider a system with a 2-way set-associative L1 cache and a 4-way set-associative L2 cache, both using LRU replacement. Three memory lines X, Y, and Z all map to the same set in both L1 and L2. Initially, only line X is cached in both L1 and L2. Trace through the following sequence of processor accesses: Y, Z, X, and explain at which point (if any) the inclusion property is violated. Describe the state of both caches after each access and identify the fundamental reason why this violation occurs despite L2 having higher associativity.

**Answer:**

| Access | L1 State | L2 State | Notes |
|--------|----------|----------|-------|
| Initial | [X, -] | [X, -, -, -] | X in both |
| Y | [X, Y] | [X, Y, -, -] | Y fetched, fills empty slots |
| Z | [Y, Z] | [X, Y, Z, -] | Z fetched; L1 evicts X (LRU), but L2 keeps X |
| X | [Z, X] | [Y, Z, X, -] | X hits in L2, fills L1; L1 evicts Y |

**Violation occurs after access to Z.** At this point, X remains in L2 but has been evicted from L1. The fundamental problem is that L1 and L2 have independent LRU states. Even though L2 has higher associativity, the replacement decisions are made locally at each level without coordination. The processor's access pattern updates L1's LRU ordering, but L2 only sees misses—it doesn't track which lines are "hot" in L1. This causes L1 to evict lines that L2 still retains.

---

## Question 2: Modified-But-Stale State

**Question:** Explain why a "modified-but-stale" state is necessary in L2 when maintaining the inclusion property with a write-back L1 cache. Describe a specific scenario where the absence of this state would lead to data loss or coherence violation. What alternative design choice could eliminate the need for this state, and what performance trade-off would it introduce?

**Answer:**

The modified-but-stale state tracks lines where L1 holds the only valid (dirty) copy, while L2's copy is outdated. This is necessary because with write-back L1, modifications stay in L1 and aren't immediately propagated to L2.

**Scenario without this state:** 
1. P1 loads line A into L1 and L2 (both in Shared state)
2. P1 writes to A → L1 goes to Modified, but L2 still thinks it has valid data
3. P2 requests A → snooping controller checks L2, finds the line, and supplies stale data
4. **Result:** P2 receives incorrect data; coherence violated

With the modified-but-stale state, L2 knows the line is dirty somewhere in L1 and can either forward the snoop to L1 or intervene appropriately.

**Alternative:** Make L1 write-through. All writes immediately update L2, so L2 always has current data. **Trade-off:** Significantly increased memory traffic and L1-L2 bandwidth consumption, higher write latency, and reduced effectiveness of write buffering.

---

## Question 5: Split-Transaction Bus Deadlock

**Question:** In a split-transaction bus system, explain how deadlock could potentially occur when using negative acknowledgments (NACKs) for flow control on response transactions. Why is this more problematic for responses than for requests? Describe the design constraint mentioned in the slides that can eliminate this deadlock possibility entirely.

**Answer:**

**Deadlock scenario:** 
- Node A sends request to Node B; Node B's response buffer fills
- Node B sends request to Node A; Node A's response buffer fills  
- Both nodes need to send responses but cannot accept each other's responses
- Both NACK incoming responses while waiting for buffer space
- Neither can make progress → deadlock

**Why responses are more problematic:** Requests can be NACKed and retried later without correctness issues—the requester simply waits. But responses complete transactions that may be blocking other operations. If a processor is stalled waiting for a response, it cannot process other work that might free buffer space. This creates circular dependencies that don't exist with request-only NACKs.

**Design constraint solution:** Size all response queues for the worst-case scenario—ensure each node has enough buffer space to hold responses for all possible outstanding requests simultaneously. This guarantees responses can always be accepted, eliminating the need to NACK responses and preventing deadlock.

---

## Question 7: Pending Transaction Tracking

**Question:** Split-transaction buses require snoop controllers to track pending transactions. Explain why this tracking is necessary for correctness when multiple requests to the same cache line can be outstanding simultaneously. Describe a race condition that could occur without this mechanism and how the "one request at a time per line" policy prevents it.

**Answer:**

**Why tracking is necessary:** In split-transaction buses, the time between request and response allows other requests to interleave. Without tracking, the system cannot determine the correct ordering or detect conflicts between overlapping operations on the same data.

**Race condition example:**
1. P1 issues Read request for line A (to get Shared copy)
2. Before P1's response arrives, P2 issues Write request for line A
3. P2's write completes first (response arrives before P1's)
4. P1's read response arrives with stale data
5. P1 now has "Shared" copy that doesn't reflect P2's write
6. **Result:** Coherence violation—P1 and P2 have inconsistent views

**How "one request at a time per line" prevents this:** When P1's read for line A is pending, P2's write request for line A is rejected (NACKed) or queued. P2 must wait until P1's transaction completes. This serializes all operations to a given line, ensuring responses match the correct system state at completion time. The snoop controller tracks which lines have pending transactions and blocks conflicting requests.

---

## Question 9: Inclusion Property Static Constraints

**Question:** The slides mention static constraints (a1=1, a2≥1, b1=b2, n1≤n2) that guarantee the inclusion property. Explain the intuition behind each constraint. Then describe a scenario where all these constraints are satisfied, yet a dynamic invalidation mechanism is still required for correctness.

**Answer:**

**Constraint intuitions:**

| Constraint | Intuition |
|------------|-----------|
| **a1=1** | L1 is direct-mapped, so each address maps to exactly one L1 location. No replacement "choice" means behavior is deterministic. |
| **a2≥1** | L2 must have at least as many ways; otherwise L2 couldn't hold all lines that might be in L1. |
| **b1=b2** | Equal block sizes ensure one-to-one mapping between L1 and L2 lines. If b1<b2, one L2 line covers multiple L1 lines, complicating inclusion. |
| **n1≤n2** | L2 has at least as many sets, ensuring L2 capacity can accommodate all possible L1 contents. |

**Scenario requiring dynamic invalidation despite constraints being satisfied:**

Configuration: a1=1, a2=2, b1=b2=64B, n1=4, n2=8

1. P1 has line A in L1 and L2 (Shared state)
2. P2 writes to line A on the bus
3. Coherence protocol invalidates A in P1's L2
4. **Problem:** A is still valid in P1's L1

Even with perfect static sizing, external coherence events (invalidations from other processors) require dynamic propagation from L2 to L1. The static constraints only prevent *capacity-based* inclusion violations from local accesses—they cannot prevent *coherence-based* invalidations from breaking inclusion.

---

## Question 10: Tag Matching in Split-Transaction Systems

**Question:** Explain why responses in a split-transaction bus system must be matched with their original requests using tags. What information must these tags encode? Discuss the implications for buffer sizing at each cache controller and explain why simply using the address is insufficient for matching in all cases.

**Answer:**

**Why tags are needed:** In split-transaction buses, multiple requests can be outstanding simultaneously, and responses may return out of order. The system must correctly associate each response with its originating request to deliver data to the right place and update the correct state.

**Information tags must encode:**
- **Requester ID:** Which processor/cache initiated the request
- **Transaction ID:** Unique identifier distinguishing multiple outstanding requests from the same requester
- **Transaction type:** Read, read-exclusive, upgrade, etc. (determines how to process the response)

**Buffer sizing implications:** Each controller needs buffer entries for all its outstanding requests, storing: the tag, address, requesting entity (CPU, prefetcher), destination register/buffer, and any partial state. More outstanding transactions = larger buffers = more hardware cost and complexity.

**Why address alone is insufficient:**

1. **Same requester, same address:** A processor might issue a read, get NACKed, and retry. Two requests for the same address from the same source could be in flight.

2. **Multiple requesters, same address:** P1 and P2 both request line A simultaneously. Both responses contain address A, but must go to different destinations.

3. **Upgrade transactions:** A processor with Shared copy requests upgrade to Modified. The "response" is an acknowledgment, not data. Address alone doesn't distinguish this from a read response.

Tags provide unambiguous matching regardless of address conflicts or transaction type.


# Short Answers: Directory-Based Cache Coherence

---

## Question 2: Protocol Mechanics

**Consider a directory-based coherence system with processors P0, P1, and P2. Processor P0 currently has a cache line in the Modified state, and the directory correctly reflects this.**

**a) Describe the complete sequence of messages and state transitions that occur when P2 issues a load request for this cache line.**

P2 sends a read request to the directory. The directory sees the line is Modified with P0 as owner, so it forwards the request to P0. P0 responds by sending the data value to P2 (and optionally to the directory to update memory). P0's state transitions from Modified to either Invalid or Shared (depending on protocol). P2's state transitions from Invalid to Shared. The directory updates to show the line is now Shared with P2 (and possibly P0) in the sharing vector.

**b) Why can't the directory simply send the data from main memory to P2? What would happen if it did?**

When a line is in Modified state, main memory contains stale data—the only valid copy exists in P0's cache. If the directory sent memory's value to P2, P2 would receive outdated data, violating coherence (P2 wouldn't see P0's most recent write).

**c) After the transaction completes, what are the final states in P0's cache, P2's cache, the directory, and main memory?**

P0's cache: Shared (or Invalid, depending on protocol variant). P2's cache: Shared with the correct data value. Directory: Line state = Shared, sharing vector indicates P2 (and P0 if it kept the line). Main memory: Updated with the current value (written back by P0 during the transaction).

---

## Question 3: Race Conditions and Serialization

**The slides describe a scenario where "operations have to be serialized locally" at processors.**

**a) Describe a specific race condition that can occur if P0 does not serialize its transactions locally when it has a pending read request.**

P0 sends a read request for line A to the directory. Meanwhile, P1 sends a read-exclusive request for line A. The directory responds to P0's request and sets the sharing vector, but this message is delayed. The directory then processes P1's request and tells P0 to invalidate. P0 receives the invalidation before receiving the data. Later, when P0's original data response arrives, P0 places stale data in its cache—data that should have been invalidated.

**b) What is the role of transient states in preventing such race conditions?**

Transient states (like "Invalid, waiting for data" or "Shared, waiting for acknowledgment") allow the cache controller to remember that a transaction is in progress. While in a transient state, the controller can properly handle conflicting requests—either buffering them, NACKing them, or processing them in a way that maintains correctness once the pending transaction completes.

**c) Why must the directory also serialize transactions, and what problem arises if it processes a replacement/writeback message while an ownership transfer is pending?**

If P1 requests ownership of a line that P0 owns, the directory forwards the request to P0 and expects P0 to send data to P1. If P1 then evicts the line and sends a writeback to the directory before the directory receives confirmation of the ownership transfer, the directory might incorrectly accept the writeback and think P1 is no longer the owner. When the delayed ownership-transfer confirmation arrives, the directory's state becomes inconsistent with reality.

---

## Question 7: Distributed Directories and NUMA

**Modern large-scale systems often use distributed directories in a cc-NUMA architecture.**

**a) Explain how a distributed directory differs from a centralized directory, and where directory information for a given memory line is located.**

In a centralized directory, one single directory structure contains coherence information for all memory lines—creating a potential bottleneck and single point of failure. In a distributed directory, the directory is partitioned across nodes, with each node's directory tracking coherence state for memory lines that are "homed" at that node. Directory information for a given line resides at the node whose local memory contains that line's home location.

**b) What is the "home node" for a cache line, and how is it typically determined?**

The home node is the node responsible for maintaining coherence information for a particular memory address. It's typically determined by the physical address—certain address bits map to specific nodes. For example, in a 4-node system, address bits might directly select which node is home (addresses 0-63 → Node 0, addresses 64-127 → Node 1, etc.).

**c) Describe the message path for a read request when the requesting processor, the home node, and the current owner are all different processors. Why does this create longer latencies than snooping protocols?**

The requesting processor sends a read request to the home node (1st message, 1st network hop). The home node looks up the directory, sees another node owns the line in Modified state, and forwards the request to the owner (2nd message, 2nd hop). The owner sends data back to the requester (3rd message, 3rd hop) and possibly an acknowledgment to home (4th message). This requires 3+ network traversals, whereas snooping broadcasts once and all processors (including the owner) see the request simultaneously, allowing a single-round-trip response.

---

## Question 8: Hybrid Coherence Schemes

**The slides describe combined coherence schemes that use bus-based snooping within nodes and directory protocols across nodes.**

**a) What advantages does this hierarchical approach provide over using either pure snooping or pure directory protocols?**

Bus-based snooping is simple and fast for small processor counts but doesn't scale. Pure directory protocols scale well but add latency and complexity for every transaction. The hybrid approach gets the best of both: fast, low-latency snooping for the common case of intra-node communication, while using directories only for inter-node coherence when necessary. It also allows easier scaling—add more nodes without redesigning the intra-node protocol.

**b) Explain what "two levels of state" means in this context. Why might the directory only track ownership at the node level rather than the processor level?**

The inter-node directory tracks coarse-grained state (e.g., "Node 2 has this line modified") while the intra-node snooping protocol tracks fine-grained state (which specific processor within Node 2 owns it). The directory only needs node-level tracking because once a request reaches the correct node, the local bus snoop will find the exact owner. This reduces directory storage (log₂(nodes) bits instead of log₂(processors) bits) and simplifies the directory protocol.

**c) What happens when a processor in Node A needs data that is modified by some processor in Node B, but the directory doesn't know which specific processor in Node B has the data?**

The directory forwards the request to Node B. Within Node B, the request is broadcast on the local bus (snooped). The processor in Node B that actually has the modified line responds to the snoop, provides the data, and the response is sent back to the requesting processor in Node A. The local snooping protocol resolves the ambiguity that the directory intentionally doesn't track.

---

## Question 9: Critical Analysis

**A colleague claims: "Directory protocols are strictly superior to snooping protocols because they eliminate broadcasts entirely."**

**a) Identify the flaw in this claim. Under what circumstances might a directory protocol still need to send messages to multiple processors?**

The claim is flawed because directory protocols must send invalidation messages to all sharers when a processor wants exclusive access for a write. If a line is shared by 10 processors, the directory must send 10 separate invalidation messages and collect 10 acknowledgments. This is point-to-point rather than broadcast, but the message count still scales with the number of sharers. Additionally, some directory protocols use broadcast for certain operations (like finding an owner in some implementations).

**b) For what types of sharing patterns do directory protocols perform poorly compared to their theoretical benefits?**

Directory protocols perform poorly with widely-shared data (e.g., global locks, barriers, or frequently-read shared counters). When many processors share a line and one processor writes, the directory must send invalidations to all sharers—negating the scalability benefit. Producer-consumer patterns with many consumers also suffer. Migratory data (where ownership bounces between processors) incurs the full directory lookup latency on every access.

**c) The slides mention that "directories should work well if only a small number of processors share common data at any given time." Explain why widely-shared data is problematic for directory protocols.**

With widely-shared data, every write requires invalidating many sharers, generating O(n) messages where n is the sharer count. The directory becomes a bottleneck serializing these operations. The sharing vector storage also grows. A broadcast (as in snooping) sends one message that all processors hear simultaneously, which is actually more efficient when most processors need to receive the message anyway. Directories save bandwidth only when sharing is sparse.