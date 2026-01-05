# Chapter 1 Reliable, Scalable, and Maintainable Applications

**Topics:**

1. Leaders and Followers -> [ref](./notes_1.md)
   - Synchronous Versus Asynchronous Replication
   - Setting Up New Followers
   - Handling Node Outages
   - Implementation of Replication Logs

2. Problems with Replication Lag -> [ref](./notes_2.md)
   - Reading Your Own Writes
   - Monotonic Reads
   - Consistent Prefix Reads
   - Solutions for Replication Lag

3. Multi-Leader Replication -> [ref](./notes_3.md)
   - Use Cases for Multi-Leader Replication
   - Handling Write Conflicts
   - Multi-Leader Replication Topologies

4. Leaderless Replication -> [ref](./notes_4.md)
   - Writing to the Database When a Node Is Down
   - Limitations of Quorum Consistency
   - Sloppy Quorums and Hinted Handoff
   - Detecting Concurrent Writes
