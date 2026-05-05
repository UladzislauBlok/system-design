# Chapter 11.3 Stream Processing: Processing Streams

Once a stream is acquired, it can be processed in three broad ways:

1. Write the event data to a storage system (database, cache, search index) for querying. This keeps the database in sync with system changes.
2. Push events to users via alerts, push notifications, or real-time dashboards.
3. Process input streams to produce one or more output streams.

This section focuses on the third option: processing streams to produce derived streams.

A piece of code that processes streams is known as an **operator** or a **job**. Similar to batch processing jobs, a stream processor consumes input streams in a read-only fashion and writes its output to a different location in an append-only fashion. Sharding, parallelization, and basic mapping operations (transforming, filtering) follow similar patterns to batch processing.

The crucial difference from batch jobs is that a stream never ends. This unbounded nature has significant implications:

- Sorting is not applicable, meaning sort-merge joins cannot be used.
- Fault-tolerance mechanisms must adapt; restarting a long-running stream job from the beginning after a crash is often not viable.

<br>

---

<br>

### Uses of Stream Processing

Stream processing has long been used for monitoring purposes, where an organization wants to be alerted if certain things happen. Examples include:

- Fraud detection systems blocking potentially stolen credit cards based on usage pattern changes.
- Trading systems executing trades based on financial market price changes.
- Manufacturing systems monitoring machine status to quickly identify malfunctions.
- Military and intelligence systems tracking potential aggressors.

These applications require sophisticated pattern matching and correlations. Other uses of stream processing have also emerged.

**Complex event processing**

**Complex event processing (CEP)** is an approach for analyzing event streams to search for specific event patterns. Similar to regular expressions for strings, CEP allows defining rules to find event patterns in a stream.

CEP systems often use declarative query languages (like SQL) or GUIs. These queries are submitted to a processing engine that consumes input streams and maintains a state machine for matching. When a match occurs, the engine emits a **complex event** containing the detected pattern details.

In CEP, the query-data relationship is reversed compared to normal databases. Instead of storing data persistently and treating queries as transient, CEP engines store queries long-term and check arriving events against these standing queries.

**Stream analytics**

Stream analytics focuses on aggregations and statistical metrics over large volumes of events, rather than detecting specific sequences. Example applications include:

- Measuring the rate of specific events (e.g., frequency per time interval).
- Calculating rolling averages over a time period.
- Comparing current statistics to previous intervals (e.g., trend detection or anomaly alerting).

These statistics are typically computed over fixed time intervals, known as a **window**. Windowing smooths out fluctuations while providing a timely view of traffic pattern changes.

Stream analytics often employs probabilistic algorithms for efficiency, such as Bloom filters for set membership or HyperLogLog for cardinality estimation. While these algorithms produce approximate results and require less memory, stream processing itself is not inherently lossy or inexact; probabilistic algorithms are merely an optimization.

**Maintaining materialized views**

A stream of database changes can be used to keep derived data systems (caches, search indexes, data warehouses) up-to-date with a source database. This is a form of maintaining **materialized views**—deriving an alternative view of a dataset for efficient querying and updating it when underlying data changes.

In event sourcing, application state is also a materialized view derived from applying an event log. Unlike stream analytics, maintaining a materialized view often requires processing all events over an arbitrary time period, effectively needing a window that stretches back to the beginning of time (excluding compacted obsolete events).

**Incremental View Maintenance**

While databases can maintain materialized views, they often refresh them using periodic batch jobs or on-demand requests, which reprocesses all data and delays data freshness.

**Incremental view maintenance (IVM)** offers a more efficient solution by converting queries into operators capable of incremental computations. Instead of reprocessing entire datasets, IVM algorithms only recompute and update data that has changed. This efficiency allows for much more frequent updates, significantly increasing data freshness. Databases leveraging IVM ingest event streams, buffer recent events in memory, and periodically update on-disk materialized views in real time.

**Search on streams**

Some applications require searching for individual events based on complex criteria, such as full-text search queries (e.g., media monitoring services searching news feeds for specific topics, or real estate sites notifying users of new matching properties).

Similar to CEP, searching a stream reverses traditional search engine processing. The queries are stored, and arriving documents are evaluated against them. To optimize testing many documents against many queries, both queries and documents can be indexed.

**Event-driven architectures and RPC**

Message-passing systems are sometimes used as an alternative to Remote Procedure Calls (RPC), for example, in the actor model. However, they are not typically considered stream processors because:

- Actor frameworks manage concurrency and distributed execution, while stream processing is a data management technique.
- Actor communication is often ephemeral and one-to-one, whereas event logs are durable and multi-subscriber.
- Actors can communicate arbitrarily (including cyclic patterns), while stream processors usually form acyclic pipelines.

Despite this, there is overlap. Some systems, like Apache Storm's distributed RPC, blend user queries with event stream processing.

<br>

---

<br>

### Reasoning About Time

Stream processors often use time windows (e.g., "average over the last five minutes") for analytics, but defining time is tricky.

In batch processing, tasks read historical events rapidly, so processing is deterministic and relies entirely on timestamps embedded in the events. However, many stream processing frameworks use the local system clock on the processing machine (the **processing time**) for windowing. While simple, processing time breaks down when there is significant processing lag.

**Event time versus processing time**

Processing can be delayed by queueing, network faults, performance contention, or reprocessing past events during fault recovery.

These delays cause unpredictable message ordering (e.g., an event occurring later may reach the broker earlier). Relying on processing time instead of the event's actual timestamp (**event time**) leads to bad data. For example, processing a backlog of delayed events based on processing time might falsely indicate an anomalous spike in activity.

%image 1%

**Handling straggler events**

When windowing by event time, it is difficult to know if all events for a given window have arrived. For instance, after finishing a 1-minute window, you might still receive delayed events (**straggler events**) due to network interruptions.

Broadly, there are two options for handling stragglers:

- Ignore them, tracking the number of dropped events as a metric to alert on significant data loss.
- Publish a correction for the window that includes the stragglers, which may involve retracting the previous output.

In some systems, a special message can indicate that no more messages with earlier timestamps will arrive, triggering window completion. However, this is difficult when tracking multiple independent producers.

**Whose clock are you using, anyway?**

Assigning timestamps is complicated when events are buffered across the system (e.g., a mobile app offline). The true event time relies on the device's local clock, but user-controlled clocks cannot be trusted as they may be incorrect. The server's clock is accurate but does not reflect when the user interaction actually occurred.

To estimate the true event time and adjust for incorrect device clocks, log three timestamps:

- The time the event occurred (device clock).
- The time the event was sent (device clock).
- The time the event was received (server clock).

By subtracting the sent time from the received time, you can estimate the offset between the device and server clocks and apply this offset to the original event timestamp.

**Types of windows**

Once timestamps are established, you must define how windows are grouped over time for aggregations. Common types include:

- **Tumbling windows:** Fixed-length windows where every event belongs to exactly one window (e.g., 1-minute windows from 10:03:00 to 10:03:59).
- **Hopping windows:** Fixed-length windows with overlap to provide smoothing (e.g., a 5-minute window hopping forward every 1 minute).
- **Sliding windows:** Contains all events that occur within a certain interval of each other. Implemented by keeping a time-sorted buffer and expiring old events.
- **Session windows:** No fixed duration. Groups events for a single user that occur closely together, ending when the user is inactive for a specific period (e.g., 30 minutes).

Window operations maintain temporary state. Simple counters require minimal state, while sliding windows or stream joins require buffering events until the window finishes. Large windows or high-throughput streams demand sufficient capacity (memory or disk) on stream processing machines to maintain this state.

<br>

---

<br>

### Stream Joins

Since stream processing generalizes data pipelines to incremental processing of unbounded datasets, there is exactly the same need for joins on streams as there is in batch jobs. However, the continuous arrival of new events makes stream joins more challenging.

There are three broad types of joins: stream-stream, stream-table, and table-table.

**Stream-stream join (window join)**

Consider tracking the click-through rate of search results on a website. You must join a stream of search queries with a stream of click events, linked by a session ID.

Because a user might click a result minutes or hours later (or never), a simple point-in-time join fails. Instead, the stream processor must maintain a **window** of recent events (e.g., storing all search events for the last hour). When a click arrives, it checks the window for a matching search to emit a joined result. If a search event expires from the window without a matching click, it emits an event indicating no click occurred.

**Stream-table join (stream enrichment)**

A stream of activity events often contains only identifiers (like a user ID) and needs to be enriched with context from a database. This is a **stream-table join**.

Querying a remote database for every event is slow. Instead, the stream processor can keep a local copy of the database, performing a **hash join**. To keep this local copy up-to-date, the stream processor subscribes to a Change Data Capture (CDC) changelog of the database. The process effectively joins the activity event stream with the profile update changelog stream.

Unlike a stream-stream join, the table's "window" conceptually reaches back to the beginning of time, with newer records overwriting older ones.

**Table-table join (materialized view maintenance)**

Consider maintaining a social network timeline cache. Iterating over all followees to merge recent posts on read is too expensive. Instead, a stream processor can maintain a per-user "inbox" as posts arrive.

This requires streams for posts (sending/deleting) and follow relationships (following/unfollowing). The stream processor maintains a **materialized view** corresponding to a query that joins two tables (`posts` and `follows`). The timelines are effectively a cache of this query result, continuously updated as the underlying tables change.

**Time dependence of joins**

All stream joins require the processor to maintain state derived from one input and query it when processing the other. The order of events is critical (e.g., following then unfollowing vs. unfollowing then following).

If events on different streams happen simultaneously, the order they are processed in can affect the result. If event ordering across streams is undetermined, the join becomes **nondeterministic**; rerunning the job might yield different results due to different event interleaving.

In data warehouses, this is addressed using **slowly changing dimensions (SCD)**, assigning unique identifiers to specific record versions (e.g., tracking the specific tax rate version applicable at the time of a sale). This makes the join deterministic but prevents log compaction, as all versions must be retained. Alternatively, the data can be denormalized by including the necessary context directly in the event.

<br>

---

<br>

### Fault Tolerance

Batch processing frameworks tolerate faults easily: failed tasks are restarted, and partial outputs are discarded. This ensures the output is identical to a failure-free run, providing **exactly-once semantics** (or effectively-once semantics).

Stream processing cannot wait for a task to finish before making output visible because streams are infinite. Different fault-tolerance mechanisms are required.

**Microbatching and checkpointing**

One approach is **microbatching**, breaking the stream into small blocks (e.g., 1-second batches) and treating each as a miniature batch process. This provides exactly-once semantics within the framework, but implicitly uses processing-time windows equal to the batch size.

A variant is periodically generating rolling **checkpoints** of state to durable storage. If a crash occurs, the operator restarts from the last checkpoint and discards uncommitted output.

While these approaches provide exactly-once semantics internally, they cannot recall external side effects (like sending an email or writing to an external database) if a microbatch or checkpoint fails after the side effect occurred.

**Atomic commit revisited**

To achieve exactly-once processing with external systems, all outputs and side effects must persist if and only if processing is successful. This requires an atomic commit facility across operator state, database writes, and message acknowledgments.

While traditional distributed transactions (like XA) are problematic across heterogeneous systems, stream processing frameworks can implement atomic commits efficiently by keeping the transactions internal and managing both state changes and messaging themselves.

**Idempotence**

Another way to handle retries safely is relying on **idempotence**—an operation that can be performed multiple times with the same effect as being performed once (e.g., setting a key-value pair vs. incrementing a counter).

Operations that are not inherently idempotent can often be made so using metadata. For example, attaching a Kafka message offset to a database write allows the system to ignore duplicate updates. This relies on assumptions like deterministic processing and strict replay order, but offers a low-overhead path to exactly-once semantics.

**Rebuilding state after a failure**

Stream processes requiring state (windowed aggregations, tables for joins) must recover that state after a failure. Options include:

- Keeping state in a remote datastore (can be slow).
- Keeping state local and periodically replicating it to durable storage (like a distributed filesystem) or a log-compacted topic.
- Rebuilding state entirely from input streams (viable for short windows or fast CDC replays).

The best approach depends on the performance characteristics of the underlying infrastructure, balancing network latency against disk access speeds.
