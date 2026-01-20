## Partitioning

### Data Partitioning Strategies

When you scale out, the primary goal is to distribute data so that you maximize throughput (number of requests processed) while minimizing latency (time for a single request).

**Range-Based Partitioning**

Data is assigned to partitions based on a continuous range of keys (e.g., `A–D`, `E–H`).

- Pros:
  - Simple Implementation: Easy to understand and map keys to nodes.
  - Range Query Efficiency: Since related keys are stored together, queries like "Find all users with names starting with B" are very fast because they hit only one partition.
- Cons:
  - Hotspots (Data Skew): Popular ranges can lead to unfair load distribution.
  - Example: If you partition logs by date, the partition for "Today" will be hit with all writes, while the partition for "Last Month" remains idle.

**Hash-Based Partitioning**

A hash function is applied to the key, and the resulting hash value determines the partition (e.g., `hash(key) % number_of_nodes`, hash range).

- Pros:
  - Uniform Distribution: Effectively spreads data across all nodes, significantly reducing the risk of hotspots.
  - High Throughput: Balances both read and write traffic across the cluster.
- Cons:
  - No Data Locality: Range queries become expensive. Since sequential keys are hashed to random locations, a range query must often scan every single node.
  - Example: Partitioning a `Users` table by `UserID`. Even if IDs are sequential (`1`, `2`, `3`), they will be scattered across different physical servers.

<br>

---

<br>

### Secondary Indexing Strategies

In a sharded system, the data is partitioned by a Partition Key. However, if you need to query by another attribute (a secondary index), you face a design trade-off.

**Local Secondary Index (LSI)**

Each partition maintains its own index for the data it stores. The index is "local" to the node.

- Pros:
  - Fast Writes: When you update a row, you only need to update the index on the same node. This keeps write latency low and avoids distributed transactions.
- Cons:
  - Scatter-Gather Latency: To find an entity by the secondary index, the system must query every single node and merge the results.
  - The "Slowest Node" Problem: The total query time is determined by the slowest responding node (tail latency), which decreases overall availability and performance.
- Example: A `Products` table sharded by `BrandID`. If you have a local index on `Color`, searching for "Red" products requires asking every shard if they have any red items.

**Global Secondary Index (GSI)**

The index itself is partitioned and stored on different nodes from the actual data.

- Pros:
  - Efficient Reads: Reads are targeted. You query the specific node that holds the index range you need, allowing for low-latency searches.
- Cons:
  - Complex Writes: A single data write might require updating an index on a completely different node.
  - Consistency Challenges: Maintaining atomic updates between the data node and the index node is difficult. Often, GSIs are "eventually consistent" to avoid the heavy performance hit of distributed transactions.
- Example: An `Orders` table sharded by `OrderID`. A global index is created on `CustomerEmail`. When searching for orders by email, you go straight to the node holding that email's index.

<br>

### Comparison Summary

| Feature          | Range-Based   | Hash-Based     | Local Index            | Global Index     |
| ---------------- | ------------- | -------------- | ---------------------- | ---------------- |
| Primary Strength | Range Queries | Load Balancing | Write Performance      | Read Performance |
| Primary Weakness | Hotspots      | Range Scans    | Read Latency (Fan-out) | Write Complexity |
