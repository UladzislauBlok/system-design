## DB ACID

### What is ACID?

ACID stands for four key properties that ensure data validity despite errors, power failures, or concurrent access: Atomicity, Consistency, Isolation, and Durability.

<br>

##### Atomicity

Definition: All operations within a transaction must fully succeed, or none of them take effect. It's an "all-or-nothing" rule.

- Your Idea: All writes are going to succeed or none of them.
- Example: Consider a money transfer between two accounts. If $10 is deducted from Account A but the system fails before adding $10 to Account B, the transaction is aborted entirely. The money must not be lost or created. The database state reverts as if the transaction never started.

<br>

##### Consistency

Definition: A transaction brings the database from one valid state to another. Any data written to the database must be valid according to all defined rules and constraints (invariants).

- Your Idea: All fails are graceful and don't break invariants.
- Example: If a rule (an invariant) dictates that there must always be an officer on duty:
  - Scenario: You attempt to replace Officer 1 with Officer 2.
  - If the database removes Officer 1 but fails before writing Officer 2, the entire transaction must be rolled back. The database remains in a consistent state where Officer 1 is still on duty, thus preserving the invariant.

<br>

##### Isolation

Definition: Concurrent transactions should execute as if they were running sequentially. The intermediate state of one transaction is invisible to other transactions.

- Your Idea: This means there is no race conditions, all writes are executed simultaneously and don't affect each other or override the results of others.
- Example: Imagine two separate threads attempting to update the same row (e.g., updating a product's stock count). Isolation guarantees that:
  - One thread's write does not override the result of the other's update unintentionally (known as the Lost Update problem).
  - Each thread has a consistent visibility guarantee of the data, preventing common issues like Dirty Reads or Phantom Reads.

<br>

##### Durability

Definition: Once a transaction has been successfully committed, the changes are permanent and will survive any subsequent system failures, such as a power outage or crash.

- Your Idea: Committed writes are not getting lost.
- Example: Immediately after a bank transfer transaction is committed, a power supply failure causes the database to shut down. Upon restarting, the system must recover and ensure the transfer is permanently recorded, reflecting all committed results.

<br>

##### Basic Techniques for Achieving ACID

The basic techniques used to achieve these ACID properties include:

- WAL (Write-Ahead Logging): Essential for _Durability_ and _Atomicity_. Before any changes are applied to the main data files, they are first recorded in a sequential log.
- Transactions: The mechanism that groups operations to enforce _Atomicity_ and _Consistency_.
- Synchronization/Locking/Multi-Version Concurrency Control (MVCC): Techniques used to ensure _Isolation_.

| Property    | What it Guarantees                                                                                        | Key Concept             | Why it Matters                                                                             |
| ----------- | --------------------------------------------------------------------------------------------------------- | ----------------------- | ------------------------------------------------------------------------------------------ |
| Atomicity   | All operations in a transaction either succeed completely or fail completely ("All or nothing")           | Transactional Integrity | Prevents partial updates (e.g., ensures money is never deducted without being credited)    |
| Consistency | Transactions move the database from one valid state to another, upholding all defined rules (invariants)  | Validity of Data        | Ensures the data conforms to the schema and business logic constraints                     |
| Isolation   | Concurrent transactions execute without interfering with each other, as if run sequentially               | Concurrency Control     | Protects against race conditions and lost updates when multiple users access the same data |
| Durability  | Once a transaction is committed, the changes are permanent and survive system failures (e.g., power loss) | Persistence of Data     | Guarantees that committed data is safely stored on stable storage                          |

<br>

---

<br>

### Database Concurrency and Isolation Levels

Most modern databases are multithreaded. When multiple threads perform reads and writes simultaneously, they can interfere with one another, leading to race conditions.

- Transaction: A group of operations treated as a single unit.
- Commit: The signal from the database that all operations in a transaction finished successfully and are now permanent.
- Invariance: A rule or condition that must always remain true (e.g., "The total sum of money in all accounts must remain constant during a transfer").

### Common Isolation Issues and Fixes

**1. Dirty Writes**

_Definition_: Occurs when a transaction overwrites a value that has been written by another transaction that hasn't committed yet.
_Example_: Two threads (T1, T2) try to update purchase and delivery tables for the same item.

- T1: Updates purchase (Name: Tom).
- T2: Updates purchase (Name: Joy).
- T2: Updates delivery (Address: Joy’s).
- T1: Updates delivery (Address: Tom’s).
  _Result_: Data corruption. The purchase says "Joy" but the delivery says "Tom."
  _Fix_: Row-level locking. Ensure threads acquire locks in the same order to prevent deadlocks.

**2. Dirty Reads**

_Definition_: A transaction reads data that has been written by another transaction but not yet committed.

_Example_: T1 adds $100 to an account. T2 reads the new balance. T1 then hits an error and performs a rollback. T2 is now working with "ghost" money that doesn't exist.
_Fix_: - Locking: Prevent reads on rows being updated (slow). - Read Committed: The DB keeps the "old" value and the "new" uncommitted value simultaneously. Readers are given the old version until the writer commits.

**3. Read Skew (Non-repeatable Read)**

Definition: A transaction reads different data at different points in time because another transaction committed a change in between.

_Example_: You are calculating the total balance of a family ($1M). You read Account A, then T2 transfers money from Account B to Account A. When you finally read Account B, the money is gone from there, but you didn't see it in your earlier read of Account A.
_Fix_: Snapshot Isolation. The DB uses a Write-Ahead Log (WAL) with monotonically increasing sequence numbers. Each transaction sees a consistent "snapshot" of the DB as it existed at the moment the transaction started.

**4. Lost Updates**

Definition: Two transactions read a value, calculate a new value based on it, and try to write it back. The second write overwrites the first, losing the first update entirely.

_Example_: T1 and T2 both read balance $100. T1 adds $50 (writes $150). T2 adds $10 (writes $110). T1's update is lost.

_The Fix_:

- Atomic writes: `UPDATE table SET val = val + 10`.
- Explicit Locking: `SELECT ... FOR UPDATE`.
- CAS (Compare-and-Swap): Only write if the value is still what you originally read.

**5. Write Skew**

Definition: A generalization of the Lost Update problem. Two transactions read the same data, but then update different objects. The logic of the update depends on the data read, but once the update happens, the premise of the other transaction is no longer true.

_The Fix_:

- Explicit Row Locking: If the rows you are checking exist, you must lock all of them (e.g., `SELECT * FROM doctors WHERE on_call = true FOR UPDATE`). This prevents other threads from changing their status until you are done.

**6. Phantoms (The "Missing Row" Problem)**

Definition: A phantom occurs when a write in one transaction changes the result of a search query in another transaction. You can't lock a row that doesn't exist yet!

_The Fix_:

- Materializing Conflicts:
  - If you are building a meeting room booking system, you can’t lock a "reservation" that isn't there.
  - Strategy: You create a "Time_Slots" table with every possible hour/room combination. Now, instead of checking if a reservation exists, you try to lock the specific Time_Slot row. You have turned a phantom (a missing reservation) into a concrete conflict on a physical row.

### Performance-Oriented Approaches

**Serial Execution**

Used in systems like VoltDB or Redis, this achieves isolation by literally running only one transaction at a time on a single CPU core.

- Why it works: It avoids the overhead of locking.
- Requirement: Transactions must be stored in RAM (disk is too slow) and executed as Stored Procedures to avoid network round-trip delays.
- Trade-off: Stored procedures are harder to debug, version control, and integrate into CI/CD.

**Two-Phase Locking (2PL)**

A pessimistic approach where readers don't just block writers, but writers also block readers.

- Shared Lock: Many can read.
- Exclusive Lock: Only one can write (and no one can read).
- Predicate Locks: Locks a condition (e.g., `WHERE room_id = 5`) rather than specific rows. It prevents phantoms by blocking any INSERT or UPDATE that would match that condition.
- Index Range Locking: A simplified version of predicate locking that locks a range in the index (e.g., all IDs between 100 and 200). It's faster to manage but may lock more than necessary.

**Serializable Snapshot Isolation (SSI)**

An optimistic approach. It allows transactions to proceed without locks, but monitors them for conflicts.

- Case 1 (Reading stale data): T1 updates a row. T2 reads the old version but tracks that T1 is active. If T1 commits and T2 then tries to commit, the DB forces T2 to abort because its premise (the old data) is now invalid.
- Case 2 (Checking for changes): The DB keeps track of which rows were read. Before committing, it checks if any of those rows changed since the read. If yes, it aborts and restarts.
