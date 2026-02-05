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
