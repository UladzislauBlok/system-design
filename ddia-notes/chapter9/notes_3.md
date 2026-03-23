# Chapter 9.3 Consistency and Consensus: Distributed Transactions and Consensus

Consensus is one of the most important and fundamental problems in distributed computing. Informally, the goal is simply to get several nodes to agree on something. Although it seems simple, many broken systems have been built in the mistaken belief that this problem is easy to solve.

There are a number of situations in which it is strictly necessary for nodes to agree:

- Leader election: In a database with single-leader replication, all nodes must agree on the leader. If a network fault causes a split-brain situation where two nodes accept writes, their data will diverge, leading to inconsistency and data loss.
- Atomic commit: In transactions spanning several nodes, all nodes must agree on the outcome: either they all abort/roll back or they all commit.

<br>

---

<br>

### Atomic Commit and Two-Phase Commit (2PC)

The outcome of a transaction is either a successful commit, making writes durable, or an abort, discarding the writes. Atomicity prevents failed transactions from littering the database with half-updated state.

On a single node, transaction commitment crucially depends on the order in which data is durably written to disk: first the data, then the commit record. The key deciding moment is when the disk finishes writing the commit record. Thus, a single device makes the commit atomic.

However, if multiple nodes are involved, simply sending independent commit requests is not sufficient. Doing so violates atomicity because:

- Some nodes may detect a constraint violation and abort, while others commit.
- Some commit requests might get lost in the network.
- Some nodes may crash before the commit record is fully written.

A transaction commit must be irrevocable because committed data immediately becomes visible to other transactions. Therefore, a node must only commit once it is certain that all other nodes are also going to commit.

**Introduction to two-phase commit**

Two-phase commit (2PC) is an algorithm for achieving atomic transaction commit across multiple nodes. (Note: 2PC is entirely separate from Two-Phase Locking, or 2PL, which provides serializable isolation rather than atomic commit).

![two_pc](./images/two_pc.png)

2PC uses a new component: a coordinator (or transaction manager). Instead of a single commit request, the process is split into two phases:

- Phase 1 (Prepare): The coordinator sends a prepare request to each participant node. The node ensures it can definitely commit (writing data to disk and checking conflicts). By replying "yes," the node promises to commit without error if requested later, effectively surrendering its right to unilaterally abort.
- Phase 2 (Commit/Abort): If all participants reply "yes," the coordinator makes a definitive decision and writes it to its own disk log (the commit point). It then sends a commit request to all participants. If any participant replies "no" or times out, the coordinator sends an abort request instead.

These two points of no return—participants promising they can commit, and the coordinator's irrevocable decision—ensure the atomicity of 2PC.

**Coordinator failure**

If a participant or the network fails, the coordinator can abort the transaction or retry requests indefinitely. However, if the coordinator crashes after sending prepare requests, a participant who voted "yes" cannot abort unilaterally.

![coordinator_crashes](./images/coordinator_crashes.png)

Without hearing from the coordinator, the participant has no way of knowing whether to commit or abort. A participant in this state is called in doubt or uncertain. The only way 2PC can complete is by waiting for the coordinator to recover and read its own transaction log to determine the status of the in-doubt transactions.

Because 2PC can become stuck waiting for the coordinator to recover, it is called a blocking atomic commit protocol. An alternative, three-phase commit (3PC), is nonblocking but assumes a network with bounded delay and perfect failure detectors. Because a timeout is not a reliable failure detector in real-world networks with unbounded delays, 2PC continues to be used despite the risk of coordinator failure.

<br>

---

<br>

### Distributed Transactions in Practice

Distributed transactions, especially those implemented with two-phase commit, have a mixed reputation. While they provide important safety guarantees, they are criticized for causing operational problems and severe performance penalties. Much of this cost is due to the additional disk forcing (`fsync`) required for crash recovery and the extra network round-trips.

To understand them better, we must distinguish between two quite different types of distributed transactions:

- Database-internal distributed transactions: All participating nodes run the same database software. Because they do not need to be compatible with other systems, they can use technology-specific protocols and optimizations, and therefore often work quite well.
- Heterogeneous distributed transactions: The participants involve two or more different technologies (e.g., databases from different vendors or non-database systems like message brokers). Ensuring atomic commit across these fundamentally different systems is much more challenging.

**Exactly-once message processing**

Heterogeneous transactions allow diverse systems to be integrated powerfully. For example, by atomically committing a message acknowledgment and the resulting database writes in a single transaction, we can ensure a message is effectively processed exactly once. If either the delivery or the database transaction fails, both are aborted, discarding any side effects and allowing the message broker to safely redeliver the message later. This is only possible, however, if all affected systems support the same atomic commit protocol.

**XA transactions**

X/Open XA is a standard introduced in 1991 for implementing two-phase commit across heterogeneous technologies. XA is not a network protocol; it is merely a C API for interfacing with a transaction coordinator.

The transaction coordinator implements the XA API and is often simply a library loaded into the same application process requesting the transaction. It tracks participants and uses a log on the local disk to record commit/abort decisions. If the application process crashes, the coordinator goes down with it. Any participants with prepared but uncommitted transactions are left stuck in doubt until the application server is restarted and the coordinator library reads the log to recover the outcomes.

**Holding locks while in doubt**

Being stuck in doubt is a severe problem because of database locking. To prevent dirty writes or ensure serializability, database transactions take row-level exclusive or shared locks.

A database cannot release these locks until the transaction definitively commits or aborts. Therefore, a transaction must hold onto these locks for the entire time it is in doubt. If the coordinator crashes and takes minutes to recover—or forever, if the logs are lost—those locks remain held. This blocks other transactions from modifying or reading that data, which can cause large parts of the application to become unavailable.

**Recovering from coordinator failure**

While a restarted coordinator should cleanly recover its state from the log, in practice, orphaned in-doubt transactions occur (e.g., due to lost or corrupted logs). These cannot be resolved automatically. Even rebooting the database servers will not fix it, as a correct 2PC implementation must preserve the locks of an in-doubt transaction even across restarts.

The only resolution is manual intervention by an administrator to decide the outcome. Many XA implementations offer an emergency escape hatch called heuristic decisions, which allows a participant to unilaterally decide to abort or commit without the coordinator. This is essentially a euphemism for probably breaking atomicity and is intended only for catastrophic situations.

**Limitations of distributed transactions**

XA transactions solve the problem of keeping several participant data systems consistent, but they introduce major operational problems. Because the transaction coordinator is itself a kind of database storing transaction outcomes, it comes with strict limitations:

- Single Point of Failure: If the coordinator runs only on a single machine, its failure causes other servers to block on locks held by in-doubt transactions.
- Breaks Statelessness: Server-side applications are traditionally stateless, but embedding a coordinator makes its local disk logs a crucial part of the durable system state.
- Lowest Common Denominator: Because XA must be compatible with wide-ranging systems, it cannot detect cross-system deadlocks or work with advanced protocols like Serializable Snapshot Isolation (SSI).
- Amplifying Failures: Because 2PC requires every participant to respond, if any single part of the system is broken, the entire transaction fails.

<br>

---

<br>

### Fault-Tolerant Consensus

Informally, consensus means getting several nodes to agree on something. The consensus problem is normally formalized as follows: one or more nodes may propose values, and the consensus algorithm decides on one of those values.

A consensus algorithm must satisfy the following properties:

- Uniform agreement: No two nodes decide differently.
- Integrity: No node decides twice.
- Validity: If a node decides value `v`, then `v` was proposed by some node.
- Termination: Every node that does not crash eventually decides some value.

The uniform agreement and integrity properties define the core idea of consensus: everyone decides on the same outcome, and once you have decided, you cannot change your mind. The validity property exists mostly to rule out trivial solutions.

If you don’t care about fault tolerance, satisfying the first three properties is easy: you can just hardcode one node to be the "dictator." However, if that one node fails, the system can no longer make decisions—which is what happens in two-phase commit (2PC) when the coordinator fails.

The termination property formalizes the idea of fault tolerance, ensuring the algorithm makes progress. The system model of consensus assumes a crash-stop model, where a node suddenly disappears and never comes back. Any consensus algorithm requires at least a majority of nodes to be functioning correctly to assure termination. Thus, termination is subject to the assumption that fewer than half of the nodes are crashed, though safety properties are always met even if a majority fails. Most algorithms also assume there are no Byzantine faults.

**Consensus Algorithms and Total Order Broadcast**

The best-known fault-tolerant consensus algorithms are Viewstamped Replication (VSR), Paxos, Raft, and Zab. Most of these algorithms don't directly decide on a single value; instead, they decide on a sequence of values, which makes them total order broadcast algorithms.

Total order broadcast requires messages to be delivered exactly once, in the same order, to all nodes. This is equivalent to repeated rounds of consensus:

- Due to the agreement property, all nodes decide to deliver the same messages in the same order.
- Due to the integrity property, messages are not duplicated.
- Due to the validity property, messages are not fabricated.
- Due to the termination property, messages are not lost.

Single-leader replication takes all writes to the leader and applies them to followers in the same order. If the leader is manually chosen, it doesn't satisfy termination. Some databases perform automatic leader election, bringing them closer to solving consensus. However, to avoid split brain, we need consensus to elect a leader. This creates a conundrum: to solve consensus, we must first solve consensus.

**Epoch Numbering and Quorums**

To break out of this conundrum, consensus protocols internally use a leader but weaken the uniqueness guarantee: they define an epoch number (ballot number, view number, or term number) and guarantee that within each epoch, the leader is unique.

Every time the current leader is thought to be dead, an election starts with an incremented epoch number. If there is a conflict between two different leaders, the leader with the higher epoch number prevails.

Before a leader decides anything, it must collect votes from a quorum of nodes. A node votes in favor of a proposal only if it is not aware of any other leader with a higher epoch. This requires two overlapping rounds of voting: once to choose a leader, and a second time to vote on a proposal. If a vote on a proposal succeeds, the overlap ensures the current leader hasn't been ousted by a higher-numbered epoch.

**Limitations of Consensus**

Consensus algorithms are a huge breakthrough: they bring concrete safety properties and fault tolerance to distributed systems, providing total order broadcast and linearizable atomic operations. Nevertheless, the benefits come at a cost:

- Performance: The voting process is a kind of synchronous replication, which many avoid in favor of the better performance of asynchronous replication.
- Strict Majority: They require a strict majority to operate (e.g., three nodes to tolerate one failure). If a network failure cuts off a minority, that portion is blocked.
- Static Membership: Most assume a fixed set of nodes; dynamic membership extensions are much less well understood.
- Timeout Sensitivity: They rely on timeouts to detect failures. In environments with highly variable network delays, this can cause frequent, wasteful leader elections.
- Network Edge Cases: They can be particularly sensitive to network problems, sometimes causing leadership to continually bounce so the system never makes progress.

<br>

---

<br>

### Membership and Coordination Services

Projects like ZooKeeper or etcd have APIs that look like databases (reading and writing keys), but they are actually coordination and configuration services. They are rarely used directly as general-purpose databases; instead, systems like HBase and Kafka rely on them indirectly in the background.

They are designed to hold small amounts of data entirely in memory (while writing to disk for durability). This data is replicated across all nodes using a fault-tolerant total order broadcast algorithm, which keeps replicas consistent by applying writes in the exact same order.

Modeled after Google’s Chubby lock service, ZooKeeper implements consensus alongside a specific set of features that are highly useful for building distributed systems:

- Linearizable atomic operations: Allows the implementation of distributed locks using atomic compare-and-set operations. These locks are usually implemented as leases with an expiry time to handle client failures.
- Total ordering of operations: Provides fencing tokens (monotonically increasing transaction IDs and version numbers) to prevent client conflicts in the event of process pauses.
- Failure detection: Clients maintain long-lived sessions with the servers via periodic heartbeats. If heartbeats cease and the session times out, ZooKeeper can automatically release any locks held by that session (using ephemeral nodes).
- Change notifications: Clients can "watch" locks and values for changes (such as another client joining or failing) without having to frequently poll the service.

While only the linearizable atomic operations strictly require consensus, it is the combination of these features that makes ZooKeeper so useful for distributed coordination.

**Allocating Work to Nodes**

This model is heavily used for tasks like electing a leader among service instances or deciding which partition to assign to which node (and rebalancing them when nodes join or fail). By correctly using atomic operations, ephemeral nodes, and notifications, applications can automatically recover from faults without human intervention.

Performing majority votes across thousands of application nodes would be terribly inefficient. Instead, ZooKeeper runs on a fixed, small number of nodes (usually three or five) to handle the votes while supporting a massive number of clients. It allows large clusters to effectively "outsource" the hard work of coordination, consensus, and failure detection.

The data managed by ZooKeeper is intended to be slow-changing (e.g., leadership assignments that change over minutes or hours). It is not meant for storing high-frequency runtime application state.

**Service Discovery**

ZooKeeper and etcd are also used for service discovery—registering and finding the routing endpoints of virtual machines that continually come and go.

Strictly speaking, service discovery does not require consensus. Traditional DNS handles this using multiple layers of caching, where reads are absolutely not linearizable, but high availability is prioritized over strictly fresh data. However, because leader election does require consensus, it makes sense to use the consensus system to help services discover who the leader is. To support this without overloading the voting nodes, consensus systems often use read-only caching replicas to serve non-linearizable read requests.

**Membership Services**

A membership service determines which nodes are currently active cluster members. Because unbounded network delays make reliable failure detection impossible on its own, coupling failure detection with consensus allows nodes to come to an agreement about who is alive.

Even if a node is incorrectly declared dead by consensus, having a cluster-wide agreement on the current membership is absolutely essential. Without this agreement, different nodes would have divergent opinions on the membership state, making basic tasks—like simply picking the lowest-numbered member to be the leader—impossible.
