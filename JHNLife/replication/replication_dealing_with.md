## Replication

### Dealing with Stale Reads

In a system with Eventual Consistency, there is a period called "replication lag." During this window, a client might write data to the leader but then read from a follower that hasn't received that update yet. To provide a better user experience, we aim for the following guarantees:

**1. Reading Your Own Writes (Read-after-Write Consistency)**

This ensures that if a user updates their profile, they immediately see that update when they refresh the page, even if other users don't see it yet.

- The Problem: A user writes to the primary node but reads from a lagging replica, making it look like their update was lost.
- Solutions:
  - User-Based Routing: Always read the user's own profile from the leader (primary) node.
  - Time-Based Logic: Track the timestamp of the last write. If a replica is behind that timestamp, the system redirects the read to a more up-to-date node.
  - Sticky Sessions: Route all requests from a specific User ID to the same replica.
- Note on Distributed Clocks: Physical clocks are unreliable in distributed systems; we often use Logical Clocks or Sequence Numbers instead.

**2. Monotonic Reads**

This ensures that a user does not see time "run backward"

-The Problem: A user refreshes a page multiple times. The first read hits a fast replica (shows 3 comments), but the second read hits a slower replica (shows only 1 comment). It appears as though data has disappeared.

- Solution: Replica Pinning: Ensure a specific user always reads from the same replica (e.g., by hashing the Client ID to a specific node). Even if the data is slightly stale, it will never "roll back" to an even older state.

**3. Consistent Prefix Reads**

This is critical in Partitioned (Sharded) databases where different pieces of data live on different nodes.

- The Problem (Causal Dependency): Imagine a conversation where "Message A" happens before "Message B." If Message B is replicated to a node faster than Message A, a user might see the answer before the question. The "causal" order is broken.
- Solution: Causal Partitioning: Ensure that data with a causal dependency (like a chat thread) is always written to the same partition. By keeping the conversation on one node, the replication order is preserved, ensuring the user sees the "prefix" of the conversation in the correct sequence.

| Guarantee            | Problem Addressed                         | Common Fix                                           |
| -------------------- | ----------------------------------------- | ---------------------------------------------------- |
| Read-your-own-writes | "I just posted a photo, but it's gone."   | Read user-owned data from the Leader (or same node). |
| Monotonic Reads      | "I saw 10 likes, then 5, then 10 again."  | Route user to the same replica every time.           |
| Consistent Prefix    | "The reply appeared before the question." | Keep causally related data in the same partition.    |

<br>

---

<br>

### Dealing with Write Conflicts

In multi-leader or leaderless replication, write requests can occur at multiple nodes simultaneously. While this improves availability and write throughput, it introduces the risk of write conflicts, where two nodes receive different updates for the same data point.
The Problem with Timestamps

A common but flawed approach to conflict resolution is Last Write Wins (LWW) using physical timestamps.

- Issue: Relying on timestamps is risky due to clock skew (unsynchronized hardware clocks across servers).
- Result: This can lead to silent data loss, where a "later" write (according to a skewed clock) overwrites a truly more recent update.

**Scenario: The Distributed Counter**

Consider a distributed counter where two leaders (Leader A and Leader B) accept increments concurrently for the same key.

- State: Leader A has a local count of `5`. Leader B has a local count of `3`.
- Conflict: If they simply sync by overwriting, one value will be lost.
- Requirement: Since these writes are concurrent (neither leader knew about the other's write when it occurred), we must merge the values rather than choosing one.

**Version Vectors**

To achieve eventual consistency, we replace a single counter with a Version Vector (an array of size n, where n is the number of nodes). Each index represents the number of writes seen from a specific node.
Example Walkthrough:

- Before Sync:
  - Leader A: `[5, 0]` (5 writes on A, 0 from B)
  - Leader B: `[0, 3]` (0 writes from A, 3 on B)
- During Sync: Nodes exchange vectors and take the maximum value for each index.
  - New Vector: `[max(5,0), max(0,3)]` → `[5, 3]`
- Result: When a user queries the data, the system sums the vector (5+3=8). Both nodes now reflect the same state.

**Identifying and Handling Complex Conflicts**

Not all data can be mathematically summed. We must be able to detect when one write "happened before" another versus when they are truly concurrent.

_Detecting Causality_

We compare the local version vector against the incoming (replicated) vector:

- Not Concurrent: If every element in the local vector is greater than or equal to the incoming vector, the local node has already seen these changes.
- Concurrent (Conflict): If the vectors are "interleaved."
  - Example: Local `[5, 2, 1]` vs. Incoming `[4, 3, 2]`.
- Logic: The local node is "ahead" on Leader 1 (5 > 4), but "behind" on Leader 2 (2 < 3). Neither write could have known about the other.

_Resolution Strategies_

When a conflict is detected via interleaved vectors, we have two primary paths:

- Sibling Storage (Application-level resolution): The database stores both versions (called "siblings").
  - Example: Value: `"Cart-A" | "Cart-B"`.
  - The conflict is pushed to the application. When a user next reads the data, the application logic (or the user manually) must merge the versions and write the result back to the database.
- Conflict-free Replicated Data Types (CRDTs): The data structure is designed specifically so that merges are mathematically deterministic.
  - Example: The distributed counter mentioned above is a simple form of a CRDT. Another example is a G-Set (Grow-only Set) where the merge is simply the union of two sets.

#### CRDT - Conflict-free Replicated Data Types

CRDTs are data structures designed to be replicated across multiple nodes where updates can be made independently and concurrently without coordination. They guarantee that if all nodes have received the same set of updates, they will eventually reach the same state.

**1. Operational-based CRDTs (CmRDT)**

Instead of sending the entire state, nodes broadcast the specific operations (e.g., `add(element)`, `increment(1))` to all other replicas.

- Advantage: Significantly lower network overhead because only the delta (change) is transmitted.
- Challenges:
  - Causal Dependencies: If operations are received out of order (e.g., a `remove` arrives before the `add`), the state may become inconsistent.
  - Infrastructure Requirements: Requires a reliable messaging layer that guarantees exactly-once delivery and preserves causal ordering (or the operations must be designed to be commutative).
- Example: Sending `inc(1)` for a specific index in a version vector rather than the whole vector.

**2. State-based CRDTs (CvRDT)**

Nodes send their entire state to other replicas, which then merge the incoming state with their local state.

- Gossip Protocols: These are ideal for state-based CRDTs. A node randomly selects n peers to sync with. Because the merge function is resilient, it doesn't matter if a node receives the same data multiple times or from different paths.
- Merge Function Requirements: For a state-based merge to work correctly, the function f must satisfy:
  - Commutative: `f(a,b)` = `f(b,a)` (Order doesn't matter).
  - Associative: `f(a,f(b,c))` = `f(f(a,b),c)` (Grouping doesn't matter).
  - Idempotent: `f(a,a)=a` (Applying the same state twice doesn't change the result).

_Examples of Common CRDT Data Structures_

| Type                               | Operational Approach                                                                               | State-based Approach                                                                       |
| ---------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Counter                            | Broadcast `increment(n)`.                                                                          | "Sync the entire Version Vector; Merge via `max(vec_a, vec_b)`. Result is `sum(vector)`."  |
| Decrementable Counter (PN-Counter) | Maintain two internal vectors: _P_ (Positive) and _N_ (Negative). Broadcast increments/decrements. | Sync both _P_ and _N_ vectors. Merge both using `max()`. Result is `sum(P)` - `sum(N)`.    |
| Set (OR-Set)                       | Broadcast `add(element)` or `remove(element)`.                                                     | Sync two sets: an Add Set and a Remove Set. Result is the set difference (`Add - Remove`). |

_The "Remove" Problem in Sets_

In a simple Add/Remove set, once an element is added to the "Remove Set," it can never be re-added because the "Remove" state persists forever (the "tombstone" problem).

- Solution: Tagging (Observed-Remove Set): Each time an element is added, it is assigned a unique tag (e.g., a UUID or timestamp).
  - If you add "Apple" (Tag: `101`) and then remove "Apple" (Tag: `101`), that specific instance is gone.
  - If you add "Apple" again, it gets a new tag (`102`). Since `102` is not in the "Remove Set," the element reappears correctly.
