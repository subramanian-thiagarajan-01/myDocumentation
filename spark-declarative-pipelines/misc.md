# 16. Misc

## 16. Comparison & Alternatives

### 16.1 DLT vs Traditional Spark Jobs

| Aspect           | DLT                                  | Spark Jobs                |
| ---------------- | ------------------------------------ | ------------------------- |
| Orchestration    | Automatic (DAG inferred)             | Manual (script order)     |
| State management | Automatic (checkpoints)              | Manual implementation     |
| Failure recovery | Automatic (checkpoint replay)        | Custom logic required     |
| Data quality     | Built-in (Expectations)              | Manual validation         |
| Cost             | Optimized (incremental, autoscaling) | Depends on implementation |
| Learning curve   | Lower (declarative)                  | Higher (imperative)       |

### 16.2 DLT vs Databricks Workflows

- **DLT**: Table-to-table dependencies; ideal for pure data pipeline orchestration
- **Workflows**: Task orchestration; SQL queries, Python scripts, shell commands; more flexible

Use **DLT** for ETL pipelines. Use **Workflows** for orchestrating heterogeneous tasks (DLT pipeline + Notebook + SQL query + API call).

### 16.3 DLT vs Airflow

- **DLT**: Native Databricks; simpler; automatic retry, monitoring, state management
- **Airflow**: Open-source; more flexible; requires manual Databricks hook configuration

DLT is **not a replacement** for Airflow; different abstraction levels.

### 16.4 DLT vs Kafka Streams / Flink

- **DLT**: Batch + stream unified; managed by Databricks
- **Kafka Streams / Flink**: Lower-level stream processing; more control, steeper ops burden

DLT is higher-level; preferred for lakehouse pipelines.

---

## 17. Interview-Specific Scenarios

### 17.1 Designing an End-to-End Pipeline

**Scenario**: Ingest real-time e-commerce transactions, clean, aggregate hourly revenue.

```python
import dlt
from pyspark.sql import functions as F
from datetime import datetime

# Bronze: Raw ingestion
@dlt.create_streaming_table(name="transactions_bronze")
def transactions_bronze():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .option("cloudFiles.schemaLocation", "/mnt/schemas/transactions") \
        .option("cloudFiles.inferColumnTypes", "true") \
        .load("s3://transactions/events/")

# Silver: Cleaned, with quality expectations
@dlt.table
@dlt.expect_or_drop("non_negative_amount", "amount >= 0")
@dlt.expect_or_drop("valid_customer_id", "customer_id IS NOT NULL")
@dlt.expect_or_fail("valid_timestamp", "transaction_timestamp <= current_timestamp()")
def transactions_silver():
    return dlt.read_stream("transactions_bronze") \
        .withColumn("transaction_date", F.to_date("transaction_timestamp")) \
        .filter("amount > 0") \
        .filter("customer_id IS NOT NULL")

# Gold: Hourly aggregation (materialized view for cost efficiency)
@dlt.table
def revenue_hourly_gold():
    return dlt.read("transactions_silver") \
        .withColumn("hour", F.date_trunc("hour", "transaction_timestamp")) \
        .groupBy("hour") \
        .agg(
            F.sum("amount").alias("total_revenue"),
            F.count("*").alias("transaction_count"),
            F.approx_percentile("amount", 0.5).alias("median_amount")
        ) \
        .orderBy("hour")
```

**Interview talking points**:

- Streaming table at Bronze (incremental)
- Expectations at Silver (quality enforcement)
- Materialized view at Gold (cost-effective aggregation)
- Windowing/groupBy automatically handles lateness with watermarks

### 17.2 Debugging a Failed Pipeline

**Scenario**: DLT pipeline fails with "Cannot cast StringType to IntegerType" error.

**Approach**:

1. **Check event log**: Query `event_log()` for `update_status = 'FAILED'`, read error details
2. **Examine source data**: Sample raw table, identify rows with unexpected types
3. **Add debugging expectations**: Log data quality metrics before transformation
4. **Use development mode**: Faster iteration for troubleshooting
5. **Extract problematic step**: Run in separate notebook for interactive debugging

```python
# Debugging notebook
raw = spark.read.table("transactions_bronze")
raw.filter("amount LIKE '%text%'").display()  # Find non-numeric amounts

# Add rescue column to handle bad data
@dlt.table
def transactions_silver_robust():
    return dlt.read("transactions_bronze") \
        .withColumn("amount_parsed",
            F.try_cast("amount", "DECIMAL(18,2)")) \
        .filter("amount_parsed IS NOT NULL")  # Drop unparseable rows
```

### 17.3 Handling Schema Evolution

**Scenario**: Source system adds new column; existing pipeline breaks.

**Root cause**: Explicit schema definition is too strict.

**Solution**:

```python
# Before (breaks on new columns)
@dlt.table(schema="id LONG, name STRING")
def users():
    return spark.readStream.format("cloudFiles").load("s3://users/")

# After (allows new columns via rescue)
@dlt.table
def users():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.schemaLocation", "/mnt/schemas/users") \
        .option("cloudFiles.schemaEvolutionMode", "additive") \
        .option("cloudFiles.rescuedDataColumn", "_rescued_data") \
        .load("s3://users/")
```

### 17.4 Trade-offs: Continuous vs Triggered

**Scenario**: CEO demands real-time dashboard; cost is high.

**Decision matrix**:

- **Continuous**: Latency <5 min; cost ~$10k/month (constant cluster)
- **Triggered (5-min schedule)**: Latency ~5 min; cost ~$2k/month

**Recommendation**: Triggered pipeline with 5-minute schedule; achieves real-time for most business use cases at 1/5 the cost.

### 17.5 CDC Implementation with SCD Type 2

**Scenario**: Track customer attribute changes over time; maintain full history.

```python
@dlt.create_streaming_table(name="customer_updates_stream")
def customer_updates_stream():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .load("s3://cdc/customers/")
    # Schema: customer_id, email, address, change_type, change_timestamp

@dlt.apply_changes(
    target="customers_scd2",
    source="customer_updates_stream",
    keys=["customer_id"],
    sequence_by="change_timestamp",
    stored_as_scd_type=2,
    apply_as_deletes=expr("change_type = 'DELETE'")
)
def apply_cdc_scd2():
    pass

# Query current state
SELECT * FROM customers_scd2 WHERE __end_at IS NULL

# Query historical state at specific date
SELECT * FROM customers_scd2 WHERE __start_at <= '2025-06-01' AND __end_at > '2025-06-01'
```

---

## Critical Knowledge for Interviews

### Mechanics You Must Know

1. **Checkpoints are incremental**: Streaming tables store processed offsets; recovery resumes from last offset
2. **Exactly-once is per-partition**: Kafka/message queues guarantee exactly-once per partition; Auto Loader per file
3. **Materialized views are deterministic**: Same input always produces same output; safe to recompute fully
4. **Expectations are real-time**: Metrics logged as data passes through; not post-hoc validation
5. **DLT infers streaming vs batch**: If source is streaming (readStream), table is streaming; else, it's a materialized view
6. **State is locality-sensitive**: Large state stores benefit from RocksDB; serverless auto-manages

### Common Pitfalls

1. **Idempotency**: Streaming + append-only = duplicates on restart
2. **Circular deps**: Will be caught immediately; design tables bottom-up
3. **Checkpoint corruption**: Manual recovery required; plan for it
4. **Schema drift**: Auto Loader rescue column is your friend
5. **Over-specifying schema**: Avoid explicit schemas in streaming ingestion; use inference + evolution
6. **Stateful ops without watermarks**: Windows/joins will backlog indefinitely without watermarks
7. **Confusing EXPECT variants**: EXPECT (logs), EXPECT_OR_DROP (silent removal), EXPECT_OR_FAIL (hard stop)

### Trade-offs to Articulate

- **Latency vs Cost**: Streaming continuous (low latency, high cost) vs triggered (higher latency, lower cost)
- **Incremental vs Determinism**: Streaming tables are incremental but require idempotent logic; MVs are deterministic but expensive
- **Strictness vs Flexibility**: Early schema enforcement prevents bad data; late schema enforcement (rescue columns) allows drift discovery
- **Complexity vs Automation**: DLT abstracts away (simpler, less control); Workflows/Spark jobs expose (more control, more work)

---

## References & Latest Updates (as of Dec 2025)

- **Spark Declarative Pipelines** (Apache Spark 4.1+): Databricks contributing DLT to open-source Spark
- **Serverless Incremental MVs**: 6.5x throughput improvement, 85% latency reduction (200B-row benchmark)
- **RocksDB state management**: Default for stateful ops; automatically tuned on serverless
- **Photon Acceleration**: 2-4x speedup; ~2x DBU cost; skip for I/O-bound workloads
- **Unity Catalog**: All DLT tables eligible for row/column security
- **Event log querying**: `event_log(table(...))` syntax enables centralized monitoring

---

## No-Fluff Summary

DLT is a declarative ETL framework built on Spark Structured Streaming. You declare tables; DLT handles DAG construction, orchestration, state management, and failure recovery. Streaming tables are incremental; materialized views deterministic. Expectations enforce quality in real-time. APPLY CHANGES handles CDC (SCD1/2) natively. Checkpoints enable exactly-once semantics. Event logs provide deep observability. Biggest pitfall: assuming idempotency; design for it. Trade-offs are latency vs cost, complexity vs control. Master checkpoints, expectations, and the streaming-batch unification; you'll ace the interview.
