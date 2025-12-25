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
