# Chapter 3.1 Storage and Retrieval: Data Structures That Power Your Database

The most basic key-value store functions by appending every new write to the end of a text file (a log).

- db_set (Writes): Extremely efficient. Because it only appends to the end of a file, it avoids the complexity of overwriting data.
- db*get (Reads): Extremely inefficient. To find the latest value for a key, the database must scan the entire file from start to finish. This results in \*\*O(\_n*)\*\* complexity, meaning search time grows linearly with the amount of data.

```
#!/bin/bash

db_set () {
  echo "$1,$2" >> database
}

db_get () {
  grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

To solve the performance issues of reads, databases use indexes. An index is "side-metadata" that acts as a signpost to help locate specific data without scanning the whole file.

Key Characteristics of Indexes:

- Derived Data: An index is a separate structure built from the primary data. Removing it doesn't delete your data; it only slows down your queries.
- Manual Selection: Because indexes aren't "free," developers must manually choose which fields to index based on the application's specific query patterns.

Storage systems face a fundamental balancing act regarding performance:

| Operation | Without Index (Log-structured)        | With Index                                      |
| --------- | ------------------------------------- | ----------------------------------------------- |
| Writes    | Fast: Simple append operation.        | Slower: Must update both the log and the index. |
| Reads     | Slow: Requires a full file scan O(n). | Fast: Uses the index to jump to the data.       |

<br>

---

<br>

### Hash Indexes (In-Memory)

The simplest indexing strategy for key-value stores is to keep a hash map in RAM.

- How it works: Each key in the hash map points to a byte offset in the data file on disk.
- Writes: When you append data to the log, you update the hash map with the new offset.
- Reads: To find a value, you look up the key in the hash map, jump to the specific offset on disk (a "disk seek"), and read the value.
- Constraint: This approach requires all keys to fit in RAM. However, the values can be much larger than RAM because they stay on disk until needed.

![hash_map](./hash_map.png)

Since an append-only log grows indefinitely, the database must reclaim space. This is handled through Segmentation and Compaction.

The log is broken into segments of a fixed size. When a segment reaches its limit, it is closed ("frozen"), and a new segment is started for subsequent writes.

Because many writes might update the same key, we only care about the latest version. Compaction processes these frozen segments by discarding old values and keeping only the most recent entry for each unique key.

During compaction, multiple segments can be merged into a single, smaller segment.

- Background Processing: Merging happens in a background thread, so it doesn't block incoming reads or writes.
- Switching: Once a new merged segment is ready, the system swaps the old segments for the new one and deletes the old files.

![compaction](./segment_compaction.png)

When the database is split into multiple segments, the lookup process changes slightly:

- Check the in-memory hash map of the most recent (active) segment.
- If the key isn't there, check the hash map of the next-most-recent segment.
- Continue until the key is found.

Because the merging process keeps the total number of segments low, these lookups remain very fast.

Transforming this simple idea into a robust database requires handling several real-world edge cases:

- Binary File Format: Instead of CSV, binary formats are used. They encode the length of a string followed by the raw bytes, making it faster to parse and removing the need for escaping characters.
- Deleting Records (Tombstones): Since the log is append-only, you can't "erase" data. Instead, you append a tombstone (a special deletion record). During the merging process, the tombstone tells the engine to discard all previous values for that key.
- Crash Recovery: Because hash maps are in RAM, they are lost during a crash. To avoid a slow full-file scan upon restart, systems like Bitcask store snapshots of the hash maps on disk for fast loading.
- Partial Writes: If a crash occurs mid-write, the log might contain corrupted data. Checksums are used to detect and ignore these broken records.
- Concurrency: To keep things simple, there is usually only one writer thread (since writes are sequential). However, because segments are immutable, multiple threads can read them simultaneously.

It might seem wasteful not to overwrite data in place, but append-only designs offer three major advantages:

- Speed: Sequential writes are significantly faster than random writes on both magnetic hard drives and SSDs.
- Safety: Crash recovery is simpler because you don't risk "splicing" old and new data together during an overwrite.
- Fragmentation: Merging segments prevents the data file from becoming fragmented over time.

However, the hash table index also has limitations:

- RAM Constraint: The index (every key) must fit in memory. Storing a hash map on disk is inefficient due to random I/O and collision complexity.
- No Range Queries: Because hash maps are unordered, you cannot efficiently scan a range of keys (e.g., key001 to key100); you must look up each key individually.

<br>

---

<br>

### SSTables and LSM-Trees
