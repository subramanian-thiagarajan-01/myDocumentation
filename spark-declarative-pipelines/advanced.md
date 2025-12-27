# 15. Advanced & Edge Cases

### 15.1 Backfills and Reprocessing

**Scenario**: Historical data source added; need to reprocess last 2 years.

**Approach 1**: Full refresh (reset entire pipeline)

```python
# UI: Full Refresh button
# Reprocesses all data from source; expensive but clean
```

**Approach 2**: Windowed incremental load

```python
@dlt.table
def raw_data():
    # Read files from specific date range
    return spark.readStream.format("cloudFiles") \
        .load(f"s3://bucket/events/2023/") \
        .union(spark.readStream.format("cloudFiles").load("s3://bucket/events/2024/")) \
        .union(spark.readStream.format("cloudFiles").load("s3://bucket/events/2025/"))
```

**Approach 3**: External orchestration
Use Databricks Workflows to trigger multiple DLT pipeline runs with different parameters (not natively supported in DLT itself).

### 15.2 Schema Drift Handling

**Problem**: Source schema changes unexpectedly; pipeline breaks.

**Solutions**:

1. **Auto Loader rescue column** (capture unexpected fields):

```python
@dlt.table
def bronze():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.rescuedDataColumn", "_rescued_data") \
        .load("s3://bucket/")
    # Unknown columns → JSON in _rescued_data
```

2. **Lenient schema inference**:

```python
.option("cloudFiles.schemaEvolutionMode", "additive")  # New columns OK
```

3. **Manual schema hints**:

```python
.option("cloudFiles.schemaHints", "id LONG, name STRING, amount DECIMAL(18,2)")
```

4. **Validate in Silver layer**:

```python
@dlt.table
@dlt.expect("schema_valid", "SIZE(MAP_KEYS(_rescued_data)) = 0")
def silver():
    return dlt.read("bronze").filter("_rescued_data IS NULL")  # Fail if unknown columns
```

### 15.3 Idempotency and Deduplication

**Challenge**: Pipeline failure mid-run causes duplicate writes. Solution: Make transformations idempotent.

**Idempotent pattern 1**: Upsert (overwrite, not append)

```python
@dlt.table
def customer_gold():
    updates = dlt.read("customer_silver")
    return spark.sql("""
        MERGE INTO customer_gold_target USING updates
        ON customer_gold_target.customer_id = updates.customer_id
        WHEN MATCHED THEN UPDATE SET *
        WHEN NOT MATCHED THEN INSERT *
    """)
```

**Idempotent pattern 2**: Deduplication key

```python
@dlt.table
def deduped_events():
    return dlt.read("raw_events") \
        .dropDuplicates(["event_id", "event_timestamp"]) \
        .repartition("date")
```

**Not idempotent**: Append-only inserts

```python
# Avoid this for recovery scenarios
@dlt.table
def events_append():
    return spark.readStream.table("raw_events")
    # If pipeline restarts, duplicate rows inserted
```

### 15.4 Checkpoint Corruption Recovery

**Symptom**: Pipeline halts; error message references checkpoint.

**Recovery**:

1. Stop pipeline
2. Delete checkpoint directory: `storage_location/system/checkpoints/{table_name}`
3. Restart pipeline (triggers full reprocess of streaming table)
4. Monitor for duplicates if logic is not idempotent

```bash
# Via Databricks CLI
dbutils.fs.rm("s3://my-bucket/dlt/system/checkpoints/my_table", True)
```

### 15.5 Handling Large Stateful Operations

**Problem**: Window joins or large aggregations consume excessive state store memory.

**Solutions**:

1. **Increase state store size** (serverless auto-manages):

```python
spark.conf.set("spark.sql.streaming.stateStore.rocksdb.limitCommitMemoryInMB", "512")
```

2. **Partition state**:

```python
@dlt.table
def windowed_events():
    return dlt.read_stream("events") \
        .groupBy(
            F.window("event_time", "10 minutes"),
            "user_id"
        ) \
        .agg(count("*")) \
        .repartition("user_id")  # Spread state across partitions
```

3. **Reduce window size** or retention period:

```python
.groupBy(F.window("event_time", "1 minute", "30 seconds"))  # Smaller windows = less state
```
