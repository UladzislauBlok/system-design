# Chapter 10.2 Batch Processing: MapReduce and Distributed Filesystems

MapReduce is like Unix tools but distributed across potentially thousands of machines — a blunt, brute-force, but surprisingly effective tool. A single MapReduce job is comparable to a single Unix process: it takes one or more inputs and produces one or more outputs. Like most Unix tools, it does not modify the input and has no side effects other than producing output files, which are written once in a sequential fashion.

While Unix tools use `stdin` and `stdout`, MapReduce jobs read and write files on a **distributed filesystem**. In Hadoop's implementation, that filesystem is called **HDFS** (Hadoop Distributed File System), an open source reimplementation of the Google File System (GFS). Other distributed filesystems exist (GlusterFS, Quantcast File System), and object storage services (Amazon S3, Azure Blob Storage, OpenStack Swift) are similar in many ways.

HDFS is based on the **shared-nothing** principle, in contrast to the shared-disk approach of NAS and SAN architectures. Shared-disk storage relies on a centralized storage appliance with custom hardware, while shared-nothing requires only computers connected by a conventional datacenter network.

HDFS consists of a daemon process running on each machine, exposing a network service for file access. A central server called the **NameNode** keeps track of which file blocks are stored on which machine, conceptually creating one big filesystem across all machines. To tolerate failures, file blocks are replicated on multiple machines — either as full copies or using an **erasure coding** scheme such as Reed–Solomon codes, which allows recovery with lower storage overhead than full replication. These techniques are similar to RAID but operate over a conventional network without special hardware.

<br>

---

<br>

### MapReduce Job Execution

MapReduce is a programming framework for processing large datasets in a distributed filesystem like HDFS. The pattern is very similar to the Unix log analysis example from the previous section:

1. Read a set of input files and break them into records (e.g., each line in a log file).
2. Call the **mapper** function to extract a key and value from each input record (analogous to `awk '{print $7}'` extracting the URL).
3. Sort all key-value pairs by key (analogous to the first `sort` command).
4. Call the **reducer** function to iterate over the sorted key-value pairs, combining values for the same key (analogous to `uniq -c`).

Steps 2 (map) and 4 (reduce) are where you write custom data processing code. Step 1 is handled by the input format parser. Step 3 is implicit in MapReduce — the mapper output is always sorted before it reaches the reducer.

To create a MapReduce job, you implement two callback functions:

- **Mapper**: Called once for every input record. It extracts the key and value and may generate any number of key-value pairs (including none). It keeps no state between records — each is handled independently.
- **Reducer**: Receives all values belonging to the same key (collected by the framework) via an iterator, and produces output records (e.g., the count of occurrences of the same URL).

If you need a second sorting stage (e.g., ranking URLs by request count), you write a second MapReduce job using the first job's output as input. The mapper prepares data into a form suitable for sorting; the reducer processes the sorted data.

**Distributed Execution of MapReduce**

The key difference from Unix pipelines is that MapReduce parallelizes computation across many machines without requiring explicit parallelism in user code. The mapper and reducer only operate on one record at a time and don't need to know where their input comes from or output goes — the framework handles data movement between machines.

![map_reduce_job](./images/map_reduce_job.png)

The parallelization is based on partitioning: the input to a job is typically an HDFS directory, and each file or file block is a separate partition processed by a separate map task. The MapReduce scheduler tries to run each mapper on a machine that stores a replica of the input file — a principle known as **putting the computation near the data**, which reduces network load and increases locality.

The framework copies the application code (e.g., JAR files) to the assigned machines, starts the map task, and begins reading the input file, passing one record at a time to the mapper callback.

The reduce side is also partitioned. While the number of map tasks is determined by input file blocks, the number of reduce tasks is configured by the job author. To ensure all key-value pairs with the same key reach the same reducer, the framework uses a hash of the key to determine the target reduce task.

Sorting is performed in stages. Each map task partitions its output by reducer (based on key hash) and writes each partition to a sorted file on the mapper's local disk, using a technique similar to SSTables. When a mapper finishes, the scheduler notifies reducers to start fetching. Reducers connect to mappers and download their sorted partitions. This process of partitioning, sorting, and copying data from mappers to reducers is known as the **shuffle**.

The reduce task merges the files from mappers, preserving sort order so that records with the same key from different mappers become adjacent. The reducer is called with a key and an iterator over all records with that key, can apply arbitrary logic, and writes output records to the distributed filesystem (with replicas on other machines).

**MapReduce Workflows**

A single MapReduce job is limited in scope — it could determine page views per URL, but not the most popular URLs (which requires a second round of sorting). Therefore, MapReduce jobs are commonly chained into **workflows**, where the output of one job becomes the input to the next.

The Hadoop MapReduce framework has no built-in workflow support — chaining is done implicitly by directory name: the first job writes output to a designated HDFS directory, and the second job reads from that same directory. Chained jobs are therefore less like Unix pipelines (which stream data directly between processes) and more like a sequence of commands where each writes to a temporary file that the next reads from.

A batch job's output is only considered valid when the job completes successfully (partial output from failed jobs is discarded). One job in a workflow can only start when all prior jobs that produce its input have completed. Various workflow schedulers handle these dependencies, including Oozie, Azkaban, Luigi, Airflow, and Pinball. Workflows of 50 to 100 MapReduce jobs are common when building recommendation systems. Higher-level tools such as Pig, Hive, Cascading, Crunch, and FlumeJava also set up multi-stage workflows that are automatically wired together.
