## Column Oriented Storage

**1. Row-Oriented Storage (OLTP)**

This is the most intuitive way to conceptualize a database. Data is stored sequentially by record.

- Mechanism: If a database has 5 rows, it divides those rows into chunks and stores them in files. Each row is stored as a contiguous array of fields.
- Storage Layout:
  - Row 1: `[ID, Name, Email, Company]`
  - Row 2: `[ID, Name, Email, Company]`
  - Row 3: `[ID, Name, Email, Company]`
- Best Use Case: Online Transactional Processing (OLTP). Perfect for web applications where you need to load an entire user profile. Since all fields for a specific user are stored together, the database can perform a fast fetch with minimal disk seeks.

**2. Column-Oriented Storage (OLAP)**

Instead of storing rows together, all values for a specific column are stored together in separate files.

- Mechanism: In a table with `Name`, `Email`, and `Company`, the system takes every value from the `Name` field and stores them sequentially, then does the same for `Email`, and so on.
- Best Use Case: Data Science & Analytics (OLAP). Ideal for analyzing massive datasets (petabytes) where you need to calculate aggregates (MIN, MAX, AVG) on specific attributes across millions of records.

Key Advantages:

- I/O Efficiency: If a query only needs the Company column, the system only reads the Company file, ignoring all other data.
- CPU Cache Utilization: Because data is packed densely, more relevant information fits into the CPU cache, significantly speeding up processing.
- Compression Techniques:
  - Dictionary Compression: Useful for columns with a finite set of strings. We map distinct values to a small number of bits (e.g., representing 5 distinct values using only 3 bits: `000` to `100`).
  - Bitmap Indexing: Reducing size by representing the presence of values in a bit array. (example of bitmap indexing -> [ref](https://github.com/UladzislauBlok/system-design/blob/main/ddia-notes/chapter3/notes_3.md))
- Predicate Pushdown (e.g., Parquet): Data is divided into chunks (files) containing metadata (MIN, MAX, etc.). If a query looks for `Value > 60` and the file metadata shows `MAX = 59`, the system skips the entire file.

**3. Trade-offs and Challenges**

| Feature     | Row-Oriented                 | Column-Oriented                               |
| ----------- | ---------------------------- | --------------------------------------------- |
| Writes      | Fast (single location)       | Slow (must write to multiple files/locations) |
| Reads       | Fast for whole-row retrieval | Fast for aggregates/specific columns          |
| Compression | Poor (mixed data types)      | Excellent (uniform data types)                |

- Consistency Requirement: Every column file must maintain the exact same ordering of records so they can be re-combined correctly.
- Data Duplication: To speed up different types of queries, you might store multiple copies of the data with different sort orders (a trade-off of space for speed).
- The Write Problem: Updating column-oriented storage is expensive because one "row" write affects many files.
  - Solution: Use an LSM-Tree (Log-Structured Merge-Tree) approach. Accept writes into a row-oriented memory buffer (Memtable/SSTable) and, during the "compaction" or "flush" to disk, rebuild the data into the column-oriented format.

## Data Serialization Frameworks

Serialization is the process of encoding in-memory objects into a byte stream for storage or network transmission.

**1. The Problems with Text-Based Formats (JSON/XML)**

While human-readable and ubiquitous, formats like JSON and XML have significant overhead:

- Size Efficiency: They are verbose because they repeat field names (e.g., "name": "tom") in every single record.
- Data Precision: Issues often arise with floating-point numbers and large integers.
- Binary Data: Non-text data must be encoded (e.g., Base64), which increases file size by approximately 33%.
- No Schema Enforcement: Harder to guarantee type safety across different services.

**2. Tag-Based Binary Frameworks (Protocol Buffers & Thrift)**

These frameworks use a pre-defined IDL (Interface Definition Language) to describe the data structure.

- Mechanism: Instead of storing field names like "name" or "email," they use integer tags (e.g., 1, 2, 3).
- Efficiency: Significant savings in network bandwidth and storage. While there is a minor CPU penalty for encoding/decoding, the reduced I/O usually makes it a net win.
- Evolution (Backward/Forward Compatibility):
  - Forward Compatibility: Old code can read data written by new code because it simply ignores tags it doesn't recognize.
  - Backward Compatibility: New code can read data written by old code as long as new fields are marked as optional.

**3. Apache Avro (Dynamic Schema Evolution)**

Unlike Protobuf and Thrift, which require the schema to be known at compile-time (via generated code), Avro is designed for environments like Hadoop or Data Lakes where schemas change frequently.

- No Tags/IDs: Avro doesn't use field IDs. It relies on the writer’s schema and the reader’s schema.
- Schema Storage Strategies:
  - Data Lakes: The schema is often stored once at the beginning of a large file (like an Avro Object Container File).
  - Message Brokers (e.g., Kafka): A Schema Registry is used. The message header contains a schema ID/version. The reader fetches the specific schema from the registry to deserialize the bytes.
- Schema Resolution: If the reader's schema and the writer's schema don't match exactly, Avro performs "Schema Resolution" by matching fields by name and using default values for missing fields.

**Comparison Summary**

| Feature  | JSON/XML          | Protobuf/Thrift         | Apache Avro           |
| -------- | ----------------- | ----------------------- | --------------------- |
| Format   | Text              | Binary                  | Binary                |
| Schema   | Optional/External | Required (Compile-time) | Required (Run-time)   |
| Field    | Mapping By Name   | By Tag/ID               | By Name (via Schema)  |
| Use Case | Public APIs       | Internal Microservices  | Big Data / Data Lakes |
