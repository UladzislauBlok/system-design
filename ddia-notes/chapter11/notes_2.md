# Chapter 11.2 Stream Processing: Databases and Streams

Many data systems need to be kept in sync, such as keeping a search index or a cache updated with changes from a relational database. Streams offer a powerful way to manage these relationships

<br>

---

<br>

### Keeping Systems in Sync

When you have a primary database and secondary systems (like search indexes or caches), they must be kept in sync. This is typically done through **dual-writes** or **change data capture**.

- **Dual-Writes**: The application writes to both systems simultaneously. This can lead to race conditions and consistency issues.
- **Log-Based Replication**: A cleaner approach where the database's replication log is used as a stream to update the secondary systems.

<br>

---

<br>

### Change Data Capture (CDC)

CDC is the process of extracting data from a database's write-ahead log and sending it to a stream for processing.

- **Implementation**: CDC systems often use database triggers or parse the replication log.
- **Benefits**: It provides a reliable, ordered stream of all database modifications.
- **Initial Snapshots**: Often, a CDC system needs an initial snapshot of the database to start from.

<br>

---

<br>

### Event Sourcing

A different approach where the application's state is not stored as current values but as a sequence of immutable events.

- **Event Store**: All events are saved to an event store (an append-only log).
- **Derived State**: Current state is derived by replaying the events.
- **Benefits**: Provides a complete audit trail and allows for time-traveling through state history.

<br>

---

<br>

### State, Streams, and Immutability

Immutability is a core theme in stream processing. Treating data as a sequence of immutable events makes systems easier to reason about and recover from failures.

- **Stateless vs. Stateful Processing**: Some stream processing is stateless (e.g., simple filtering), while others require state (e.g., windowed aggregations).
- **Materialized Views**: A database can be thought of as a materialized view of its change log.
- **Advantages of Immutability**: Easier recovery, auditability, and better scalability for analytical workloads.
