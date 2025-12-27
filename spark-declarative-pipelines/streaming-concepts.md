# 6. Streaming Concepts in DLT

### 6.1 Structured Streaming Fundamentals

Spark Structured Streaming models streams as unbounded tables with micro-batches:

- Each micro-batch executes as a mini-Spark job
- Fault tolerance via Write-Ahead Log (WAL) and checkpoints
- Exactly-once semantics for deterministic sinks (Delta, databases with idempotent writes)

**DLT abstracts**: You declare a streaming table; DLT manages checkpoints, triggers, micro-batch size.

### 6.2 Event-Time vs Processing-Time vs Ingestion-Time

| Concept         | Definition                    | Example                    |
| --------------- | ----------------------------- | -------------------------- |
| Event-time      | When event occurred (in data) | `order_timestamp` in JSON  |
| Processing-time | When Spark processes event    | Micro-batch execution time |
| Ingestion-time  | When file written to storage  | File modification time     |

DLT **does not expose processing-time** directly; you work with event-time via `withWatermark()`.

### 6.3 Exactly-Once Semantics in DLT

Enabled by:

1. **Idempotent producer**: Auto Loader deduplicates based on file path + offset
2. **Transactional sink**: Delta Lake ACID guarantees
3. **Checkpointing**: Track processed offsets; resume from last checkpoint

**Critical caveat**: Exactly-once is per-partition for Kafka/message queues. DLT + Auto Loader guarantees exactly-once per file.
