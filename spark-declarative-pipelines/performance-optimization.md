# 12. Performance Optimization

### 12.1 Incremental Processing (Key to Cost Reduction)

**Streaming tables**: Always incremental (only process new data).

**Materialized views on serverless**: Automatically incremental refresh (cost-based optimizer).

**Materialized views on classic compute**: Full recompute by default (unless you manually write incremental logic with MERGE).

**Benchmark**: 6.5x throughput improvement and 85% lower latency with incremental MVs on serverless (200B rows).

### 12.2 Partitioning Strategies

```python
@dlt.table(
    partition_cols=["date"]  # Partitions table by date
)
def events():
    return spark.readStream.format("cloudFiles") \
        .load("s3://events/") \
        .withColumn("date", F.to_date("event_time"))
```

Partitioning enables:

- **Partition pruning**: Queries filter out partitions (faster)
- **Incremental ingestion**: Auto Loader discovers new partitions only

### 12.3 Join Optimization

Best practices:

- **Filter early**: Reduce join input size
- **Broadcast small tables**: Use `broadcast()` hint for tables <2GB
- **Avoid self-joins**: Reframe logic to eliminate

### 12.4 State Store Optimization (RocksDB)

For stateful streaming (aggregations, joins with windows), enable RocksDB:

```python
spark.conf.set("spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.streaming.state.RocksDBStateStoreProvider")
```

Serverless automatically manages state store; classic compute requires manual configuration.

### 12.5 Backfill Strategies

**Full backfill** (reset + reprocess):

```python
# In UI: click "Full Refresh" button
# Or via API: set refresh_all=true in update request
```

**Incremental backfill** (process new data only):

- Streaming tables: Automatic after recovery
- Materialized views: No explicit control; DLT decides

**Manual incremental**:

```python
@dlt.table
def backfill_window():
    # Process data from specific date range
    return spark.read.table("source") \
        .filter(F.col("date") >= F.lit("2025-01-01")) \
        .filter(F.col("date") < F.lit("2025-02-01"))
```

---
