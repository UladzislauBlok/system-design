## Coordination Services: ZooKeeper & etcd

**The Need for Coordination**

Distributed consensus is notoriously slow, but it is strictly necessary for critical system operations. Coordination services are required to manage foundational metadata and state across a distributed system. Common use cases include:

- Unique Identifiers: Generating guaranteed unique IDs for servers and databases. (Example: ZooKeeper's sequential znodes can safely generate monotonically increasing IDs without race conditions).
- Replication Schemas: Tracking cluster topology, such as which nodes are currently leaders and which are followers.
- Partitioning Breakdown: Managing how data is sharded and keeping a directory of which server is responsible for which specific partition.

**What is a Coordination Service?**

At its core, a coordination service is a highly reliable, strongly consistent key-value store specifically designed to hold system configuration and state data safely.

- ZooKeeper: A battle-tested coordination service traditionally used in major distributed ecosystems (like older versions of Kafka, Hadoop, and HBase).
- etcd: A modern distributed key-value store built on the Raft consensus algorithm, most famous for serving as the primary backing store for all Kubernetes cluster data.

**Maintaining Linearizability and Read Strategies**

Because these services are built on top of distributed consensus algorithms (like Zab for ZooKeeper or Raft for etcd), state is replicated across multiple nodes. This introduces a challenge: follower nodes might lag slightly behind the leader.

If a client reads from a fully updated node (seeing data at index 1) and subsequently reads from a lagging node (seeing data at index 0), it breaks linearizability—the system fails to provide the illusion of a single, instantaneous, and consistent state.

To avoid stale reads and maintain linearizability, systems employ different read strategies:

- Read from the Leader: This guarantees the most up-to-date data, but it routes all read traffic to a single node, creating a bottleneck that severely impacts read latency and throughput.
- Read from Followers with sync (The ZooKeeper Approach): To scale reads while maintaining linearizability, ZooKeeper allows reading from followers using a specific synchronization mechanism.
  - The client issues an asynchronous sync command to the leader.
  - The client then waits until the specific follower it is connected to catches up to the leader's state (at the exact moment the sync was issued).
  - Once the follower is caught up, the client executes the read. This guarantees up-to-date data without hitting the leader directly for the payload. The client can safely continue reading from this follower until a new write occurs in the cluster.

**Conclusion & Best Practices**

Because they rely heavily on consensus protocols, coordination services are fundamentally too slow to be used as primary databases for general application data. Their use should be strictly reserved for the core pieces of backend configuration, metadata, and cluster management that absolutely demand strict correctness and fault tolerance.
