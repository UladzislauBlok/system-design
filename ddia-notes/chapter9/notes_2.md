# Chapter 9.2 Consistency and Consensus: Ordering Guarantees

A linearizable register behaves as if there is only a single copy of the data, with every operation taking effect atomically at one point in time. This implies that operations execute in a well-defined order. Ordering is a fundamental idea in distributed systems, as seen in several contexts:

- Single-leader replication: The leader's main purpose is to determine the order of writes in the replication log to prevent conflicts from concurrent operations.
- Serializability: Ensures that transactions behave as if they were executed in some sequential order, either by executing them serially or by preventing serialization conflicts.
- Timestamps and clocks: An attempt to introduce order into a disorderly world, such as determining which of two writes happened later.

There are deep connections between ordering, linearizability, and consensus. Understanding these theoretical concepts is very helpful for clarifying what systems can and cannot do.

<br>

---

<br>

### Ordering and Causality

Ordering is crucial because it helps preserve causality. Causality imposes an ordering on events (cause comes before effect) and defines the causal order in a system—what happened before what. If a system obeys the ordering imposed by causality, it is considered causally consistent.

Causality has been important in several examples:

- Consistent Prefix Reads: Seeing the answer to a question before the question itself violates our intuition of cause and effect. There is a causal dependency between the question and the answer.
- Replication delays: From a replica's perspective, it might look like a row was updated before it existed. Causality means a row must first be created before it can be updated.
- Concurrent writes: If operation A happened before B, B might have depended on A. If A and B are concurrent, there is no causal link between them.
- Snapshot isolation: Reading from a consistent snapshot means it is consistent with causality. If a snapshot contains an answer, it must contain the question. Read skew means reading data in a state that violates causality.
- Write skew: Going off-call is causally dependent on the observation of who is currently on call. Serializable snapshot isolation detects write skew by tracking these dependencies.
- Cross-channel timing: Hearing someone exclaim a football score means their exclamation is causally dependent on the announcement of the score, so you should also be able to see the score.

**The Causal Order is Not a Total Order**

A total order allows any two elements to be compared (e.g., natural numbers where 13 is always greater than 5). A partial order means some elements are ordered, but others are incomparable (e.g., mathematical sets where neither {a, b} nor {b, c} is a subset of the other).

- Linearizability (Total Order): There are no concurrent operations. Every request is handled atomically along a single, straight-line timeline.
- Causality (Partial Order): Operations are ordered if they are causally related, but incomparable if they are concurrent. Concurrency means the timeline branches and merges again, very much like the version histories in distributed version control systems like Git.

**Linearizability vs. Causal Consistency**

Linearizability implies causality: any linearizable system automatically preserves causality without needing to pass around timestamps. However, making a system linearizable can severely harm its performance and availability, especially with significant network delays.

- Causal consistency is a middle ground. It is the strongest possible consistency model that does not slow down due to network delays and remains available during network failures.
- Many systems that appear to require linearizability actually only require causal consistency.

**Capturing Causal Dependencies**

To maintain causality, a system must know which operation happened before another.

- When a replica processes an operation, it must ensure all causally preceding operations have already been processed. If not, it must wait.
- Determining causal dependencies requires describing the "knowledge" of a node (e.g., Did the node know about value X when it wrote Y?).
- Causal consistency tracks dependencies across the entire database, not just for a single key. This is often done by generalizing version vectors and keeping track of which version of the data was read by the application.

<br>

---

<br>

### Sequence Number Ordering

Tracking all causal dependencies can become impractical due to the large overhead of explicitly tracking all the data that a client has read before writing. A more efficient approach is to use sequence numbers or timestamps from a logical clock to order events.

Logical clocks generate compact sequence numbers (typically using counters incremented for every operation) that provide a total order. This total order can be made consistent with causality: if operation A causally happened before B, A will have a lower sequence number than B. While this captures all causality information, it imposes more ordering than causality strictly requires (as concurrent operations are ordered arbitrarily).

In single-leader replication, the replication log defines a causal total order. The leader simply increments a counter for each operation, assigning monotonically increasing sequence numbers. Followers applying these writes in order remain causally consistent.

**Noncausal Sequence Number Generators**

In multi-leader or leaderless databases, generating sequence numbers is more complex. Common, scalable methods include:

- Each node generating independent sequence numbers (e.g., interleaving odd/even numbers or reserving bits for a node identifier).
- Attaching high-resolution timestamps from a physical time-of-day clock.
- Preallocating blocks of sequence numbers to different nodes (e.g., Node A gets 1-1000, Node B gets 1001-2000).

These methods perform better than routing all operations through a single leader counter. However, they are not consistent with causality because they do not correctly capture the ordering of operations across different nodes:

- Independent node counters may lag behind one another (an odd-numbered operation's true causal order relative to an even-numbered one is unknown).
- Physical clocks are subject to clock skew, potentially assigning lower timestamps to causally later operations.
- Block allocators can assign higher sequence numbers to causally earlier operations processed by a different node.

**Lamport Timestamps**

There is a simple method for generating sequence numbers consistent with causality: the Lamport timestamp.

![lamport_timestamps](./images/lamport_timestamps.png)

- Each node keeps a counter of the operations it has processed.
- A Lamport timestamp is a pair: `(counter, node ID)`. The node ID ensures uniqueness.
- It provides total ordering: compare the counter values first; if they are the same, the greater node ID is the greater timestamp.

The key to making Lamport timestamps consistent with causality is that every node and client tracks the maximum counter value seen so far and includes it on every request. When a node receives a value greater than its own counter, it immediately increases its counter to that maximum.

For example, if a client receives a counter value of 5 from Node 2 and sends it to Node 1 (which only had a counter of 1), Node 1 immediately moves its counter to 5. Its next operation will have a counter value of 6. Carrying the maximum counter ensures causal dependencies result in increased timestamps.

_Note: Lamport timestamps enforce total ordering but cannot distinguish between concurrent and causally dependent operations. Version vectors can distinguish them but are less compact._

Timestamp Ordering is Not Sufficient
While Lamport timestamps define a causal total order, they cannot solve all distributed system problems, such as enforcing a unique username constraint.

If two users concurrently try to create the same username, you could pick the one with the lower timestamp as the winner. This works _after the fact_ once all operations are collected.

However, when a node receives the request, it needs to decide immediately if it succeeds or fails. It doesn't know if another node is concurrently processing the same username with a lower timestamp. To be sure, it would have to check with every other node, meaning a single node failure would grind the system to a halt.

The total order of operations only emerges after collecting all operations. To implement uniqueness constraints, a total ordering is not enough; a system also needs to know when that order is finalized (meaning no other node can insert an operation ahead of yours). This requirement leads to the concept of total order broadcast.

<br>

---

<br>

### Total Order Broadcast

Getting all nodes in a distributed system to agree on a total ordering of operations is difficult. Timestamp or sequence number ordering is not powerful enough if you need fault tolerance (like implementing a uniqueness constraint without a single point of failure).

Single-leader replication handles this by sequencing all operations on one CPU core, but scaling throughput and handling leader failover require a solution known in distributed systems literature as total order broadcast (or atomic broadcast).

_Note: Partitioned databases with a single leader per partition often only maintain ordering per partition, meaning they cannot offer cross-partition consistency guarantees without additional coordination._

Total order broadcast is a protocol for exchanging messages between nodes that must always satisfy two safety properties, even during network or node faults:

- Reliable delivery: No messages are lost. If a message is delivered to one node, it is delivered to all nodes.
- Totally ordered delivery: Messages are delivered to every node in the exact same order.

**Using Total Order Broadcast**

Consensus services like ZooKeeper and etcd implement total order broadcast, hinting at the strong connection between broadcast and consensus.

Total order broadcast is essential for:

- Database replication: If every message is a write, and replicas process them in the same order, they remain consistent (State Machine Replication).
- Serializable transactions: If messages represent deterministic transactions executed as stored procedures, processing them in order keeps partitions consistent.
- Lock services: Appending lock requests to a log creates sequentially numbered messages. These monotonic sequence numbers act as fencing tokens (called `zxid` in ZooKeeper).

A critical feature of total order broadcast is that the order is fixed when messages are delivered. A node cannot retroactively insert a message into an earlier position once subsequent messages are delivered. This makes it stronger than timestamp ordering and allows it to function as an append-only log that all nodes read identically.

**Implementing Linearizable Storage Using Total Order Broadcast**

Total order broadcast and linearizability are not the same. Total order broadcast is asynchronous (reliable, ordered delivery, but no guarantee when it arrives), whereas linearizability is a recency guarantee (reads see the latest written value).

However, you can build linearizable storage (like a compare-and-set register to ensure unique usernames) on top of total order broadcast acting as an append-only log:

- Append a message to the log tentatively claiming the username.
- Read the log and wait for your appended message to be delivered back to you.
- Check for any messages claiming the same username. If your message is the first one, you succeed and commit the claim. If another user's message is first, you abort.

Because all nodes see log entries in the same order, they all agree on which concurrent write came first.

While this ensures linearizable _writes_, it does not guarantee linearizable _reads_ (reading from an asynchronously updated store might yield stale data, providing only sequential/timeline consistency). To make reads linearizable, you can:

- Sequence reads through the log (append a read message and perform the read when it is delivered back).
- Query the position of the latest log message in a linearizable way, wait for all entries up to that position to be delivered, and then read.
- Read from a synchronously updated replica.

**Implementing Total Order Broadcast Using Linearizable Storage**

Conversely, you can build total order broadcast if you assume you have linearizable storage (like an atomic increment-and-get or compare-and-set register).

The algorithm:

- For every message to broadcast, you increment-and-get the linearizable integer.
- Attach the resulting value to the message as a sequence number.
- Send the message to all nodes.
- Recipients deliver the messages consecutively by sequence number.

Unlike Lamport timestamps, this linearizable register generates a sequence with no gaps. If a node delivers message 4 and receives message 6, it knows it must wait for message 5. This is the key difference between total order broadcast and timestamp ordering.

Creating this fault-tolerant linearizable integer requires handling network interruptions and node failures. Ultimately, attempting to build a linearizable sequence number generator inevitably leads to a consensus algorithm.

It is a profound insight in distributed systems that a linearizable compare-and-set (or increment-and-get) register and total order broadcast are equivalent to consensus. Solving one provides the solution to the others.
