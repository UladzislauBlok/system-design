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

<br>

---

<br>

### Multi Leader Replication

In this architecture, multiple nodes act as leaders. This is typically used in Multi-Data Center setups to ensure that if an entire region (like US-East) goes dark, the application can still process writes in another region (like EU-West).
Topology Types

How nodes communicate with each other determines the system's resilience:

- **Circular Topology**: Nodes are connected in a ring clockwise.
  - Risk: If one node fails, it breaks the replication chain for the entire ring unless there is a bypass mechanism.
- **Star Topology**: A central "hub" node redistributes writes to "spoke" nodes.
  - Risk: The hub is a single point of failure. If it goes down, the spoke nodes cannot sync.
- **All-to-All Topology**: Every leader sends its writes to every other leader.
  - Advantage: Most resilient to node failure.
  - Risk (Causality): Writes can arrive out of order. For example, an `UPDATE` might arrive at a node before the `INSERT` for that same row due to network fluctuations.

##### The Core Challenge: Write Conflicts

The biggest drawback of Multi-Leader is that two users can update the same piece of data on different leaders at the exact same time.

**1. Conflict Avoidance**

The simplest "solution" is to ensure that all writes for a specific record (e.g., a specific User ID) are always routed to the same leader.

- Trade-off: This effectively turns the system back into a Single-Leader model for that specific user, losing some of the "write anywhere" flexibility.

**2. Conflict Resolution: Last Write Wins (LWW)**

When two writes conflict, the system picks the "latest" one based on a timestamp and discards the other.

The Clock Problem: Physical clocks (Time-of-Day clocks) are not reliable.

- Clock Skew: Two servers will always have slightly different times. Even with NTP (Network Time Protocol), synchronization happens over an unreliable network, meaning one server might think it is 10:00:01 while the other thinks it is 10:00:02.
- Risk: A "later" write (in real-world time) could be deleted because a server with a "slow" clock assigned it an older timestamp. This leads to silent data loss

<br>

---

<br>

### Leaderless Replication

In Leaderless Replication (used by systems like Cassandra and Riak), there is no designated "primary" node. Clients send write and read requests to multiple nodes (or a coordinator node sends them on the client's behalf).
Core Concept: Quorums

- Write: A client sends a write to n replicas. If a node is down, the write still succeeds if a minimum number of nodes (a quorum) acknowledge it.
- Read: A client reads from multiple nodes to ensure they get the latest version. If different values are returned, the client uses Version Vectors or Timestamps to determine which is the most recent.

**Maintaining Consistency**

Since nodes can miss writes due to network partitions or downtime, the system must eventually bring them back in sync. There are two primary techniques:

1. Read Repair

When a client reads from several nodes and notices that one node has a stale (outdated) value, the client (or the coordinator) writes the newer value back to the stale node. - Best for: Data that is frequently read.

2. Anti-Entropy with Merkle Trees

This is a background process where nodes compare their data to find differences. Sending an entire database over the network to check for differences is too expensive, so we use Merkle Trees (Hash Trees).

How Merkle Trees Work:

- Bottom Layer (Leaves): Hash the individual values (e.g., `a = 10 -> 746`).
- Middle Layers: Combine the hashes of children and hash the result.
- Root: A single hash representing the entire state of that data range.

```
    a = 10 ---> 762 |
                    | ---> f(889)  ---> 524 |
    b = 45 ---> 127 |                       |
                                            | ---> f(1200) ---> 213
    c = 63 ---> 901 |                       |
                    | ---> f(1339) ---> 626 |
    d = 7  ---> 438 |

    Take hash of    | sum up and take hash  | sum up and take hash
    each row        |                       |

```

_Identifying the Conflict_

To find which data is out of sync, two nodes compare their Merkle Tree roots. If the roots differ, they check the children:

```
          305
         /   \
        /     \
       /       \
     119       463
     / \       /  \
    /   \     /    \
  746   821  784   949
 a=10   b=6  c=2   d=4

          vs

          642
         /   \
        /     \
       /       \
     505       463
     / \       /  \
    /   \     /    \
  746   314  784   949
 a=10   b=9  c=2   d=4

```

The Comparison Process:

- Compare Roots: `305` vs `642`. They differ.
- Compare Left Branch: `119` vs `505`. They differ.
- Compare Right Branch: `463` vs `463`. They are identical! We don't need to look at c or d.
- Narrow down Left Branch: Check children of the left branch.
  - `746` vs `746`: Match (Value a is fine).
  - `821` vs `314`: Mismatch!

Result: The system identifies that only the value for b needs to be synchronized. This significantly reduces the amount of data sent over the network.

Summary of differences:

| Feature    | Read Repair                                  | Anti-Entropy (Merkle Trees)                   |
| ---------- | -------------------------------------------- | --------------------------------------------- |
| Trigger    | Triggered by a read request.                 | Periodic background process.                  |
| Efficiency | Only repairs data that is actually accessed. | Repairs all data, including cold data.        |
| Overhead   | Minimal background impact.                   | Requires CPU/Disk to compute/maintain hashes. |

#### Quorums and Availability

In leaderless systems, we rely on the Quorum strategy to ensure that reads can "see" the most recent writes without needing every node to be online.

**1. The Quorum Rule**

To ensure a read request observes the latest write, we follow the formula: `W + R > N`

- `N`: The number of replicas (Replication Factor).
- `W`: The number of nodes that must acknowledge a write for it to be successful.
- `R`: The number of nodes we must query during a read.

Why it works: If `W + R > N`, then according to the pigeonhole principle, there must be at least one node that is a member of both the write set and the read set. This "overlap" node guarantees that the reader sees the latest version

**2. Quorum != Linearizability**

While Quorums provide eventual consistency, they do not guarantee linearizability (the appearance that there is only one copy of the data and all operations are instantaneous).

_Example of a Consistency Gap:_

- A writer sends a write to two nodes. Both writes succeed on the nodes' disks.
- However, due to a network glitch, the writer does not receive the acknowledgments and considers the write failed.
- Simultaneously, a reader queries the nodes. Because the data was actually written, the reader sees the "new" value.
- Conflict: From the writer’s perspective, the operation failed; from the reader's perspective, it succeeded. This inconsistency proves the system is not linearizable.

**3. Sloppy Quorums**

In a strict quorum, if you cannot reach W or R nodes from the "standard" set of N replicas, the operation fails. To increase availability, we can use a Sloppy Quorum.

- The Idea: If the N nodes designated to handle a specific key are unavailable, the system "extends" its search and writes to other nodes in the cluster that are not the standard handlers for that key.
- Trade-off: This improves write availability (you can almost always write somewhere), but it weakens consistency because the W and R nodes may no longer overlap.

**4. Hinted Handoff**

When a "sloppy" node accepts a write because the intended node was down, it needs a way to return that data to the rightful owner once it comes back online.

- The Process:
  - Node A (the intended owner) is down
  - Node D accepts the write as a "hint"
  - Node D stores this write in a separate local area
  - Node D monitors Node A. Once Node A is reachable, Node D "hands off" the data to Node A.
- Goal: To ensure that once the network partition is healed, the N intended replicas eventually contain the correct data.
