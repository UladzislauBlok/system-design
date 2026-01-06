## Replication

### Introduction

Imagine you own a website with a single server and one database serving five users across Europe, the US, and Australia. If that database server fails, you face two primary risks: **Data Loss** and **Downtime**.

While a simple restart might solve a temporary crash, a corrupted disk could lead to permanent data loss. Even if the server is healthy, physical distance creates **latency issues**—an Australian user must wait for data to travel halfway around the planet. Furthermore, a single database has a **limited throughput** capacity; if traffic spikes, the database may become a bottleneck.

**Replication**—maintaining multiple copies of the same data across different nodes—addresses these challenges by providing:

- Redundancy: Data survives hardware faults.
- Increased Throughput: Multiple replicas can share the load of read requests.
- Reduced Latency: By placing replicas in different regions (US, Europe, APAC), users access data from a geographically closer node.

##### Synchronous vs. Asynchronous Replication

The choice between these two methods involves a fundamental trade-off between consistency and availability.

**1. Synchronous Replication**

In this model, the primary database receives a write request and writes it to the local disk. However, it does not confirm success to the client until the data is successfully written to the followers (replicas).

- Advantage: Provides Strong Consistency. Every replica is guaranteed to have the same version of data at any given time.
- Disadvantage: Higher latency, as the user must wait for network communication between nodes. If a follower is down or the network is slow, the entire write operation may hang or fail.
- Example: A banking system where a balance update must be identical across all nodes to prevent a user from withdrawing money twice.

**2. Asynchronous Replication**

The primary database writes to the local disk and immediately responds "Success" to the client. The followers pull or receive the data at a later point in time.

- Advantage: High performance and low latency. The system remains available for writes even if followers are lagging or disconnected.
- Disadvantage: Provides Eventual Consistency. There is a "replication lag" where a client might read stale values (old data) from a follower that hasn't updated yet.
- Example: A social media "Like" count. If a user sees 99 likes instead of 100 for a few seconds, it doesn't break the application's core functionality

##### Replication Methods (How to Replicate)

There are three common technical approaches to moving data from the primary node to the replicas:

**1. Statement-Based Replication**

The primary node logs every write request (e.g., SQL INSERT or UPDATE statements) and sends those commands to the replicas to be re-executed.

- Issue: Non-deterministic functions. Commands like `NOW()` or `RAND()` will produce different results on different servers, leading to data divergence between the primary and the replicas.

**2. Write-Ahead Log (WAL) Shipping**

The database usually writes all changes to a low-level log (the WAL) before applying them to data files. This log is sent directly to replicas.

- Issue: Version Coupling. The WAL is often very specific to the database engine's storage internal format. This makes it difficult to perform "Zero Downtime" upgrades, as all nodes must run the exact same version of the database software. You also cannot replicate between different engines (e.g., from PostgreSQL to MySQL).

**3. Logical Log Replication (Row-Based)**

This uses a "logical" log that is decoupled from the storage engine's internals. It describes changes at the row level (e.g., "Row ID 5 in Table X was changed from A to B").

- Advantage: Because it is decoupled from the physical storage format, it allows for easier database version upgrades and better compatibility between different systems.
