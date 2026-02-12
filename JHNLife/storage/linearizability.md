## Linearizaqbility

#### The Challenge: Maintaining Distributed Invariants

When building a Distributed Lock, we must maintain a strict Invariant: Only one client can hold a specific lock at any given time. ### The Split-Brain Problem In a standard Single-Leader Replication setup, a network partition or node failure can lead to a "Split-Brain" scenario:

- Leader Failure: The current leader goes down (or is perceived as down due to a network timeout).
- Failover: A new leader is elected.
- Resurrection: The previous leader returns. If it still believes it is the leader (e.g., it hasn't realized its lease expired), it may continue to grant or acknowledge locks.
- Result: Two leaders exist simultaneously, violating the invariant and allowing two different clients to hold the same lock.

#### Linearizability: The "Black Box" of Consistency

Linearizability (or Strong Consistency) makes a distributed system behave as if there is only one copy of the data and all operations are atomic. It acts as a "black box" where all writes are strictly ordered and fault-tolerant.

Why do we need ordered writes?

Ordering ensures that the system never "goes back in time." Once a read reflects a certain write, no subsequent read should ever return an older value.

#### Mechanisms for Ordering Writes

Different replication strategies handle the ordering of events with varying degrees of success.

**Single-Leader Replication**

This uses a Replication Log to sequence operations. The leader assigns a position to every write in the log, which followers then apply in the same order.

Limitation: As noted in the lock example, it is susceptible to stale reads or split-brain during failovers unless additional consensus protocols are used.

**Multi-Leader & Leaderless Replication**

In these systems, it is often impossible to establish a total, real-time order of writes because multiple nodes accept writes simultaneously. Instead, they focus on Conflict Resolution to bring the system into a consistent state.

#### Logical Clocks (Lamport Clocks)

Lamport Clocks provide a way to establish a partial ordering of events in a distributed system without needing synchronized physical clocks.

-Mechanism: Every node maintains a counter. On every local event, the counter increments. Every message sent includes the current counter. Upon receiving a message, a node sets its own counter to `max(local_counter,received_counter) + 1`.

- Ordering: We can sort events by the counter (epoch) and use a tie-breaker (like the Node ID) to establish a Total Order.
- Example of Lamport Clock Increment:
  - Nodes A and B start at 0.
  - Two clients write to both: `A:1`, `B:1`.
  - A client writes again to A: `A:2`.
  - That client then writes to B: B's counter becomes `max(1,2)+1=3`. Result: `B:3`.

#### The Limits of SLR / Lamport Clocks

While Lamport Clocks allow us to order events after the fact, they do not provide Linearizability.

**Non-linearizability in Version Vectors/Lamport Clocks**

Imagine two nodes, A and B.

- Client 1 writes `x=1` to Node A (Counter: 1).
- Client 2 writes `x=5` to Node B (Counter: 1).
- Because Node A hasn't replicated to Node B yet, a reader querying Node A, then Node B, then Node A again might see the sequence: `x=1` → `x=5` → `x=1`.
- Problem: The system appears to "flicker" between values because the nodes are not aware of each other's state in real-time.

**Non-linearizability in Single-Leader Replication**

- You write `x=5` then `x=10` to the Leader.
- Only x=5 has been propagated to the Follower.
- A client reads from the Leader (`x=10`), but then a subsequent read hits the Follower and sees `x=5`.
- Result: The user sees a value from the past, violating linearizability.

#### The Solution: Distributed Consensus

To achieve true linearizability and prevent issues like split-brain, we need Distributed Consensus (e.g., Raft, Paxos, or Zab). These protocols ensure that a majority of nodes agree on the ordering of every write before it is considered committed, effectively turning the "black box" of linearizable storage into a reality.

<br>

---

<br>

### Distributed Consensus. Raft

#### Raft Leader Election

**Core Concept: Linearizability via Consensus**

To build a linearizable data store, we require Distributed Consensus. The primary goal is to create a distributed log that is totally ordered. If the log is strictly ordered and replicated correctly across multiple nodes, the state machine applying this log effectively becomes linearizable. Raft is the standard consensus algorithm used to achieve this.

**Raft Log Structure**

The foundation of Raft is the log. Every message (Log Entry) generally consists of three specific parts:

- Term: The logical time (epoch) when the entry was created.
- Index: The integer position of the entry in the log.
- Operation: The actual state machine command (e.g., `set x = 5`).

**Node States & Replication**

A Raft cluster consists of a Leader and multiple Followers. The system uses a Quorum-based replication strategy, which is semi-synchronous. The Leader sends log entries to Followers and, while Followers can technically be behind, the Leader must wait for a majority (quorum) to acknowledge an entry before it can be safely committed.

To maintain authority, the Leader periodically sends empty messages known as heartbeats. These assert the Leader's presence and prevent Followers from triggering new elections.

**The Leader Election Process**

If a Follower does not receive a heartbeat within a randomized timeframe (the election timeout), it assumes the Leader has failed.

Each node has a distinct, randomized timeout (typically 150ms–300ms). This randomization is crucial to prevent split votes, where multiple nodes wake up simultaneously and split the ballot, preventing a majority win. When this timeout expires, the Follower promotes itself to a Candidate.

The Candidate increments its current `term` by 1, votes for itself, and broadcasts `RequestVote` RPCs to all other nodes. The receiving nodes (Voters) evaluate the request based on specific rules:

- Term Check: If the Candidate's term is lower than the voter's, the vote is rejected.
- Single Vote Rule: If the voter has already voted for another node in this term, the request is rejected.
- Log Freshness: If the voter’s log is more up-to-date than the Candidate’s, the vote is rejected. This ensures a Candidate cannot win unless it contains all committed entries.

If the Candidate receives votes from a Quorum (Majority), it becomes the Leader and immediately sends heartbeats to assert control. If another node establishes leadership first (by sending a heartbeat with a higher term), the Candidate steps down. If no one achieves a majority, the term ends without a winner, and a new election begins.

**Why It Works (Safety Properties)**

Raft guarantees safety and linearizability through several key mechanisms:

- Leader Uniqueness: A node requires a majority to win. Since any two majorities must overlap by at least one node, and a node can only vote once per term, it is mathematically impossible to elect two leaders in the same term.
- The Epoch (Term) as a Fencing Token: The Term acts as a fencing token. If an old leader (e.g., one that was network partitioned) wakes up and attempts to replicate data, nodes will reject the request because they have already seen a higher Term. This effectively "fences" the old leader off, forcing it to step down.
- Leader Completeness: To win an election, a candidate must have the most up-to-date log. This guarantees that a new leader will never overwrite previously committed data, ensuring zero data loss.

#### Raft Writes

**The Challenge: Divergent Logs**

Once a Leader is elected, it handles all client requests. However, simply appending to the Leader's log isn't enough. We must guarantee that the log is replicated safely across the cluster.

- The Reality: Followers are not always up-to-date. A Follower might be lagging (missing entries) or, worse, have divergent entries (uncommitted entries from a previous Leader that crashed).
- The Goal: The Leader must force the Followers' logs to replicate its own. Conflicting entries in Followers are overwritten.

**The Golden Rule: The Log Matching Property**

Raft relies heavily on the Log Matching Property to maintain sanity. This property states:

- If two logs contain an entry with the same Index and Term, then the logs are identical in all entries up to that index.

This creates a chain of trust. If a Follower matches the Leader at Index 100, the Leader knows for a fact that the Follower also matches at Index 1 through 99.

**Example of Log Divergence:**

Imagine a scenario where a previous leader crashed mid-write:

- Leader: ... `[Index 10: Term 20, Op A] -> [Index 11: Term 23, Op D]` ...
- Follower: ... `[Index 10: Term 20, Op A] -> [Index 11: Term 22, Op F]` ...

_Here, the logs are identical up to Index 10. At Index 11, they diverge (Term 23 vs Term 22)._

**The "Backfill" Mechanism (Repairing Logs)**

When the Leader sends a new log entry (via the `AppendEntries` RPC), it acts as a consistency check.

1. The Propose: The Leader sends the new entry (e.g., Index 21) along with the prevLogIndex (20) and prevLogTerm.
2. The Check: The Follower checks its local log at Index 20.
   - Match: If the Follower has the same Term at Index 20, it accepts the new entry and appends it.
   - Mismatch: If the Follower has a different Term (or no entry) at Index 20, it rejects the request.
3. The Backtrack: Upon rejection, the Leader decrements its `nextIndex` pointer for that specific Follower (moving "one place left") and retries with the previous entry.
4. The Repair: This process repeats until the logs match. Once a match is found, the Leader copies its log forward from that point, overwriting any conflicting data in the Follower.

**The Commit Flow (Quorum)**

Raft uses a mechanism similar to Two-Phase Commit (2PC), but it is non-blocking for the minority:

- Replicate: The Leader sends the entry to all Followers.
- Wait for Quorum: The Leader waits for a majority of Followers to acknowledge that they have written the entry to their disk.
- Commit: Once a quorum is reached, the Leader marks the entry as committed, executes it on its local state machine, and returns the result to the client.
- Notify Followers: The commit index is piggybacked on subsequent heartbeats/messages, telling Followers it is safe to apply that entry to their own state machines.

**Conclusions on Raft**

- Fault-Tolerant Linearizability: Raft successfully creates a linearizable storage system that can survive node failures.
- Performance Bottleneck: The system is throughput-limited because all writes must go through the single Leader.
- Scope vs. Distributed Transactions: Raft guarantees consensus on a single replicated log (a single shard). It does not replace protocols like Two-Phase Commit (2PC) for cross-shard or cross-partition transactions. You cannot use Raft alone to atomically update data living on two different Raft clusters.
