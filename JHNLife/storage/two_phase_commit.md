## Two-Phase Commit (2PC)

2PC is a blocking protocol used to ensure atomicity across multiple nodes. It is essential when a write operation spans different partitions (e.g., cross-partition transactions or updating global secondary indexes) where a partial failure would lead to data inconsistency.

### The Two Phases

**Phase 1: The Prepare Phase (Voting)**

- Request: The coordinator sends a `PREPARE` message to all participants.
- Validation: Each node checks if it can safely commit (validates data, acquires necessary locks, and writes data to a temporary workspace).
- Vote:
    - If a node is ready, it responds `YES`. This is a binding promise to commit later.
    - If any node says `NO` or fails to respond, the coordinator sends an `ABORT` to everyone.

**Phase 2: The Commit Phase (Execution)**

- The Point of No Return: If all participants voted `YES`, the coordinator writes the `COMMIT` decision to its disk-based transaction log.
- Commit Command: The coordinator sends the `COMMIT` message to all nodes.
- Indefinite Retries: If a participant node is down or a network packet is lost during this phase, the coordinator must retry the commit indefinitely.
    - No Rollback: Once the coordinator decides to commit, it cannot change its mind.
    - Recovery: If a participant was down, it must commit the transaction as soon as it recovers, using the locks it held since Phase 1.

### Critical Problems with 2PC

- The "Orphaned" Lock Problem: If the coordinator crashes after a participant has voted `YES` but before receiving a `COMMIT`/`ABORT`, that participant is stuck (blocked). It cannot release its locks because it doesn't know the final outcome. This can lead to a system-wide deadlock.
- Availability vs. Consistency: 2PC prioritizes consistency. If the coordinator or any single participant is unavailable during the voting phase, the entire transaction fails, reducing the overall availability of the system.
- Performance Impact: Because nodes must hold locks across two round-trips (plus disk fsyncs), throughput is significantly lower than in local transactions.

### Example: The "Point of No Return"

- Scenario: A transfer between Partition A (User 1) and Partition B (User 2).
- Phase 1: Both A and B say `YES` and lock the respective accounts.
- Phase 2: The Coordinator logs `COMMIT`.
- Failure: Partition B's network goes down.
- Result: Partition A commits immediately. The Coordinator will continuously ping Partition B until it comes back online to force the commit. Partition B cannot roll back User 2's balance update because it already promised it could do it in Phase 1.
