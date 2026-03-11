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

<br>

---

<br>

### Timeouts and Unbounded Delays

<br>

---

<br>

### Synchronous Versus Asynchronous Networks
