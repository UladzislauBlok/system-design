# Chapter 7.3 Transactions: Serializability

Weak isolation levels (like Read Committed and Snapshot Isolation) are prone to race conditions (e.g., write skew, phantoms) that are hard to detect and test. Serializable Isolation is the strongest isolation level. It guarantees that parallel transaction execution produces the same result as if transactions ran serially (one at a time), preventing all possible race conditions.

Most databases implement serializability using one of three techniques: Actual Serial Execution, Two-Phase Locking (2PL), or Serializable Snapshot Isolation (SSI).

<br>

---

<br>

### Actual Serial Execution

This approach removes concurrency entirely by executing transactions one by one on a single thread. While this limits throughput to a single CPU core, it avoids the overhead of locking and conflict detection.

This single-threaded model became feasible around 2007 due to two factors:

- RAM Availability: Active datasets can often fit entirely in memory, eliminating slow disk I/O.
- OLTP Nature: Most write transactions are short and fast. (Long-running read-only queries can run separately on a consistent snapshot).

*Encapsulating Transactions in Stored Procedures*

Traditional interactive transactions (where an application sends queries one by one and waits for responses) are inefficient for single-threaded execution due to network latency. The database would spend most of its time idle, waiting for the next query.

To solve this, applications must submit the entire transaction code to the database ahead of time as a Stored Procedure. This allows the database to execute the code in memory immediately without network or disk I/O waits.

![stored_procedures](./images/stored_procedures.png)

While stored procedures historically had a bad reputation (ugly proprietary languages, difficult debugging/versioning), modern implementations (e.g., Redis using Lua, VoltDB using Java) use general-purpose languages.

Replication: Systems like VoltDB execute the same stored procedure on every replica rather than copying write data. This requires stored procedures to be deterministic (producing the exact same result on every node).

*PartitioningI

To scale beyond a single CPU core, data can be partitioned. If a transaction only reads/writes data within a single partition, it can run on that partition's specific thread independently. This allows throughput to scale linearly with the number of CPU cores.

However, Cross-Partition Transactions are significantly slower because they require coordination across all touched partitions to ensure serializability. Performance drops drastically if the data structure requires frequent cross-partition access (e.g., multiple secondary indexes).

*Summary of Constraints*

Serial execution is a viable implementation of serializability only under specific conditions:

- Memory Resident: The active dataset must fit in RAM.
- Short Transactions: Execution must be fast; one slow transaction blocks all others.
- Write Throughput: Limited to a single CPU core unless the data can be partitioned to avoid cross-partition coordination.
