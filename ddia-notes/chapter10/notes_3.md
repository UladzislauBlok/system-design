# Chapter 10.3 Batch Processing: Beyond MapReduce

Although MapReduce became very popular in the late 2000s, it is just one among many possible programming models for distributed systems. Depending on the volume and structure of data, other tools may be more appropriate. We discussed MapReduce extensively because it is a useful learning tool and a fairly clear abstraction on top of a distributed filesystem. While it is easy to understand, it is not easy to use. Implementing a complex processing job using raw MapReduce APIs is quite laborious, requiring developers to implement join algorithms from scratch.

In response to the difficulty of using MapReduce directly, various higher-level programming models (like Pig, Hive, Cascading, and Crunch) were created as abstractions on top of it. These are fairly easy to learn and make common batch processing tasks significantly easier to implement.

However, there are problems with the MapReduce execution model itself that manifest as poor performance for certain kinds of processing. MapReduce is very robust, capable of processing almost arbitrarily large quantities of data on an unreliable multi-tenant system with frequent task terminations, but other tools are sometimes orders of magnitude faster.

<br>

---

<br>

### Materialization of Intermediate State

Every MapReduce job is independent from every other job. The main contact points of a job with the rest of the world are its input and output directories on the distributed filesystem. If you want the output of one job to become the input to a second job, an external workflow scheduler must configure the directories and start the second job only once the first job has completed.

This setup is reasonable if the output from the first job is a dataset that you want to publish widely within your organization. Publishing data to a well-known location in the distributed filesystem allows loose coupling between the producers and consumers. However, in many complex workflows, the output of one job is only ever used as input to one other job maintained by the same team. In this case, the files on the distributed filesystem are simply **intermediate state** (a temporary means of passing data from one job to the next).

The process of eagerly computing the result of some operation and writing it out to files is called **materialization**. By contrast, Unix pipes connect the output of one command with the input of another without fully materializing the intermediate state. Instead, they stream the output to the input incrementally using only a small in-memory buffer.

MapReduce’s approach of fully materializing intermediate state has several downsides compared to Unix pipes:

- A MapReduce job can only start when all tasks in the preceding jobs have completed, whereas processes connected by a Unix pipe are started at the same time. Straggler tasks in MapReduce can slow down the execution of the workflow as a whole.
- Mappers are often redundant, simply reading back the same file that was just written by a reducer to prepare it for the next stage. If the reducer output was partitioned and sorted in the same way as mapper output, reducers could be chained together directly.
- Storing intermediate state in a distributed filesystem means those files are replicated across several nodes, which is often overkill for temporary data.

**Dataflow engines**

In order to fix these problems with MapReduce, several new execution engines for distributed batch computations were developed, such as Spark, Tez, and Flink. Since they explicitly model the flow of data through several processing stages, these systems are known as **dataflow engines**. They handle an entire workflow as one job, rather than breaking it up into independent subjobs. Like MapReduce, they work by repeatedly calling a user-defined function to process one record at a time on a single thread. They parallelize work by partitioning inputs, and they copy the output of one function over the network to become the input to another function.

Unlike in MapReduce, these functions need not take the strict roles of alternating map and reduce, but instead can be assembled in more flexible ways. We call these functions **operators**. The dataflow engine provides several different options for connecting one operator’s output to another’s input:

- One option is to repartition and sort records by key, enabling sort-merge joins.
- Another possibility is to take several inputs and partition them in the same way, but skip the sorting.
- For broadcast hash joins, the same output from one operator can be sent to all partitions of the join operator.

This style of processing engine offers several advantages compared to the MapReduce model:

- Expensive work such as sorting need only be performed in places where it is actually required.
- There are no unnecessary map tasks, since the work done by a mapper can often be incorporated into the preceding reduce operator.
- Because all joins and data dependencies in a workflow are explicitly declared, the scheduler can make locality optimizations. For example, it can try to place the consumer task on the same machine as the producer task.
- It is usually sufficient for intermediate state between operators to be kept in memory or written to local disk, which requires less I/O than writing it to HDFS.
- Operators can start executing as soon as their input is ready.
- Existing JVM processes can be reused to run new operators, reducing startup overheads.

**Fault tolerance**

An advantage of fully materializing intermediate state to a distributed filesystem is that it is durable, which makes fault tolerance fairly easy in MapReduce. Because dataflow engines avoid writing intermediate state to HDFS, they take a different approach to tolerating faults: if a machine fails and the intermediate state on that machine is lost, it is recomputed from other data that is still available.

To enable this recomputation, the framework must keep track of how a given piece of data was computed. Spark uses the resilient distributed dataset (RDD) abstraction for tracking the ancestry of data, while Flink checkpoints operator state to allow it to resume running an operator that ran into a fault.

When recomputing data, it is crucial that the computation is **deterministic** (given the same input data, operators always produce the same output). If nondeterministic operators are used and lost data has already been sent to downstream operators, it becomes very hard to resolve contradictions between the old and new data. To avoid cascading faults, causes of nondeterminism, such as iterating over hash tables without a guaranteed order or using the system clock, must be removed.
_Note: Recovering from faults by recomputing data is not always the right answer. If the intermediate data is much smaller than the source data, or if the computation is very CPU-intensive, it is probably cheaper to materialize the intermediate data to files._

**Discussion of materialization**

Returning to the Unix analogy, MapReduce is like writing the output of each command to a temporary file, whereas dataflow engines look much more like Unix pipes. Flink is built around the idea of **pipelined execution**, incrementally passing the output of an operator to other operators without waiting for the input to be complete.

While operations like sorting inevitably need to consume their entire input before producing any output, many other parts of a workflow can be executed in a pipelined manner. When the job completes, its output still needs to go to a durable location, so materialized datasets on HDFS are usually still the inputs and the final outputs of a job. The key improvement over MapReduce is avoiding the need to write all the intermediate state to the filesystem as well.

<br>

---

<br>

### Graphs and Iterative Processing

While Chapter 2 focused on using graphs for OLTP-style use (quickly executing queries to find a small number of vertices), it is also useful to look at graphs in a batch processing context to perform offline processing or analysis on an entire graph. This is often used in machine learning applications like recommendation engines or ranking systems (e.g., PageRank to estimate web page popularity).
_Note: Dataflow engines arrange operators as a directed acyclic graph (DAG) where the flow of data is structured as a graph, but the data itself consists of relational-style tuples. In graph processing, the data itself has the form of a graph._

Many graph algorithms traverse one edge at a time, joining a vertex with an adjacent vertex to propagate information, repeating until a condition is met (e.g., no more edges, or a metric converges). It is possible to store a graph in a distributed filesystem, but this "repeating until done" cannot be expressed in plain MapReduce because it only performs a single pass over the data. Thus, it is often implemented in an iterative style:

1. An external scheduler runs a batch process to calculate one step of the algorithm.
2. When the batch process completes, the scheduler checks whether it has finished based on the completion condition (e.g., no more edges, or change below a threshold).
3. If it has not yet finished, the scheduler goes back to step 1 and runs another round of the batch process.

Implementing this with MapReduce is inefficient because it does not account for the algorithm's iterative nature. MapReduce always reads the entire input dataset and produces a completely new output dataset, even if only a small part of the graph changed compared to the last iteration.

**The Pregel processing model**

As an optimization for batch processing graphs, the **bulk synchronous parallel (BSP)** model of computation (also known as the **Pregel model**) became popular, implemented by Apache Giraph, Spark's GraphX API, and Flink's Gelly API.
In Pregel, one vertex can "send a message" to another vertex (typically along the edges in a graph), similar to mappers sending messages to a reducer call. In each iteration, a function is called for each vertex, passing it all the messages that were sent to it. The key difference from MapReduce is that a vertex remembers its state in memory from one iteration to the next, so the function only needs to process new incoming messages. If no messages are sent in a part of the graph, no work is done there.

**Fault tolerance**

Vertices only communicate by message passing, which improves performance because messages can be batched with less waiting. The only waiting is between iterations: all messages sent in one iteration are guaranteed to be delivered in the next, so the prior iteration must completely finish before the next can start.
Pregel implementations guarantee that messages are processed exactly once at their destination vertex in the following iteration. The framework transparently recovers from faults by periodically checkpointing the state of all vertices at the end of an iteration (writing their full state to durable storage). If a node fails, the simplest solution is to roll back the entire graph computation to the last checkpoint and restart. If the algorithm is deterministic and messages are logged, it is possible to selectively recover only the lost partition.

**Parallel execution**

A vertex simply sends messages to a vertex ID without knowing the physical machine it executes on. The framework partitions the graph, deciding which vertex runs on which machine and how to route messages.
Because the model deals with one vertex at a time ("thinking like a vertex"), the framework often partitions the graph by an arbitrarily assigned vertex ID rather than grouping related vertices together, as optimized partitioning is hard. As a result, graph algorithms often have significant cross-machine communication overhead, where intermediate state (messages) is bigger than the original graph, slowing down the algorithm.
Consequently, if a graph can fit in memory on a single computer, a single-machine algorithm will likely outperform a distributed batch process. Even if the graph only fits on the disks of a single computer, single-machine processing (e.g., GraphChi) is viable. Distributed approaches like Pregel are unavoidable only if the graph is too big to fit on a single machine.

<br>

---

<br>

### High-Level APIs and Languages

Over the years, execution engines for distributed batch processing have matured. With the problem of physically operating batch processes at scale (petabytes of data, 10,000+ machines) mostly solved, attention has turned to improving the programming model, processing efficiency, and the set of problems these technologies can solve.
Higher-level languages and APIs like Hive, Pig, Cascading, and Crunch became popular because programming MapReduce jobs by hand is laborious. These high-level languages were able to move to the new Tez dataflow execution engine without requiring job code rewrites. Spark and Flink also include their own high-level dataflow APIs.
These dataflow APIs generally use relational-style building blocks: joining datasets on a field, grouping tuples by key, filtering by a condition, and aggregating tuples (counting, summing). Internally, these operations use the join and grouping algorithms discussed earlier.
Besides requiring less code, these high-level interfaces allow interactive use (writing and observing analysis code incrementally in a shell) and improve job execution efficiency at a machine level.

**The move toward declarative query languages**

An advantage of specifying joins as relational operators is that the framework can analyze the properties of the join inputs and automatically decide which join algorithm is most suitable. Hive, Spark, and Flink have cost-based query optimizers that do this, even reordering joins to minimize intermediate state.
This is possible if joins are specified in a **declarative** way: the application states which joins are required, and the query optimizer decides how they can best be executed.
However, MapReduce and its dataflow successors are very different from the fully declarative query model of SQL. MapReduce is built around function callbacks where user-defined functions (the mapper or reducer) are free to call arbitrary code. This approach easily draws upon a large ecosystem of existing libraries (e.g., parsing, numerical algorithms) and package managers, distinguishing batch processing systems from MPP databases.
Dataflow engines have found advantages to incorporating more declarative features outside of joins. If a callback function only contains a simple filtering condition, there is significant CPU overhead in calling the function on every record. If expressed declaratively, the query optimizer can take advantage of column-oriented storage layouts to read only the required columns from disk. Hive, Spark DataFrames, and Impala also use vectorized execution: iterating over data in a tight inner loop friendly to CPU caches and avoiding function calls. By incorporating declarative aspects, batch processing frameworks begin to look more like MPP databases while retaining their flexibility advantage.

**Specialization for different domains**

While the extensibility to run arbitrary code is useful, many common processing patterns reoccur, making reusable implementations of common building blocks valuable. Traditionally, MPP databases served business intelligence and reporting, but batch processing spans many other domains.
Another domain of increasing importance is statistical and numerical algorithms (needed for machine learning). Reusable implementations are emerging (e.g., Mahout on top of MapReduce/Spark/Flink, or MADlib inside an MPP database).
Spatial algorithms (e.g., k-nearest neighbors for similarity search) and approximate search for genome analysis algorithms are also gaining importance.
As batch processing systems gain built-in functionality and high-level declarative operators, and as MPP databases become more programmable, the two are beginning to look more alike: in the end, they are all just systems for storing and processing data.
