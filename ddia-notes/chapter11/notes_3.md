# Chapter 11.3 Stream Processing: Processing Streams

This section deals with the actual processing of event streams to generate meaningful outputs or maintain derived state.

<br>

---

<br>

### Uses of Stream Processing

Common uses for stream processing include:
- **Complex Event Processing (CEP)**: Identifying patterns in streams (e.g., fraud detection).
- **Stream Analytics**: Computing aggregations and summaries over windows of time.
- **Maintaining Materialized Views**: Updating search indexes or caches.

<br>

---

<br>

### Reasoning About Time

One of the hardest problems in stream processing is dealing with time, particularly when event production and processing are separated.

- **Event Time vs. Processing Time**: The difference between when an event happened and when it was processed.
- **Windowing**: Grouping events into time-based buckets (e.g., tumbling, sliding, or session windows).
- **Late-Arriving Events**: Handling events that arrive after their window has closed.

<br>

---

<br>

### Stream Joins

Joining two streams is more complex than joining two database tables because at least one of the datasets is continuously changing.

- **Stream-Stream Joins**: Joining two independent streams (e.g., clicks and impressions).
- **Stream-Table Joins**: Joining a stream with a static or slowly-changing table.
- **Table-Table Joins**: Both sides of the join are based on database tables, treated as streams.

<br>

---

<br>

### Fault Tolerance

Ensuring that every event is processed exactly once, or at least that effects are idempotent.

- **Checkpointing**: Saving state periodically to durable storage.
- **Idempotence**: Designing operations such that they can be safely retried without side effects.
- **Micro-batching**: Processing events in small, discrete batches to simplify recovery.

<br>

---

<br>

### Summary

Stream processing represents a shift from processing data once (batch) to processing data as it arrives. It enables lower latency, but introduces new challenges around time, state, and reliability.
