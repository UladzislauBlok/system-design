## Replication types

### Single Leader Replication

In a Single-Leader setup, one node is designated as the "leader" (or primary) and all other nodes are "followers" (or replicas).

**The Request Path**

- Write Path: Every write request must go to the leader. The leader is responsible for determining the order of writes and updating its local storage.
- Read Path: Reads can be served by the leader or any of the followers. This allows you to scale read throughput horizontally by simply adding more follower nodes.

**Core Advantages**

- Increased Durability: Data is copied to multiple disks across different nodes; if one disk fails, the data survives.
- Scalable Throughput: By offloading reads to followers, the leader can focus its resources on processing writes.

###### Failure Scenarios and Challenges

Handling failures is where Single-Leader systems become "messy."

**Follower Failure (Catch-up recovery)**

This is the easiest to handle. Each follower keeps a log of the data changes it has received from the leader.

-The Fix: If a follower crashes, it knows its last processed position (e.g., Log Sequence Number 50). When it reboots, it simply requests all changes from the leader starting from position 51.

**Leader Failure (Failover)**

When the leader goes down, one of the followers must be promoted to the new leader. This introduces several "edge case" problems:

_1. The Unreliable Network (False Positives)_

In an asynchronous network, a follower might stop receiving "heartbeats" from the leader.

- The Conflict: Is the leader actually dead, or is there just a temporary network glitch? If a follower prematurely declares itself the leader, you risk unnecessary instability.

_2. Lost Writes (Data Divergence)_

If the system uses Asynchronous Replication, the leader might acknowledge a write to a client and then crash before that write reaches the followers.

- The Conflict: When a follower is promoted, it is missing those last few writes. If the old leader comes back online, it will have data that the new leader doesn't have. Usually, the old leader's conflicting writes must be discarded, which violates durability.

_C. Split Brain_

This is the most dangerous scenario. It occurs when two nodes both believe they are the leader.

- The Conflict: If both nodes accept writes, data will diverge rapidly, and there is no easy way to merge them later without losing data. This often happens when a network partition cuts the "old" leader off from the rest of the cluster, but it's still accessible to some clients.

**The Solution: Distributed Consensus**

To solve these problems, we cannot rely on a single node's "opinion." We need Distributed Consensus.

- Quorums: Instead of one node deciding who is leader, a majority of nodes (e.g., 3 out of 5) must agree on who the leader is.
- Consensus Algorithms: Tools like Paxos or Raft are used to ensure that even in the face of network failures, the cluster agrees on a single "Source of Truth."
- Fencing Tokens: To prevent "Split Brain," every time a new leader is elected, it receives a higher term number (or epoch). If the old leader tries to write to storage, the system checks the token and rejects the write because its "term" is expired.
