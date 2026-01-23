# Chapter 7.1 Transactions: Weak Isolation Levels

Concurrency issues, or race conditions, arise when multiple transactions attempt to access or modify the same data simultaneously. These bugs are notoriously difficult to find because they are timing-dependent and often impossible to reproduce in testing. While serializable isolation theoretically eliminates these issues by making transactions appear to run one at a time, it carries a heavy performance penalty.

In practice, many databases use weak isolation levels. These provide a balance between performance and correctness by protecting against some, but not all, concurrency issues. Even systems labeled as ACID often default to these weaker levels, meaning developers must understand specific race conditions to prevent data corruption or financial loss.

<br>

---

<br>

### Read Committed

Read Committed is the most basic isolation level and is the default for many popular databases like PostgreSQL, Oracle, and SQL Server. It provides two primary guarantees:

- No Dirty Reads: A transaction will only see data that has been successfully committed.
- No Dirty Writes: A transaction will only overwrite data that has been successfully committed.

**Dirty Reads**

A dirty read occurs when a transaction sees uncommitted data from another ongoing transaction. Preventing this is crucial for two reasons:

- Inconsistency: A transaction might see a "partial" update where some objects are changed but others are not, leading to incorrect logic or user confusion.
- Rollbacks: If a transaction aborts, any data it wrote is discarded. Allowing dirty reads would mean a transaction could act on data that technically "never existed."

![dirty_read](./images/dirty_read.png)

**Dirty Writes**

A dirty write occurs when a later write overwrites a value that has not yet been committed by an earlier transaction. By delaying the second write until the first transaction finishes, the database prevents mishaps where multiple related objects end up in an inconsistent state (e.g., one buyer is listed on the car entry, but another buyer is listed on the invoice).

![dirty_write](./images/dirty_write.png)

- Note: Read committed does not prevent all race conditions, such as the "lost update" problem where two transactions concurrently increment the same counter.

**Implementation**

Most databases implement Read Committed using a combination of locking and versioning:

- Row-level locks: To prevent dirty writes, a transaction must acquire a lock on a specific object before modifying it. It holds this lock until it commits or aborts.
- Value Versioning: To prevent dirty reads without slowing down performance, databases typically avoid using read locks. Instead, they remember both the old committed value and the new value currently being written. Any concurrent readers are simply given the old committed value until the new one is officially committed.

<br>

---

<br>

### Snapshot Isolation and Repeatable Read

While Read Committed prevents dirty reads and writes, it remains vulnerable to read skew (also known as a nonrepeatable read). This occurs when a transaction reads different parts of the database at different times and sees inconsistent states because another transaction committed a change in between those reads.

![nonrepeatable_read](./images/nonrepeatable_read.png)

For example, if Alice transfers money between two accounts while she is viewing her balances, she might see one account before the transfer and the other after, making it appear as if money has vanished. While often temporary for a user, read skew is fatal for:

- Backups: A large backup takes hours; if the data is inconsistent, the restored database will be permanently corrupted.
- Analytics and Integrity Checks: Long-running queries that scan the database will return nonsensical results if the data changes mid-execution.

**Multi-Version Concurrency Control (MVCC)**

Snapshot Isolation solves read skew by ensuring each transaction sees a consistent snapshot of the database—frozen at the moment the transaction started. The guiding principle is that readers never block writers, and writers never block readers. This is implemented using Multi-Version Concurrency Control (MVCC), where the database maintains several committed versions of an object simultaneously.

In an MVCC system (like PostgreSQL):

- Transaction IDs (txid): Each transaction is assigned a unique, increasing ID.
- Version Tracking: Each row has a created_by field (the txid of the inserter) and a deleted_by field (initially empty).
- Immutable Updates: An update is treated as a deletion of the old version and an insertion of a new version.
- Garbage Collection: A background process eventually deletes rows marked for deletion once no active transactions can see them.

![multi_version_objects](./images/multi_version_objects.png)

**Visibility Rules**

To maintain the snapshot, the database applies strict visibility rules. A transaction can see an object only if:

- The transaction that created the object had already committed when the current transaction started.
- The object is not marked for deletion, or the transaction that deleted it had not yet committed when the current transaction started.

Everything else—including writes from transactions with a later ID or currently in-progress transactions—is ignored.

_Naming and the SQL Standard_

There is significant industry confusion regarding the name of this level. The SQL standard, defined in 1975, does not include "Snapshot Isolation." Consequently, many databases (PostgreSQL and MySQL) call their implementation of snapshot isolation Repeatable Read to claim standard compliance.

However, because the standard’s definition is famously ambiguous, "Repeatable Read" means different things in different databases. For instance, in Oracle, snapshot isolation is called "Serializable," while in DB2, "Repeatable Read" actually refers to true Serializability.

<br>

---

<br>

### Preventing Lost Updates
