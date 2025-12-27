# 4. Data Ingestion

### 4.1 Auto Loader (cloudFiles)

Auto Loader is the streaming source for cloud storage with built-in:

- Incremental file discovery
- Schema inference/evolution
- Exactly-once delivery
- Rescue column for unexpected data

```python
@dlt.create_streaming_table(name="bronze_events")
def bronze_events():
    return spark.readStream \
        .format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .option("cloudFiles.schemaLocation", "/mnt/schemas/events") \
        .option("cloudFiles.inferColumnTypes", "true") \
        .option("cloudFiles.rescuedDataColumn", "_rescued_data") \
        .load("s3://my-bucket/events/")
```

**Schema inference**: Samples first 50GB or 1000 files (whichever limit crossed first). Stores inferred schema in `_schemas/` directory.

**Schema evolution**:

```python
.option("cloudFiles.schemaEvolutionMode", "additive")  # or failOnNewColumns, rescue, evolutionKillBits
```

### 4.2 Streaming vs Batch Ingestion Trade-offs

| Aspect           | Streaming                                    | Batch                                 |
| ---------------- | -------------------------------------------- | ------------------------------------- |
| Latency          | Low (seconds to minutes)                     | High (hours to days)                  |
| State management | Checkpoints required                         | Not needed                            |
| Cost             | Variable (per-micro-batch overhead)          | Predictable (full scans)              |
| Failure recovery | Checkpoint-based; may require manual cleanup | Simple; just rerun                    |
| Best for         | Real-time data, continuous sources           | Historical loads, one-time migrations |

**Interview insight**: Streaming ingestion is stateful; batch is stateless. State corruption is the #1 streaming failure mode.

### 4.3 Handling Late and Out-of-Order Data

Streaming tables without watermarks process all data; with watermarking:

```python
@dlt.create_streaming_table(name="events_with_watermark")
def events_with_watermark():
    return spark.readStream \
        .format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .load("s3://bucket/events/") \
        .withWatermark("event_time", "10 minutes")  # Events within 10 min of max event time processed
```

Watermarks only affect **stateful operations** (windows, joins). For append-only ingestion, watermarks have no effect.
