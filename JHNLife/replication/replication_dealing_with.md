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
