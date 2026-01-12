# Chapter 5.4 Replication: Leaderless Replication

In leaderless replication, the traditional concept of a primary node (leader) is abandoned. Instead of routing all writes through a single node that dictates execution order, any replica can directly accept write requests from clients. This architecture, often referred to as Dynamo-style after Amazon’s Dynamo system, is utilized by popular datastores such as Riak, Cassandra, and Voldemort.

Key Characteristics of Leaderless Systems:

- Decentralized Writes: Clients send write requests to multiple replicas simultaneously, or to a coordinator node that forwards them without enforcing a global write order.
- Absence of Ordering: Unlike leader-based systiems, there is no central authority to determine the sequence of operations, which significantly impacts how data consistency is managed.
- Implementation Variants: The responsibility of multi-node communication can lie with the client application itself or a designated proxy/coordinator node.

<br>

---

<br>

### Writing to the Database When a Node Is Down

In leaderless systems, availability is maintained during node outages without requiring failover. When a node is offline, it simply misses writes, and the system relies on subsequent mechanisms to restore consistency.

![quorum_intro](./images/quorum_intro.png)

**Handling Outdated Data**

When a failed node returns to service, it contains stale data. To ensure clients receive the most recent information, the system uses two primary recovery strategies:

- Read Repair: When a client reads from multiple replicas in parallel, it detects stale values using version numbers. The client then writes the newer version back to the lagging replica. This is highly effective for frequently accessed data.
- Anti-Entropy Process: A background service that constantly compares replicas for data differences and synchronizes them. Unlike replication logs, this process is unordered and may involve significant propagation delays.

**Quorums for Consistency**

The reliability of a leaderless system is defined by its Quorum configuration, which determines the minimum number of "votes" required for an operation to be valid.

Key Definitions:

- `n`: The number of replicas.
- `w`: The number of nodes that must confirm a write for it to be successful.
- `r`: The minimum number of nodes queried for a read.

To guarantee that a read returns the most recent write, the system must satisfy the Quorum Condition: `w + r > n`

This inequality ensures that the set of nodes written to and the set of nodes read from overlap by at least one node, which will host the latest version.

![wrn_example](./images/wrn_example.png)

**Quorum Configuration Trade-offs**

The parameters `n`, `w`, and `r` are typically configurable to balance performance and fault tolerance:

- Common Setup: `n=3`, `w=2`, `r=2` or `n=5`, `w=3`, `r=3`. This allows the system to tolerate the failure of `n / 2` nodes.
- Read-Heavy Workload: Setting w=n and r=1 speeds up reads but means a single node failure will block all writes.
- Availability: If the number of available nodes drops below w or r, the database returns an error for that operation. The system does not distinguish between a crashed node or a network interruption; it only requires a success response

<br>

---

<br>

### Limitations of Quorum Consistency

While the quorum condition `w + r > n` theoretically guarantees that at least one node in a read request overlaps with the latest write, it is not an absolute guarantee of consistency. In practice, leaderless systems are typically optimized for eventual consistency.

**Edge Cases for Stale Reads**

Even when the quorum condition is met, several scenarios can result in reading outdated data:

- Sloppy Quorums: If the system uses "sloppy" quorums, writes might be stored on temporary nodes that do not overlap with the nodes designated for reads.
- Concurrent Operations: If a write occurs simultaneously with a read, the read may return either the old or new value. If two writes happen concurrently, conflict resolution (like Last Write Wins) may discard updates due to clock skew.
- Partial Failures: If a write succeeds on some nodes but fails on enough to stay below w, the successful writes are not rolled back. Future reads may still see that "failed" data.
- Durability Issues: If a node with the latest value fails and restores its data from a stale replica, the total count of nodes with the newest version can drop below w.

_Relaxing the Quorum_

If `w + r ≤ n`, the quorum condition is not satisfied.

- Trade-off: This configuration significantly increases the risk of stale reads.
- Benefit: It offers lower latency and higher availability, as the system requires fewer nodes to respond successfully before completing an operation.

**Monitoring Staleness**

Measuring how far a replica has fallen behind is more complex in leaderless systems than in leader-based ones because there is no global ordering of writes.

- Leader-based: Lag is easily measured by comparing the log positions of the leader and followers.
- Leaderless: Without a fixed write order, monitoring is difficult. If a system relies solely on read repair (without anti-entropy), infrequently accessed data can remain stale indefinitely.
- Operational Need: While "eventual consistency" is a vague term, operators should ideally quantify staleness using metrics or predictive research to ensure system health and avoid "ancient" data.

<br>

---

<br>

###
