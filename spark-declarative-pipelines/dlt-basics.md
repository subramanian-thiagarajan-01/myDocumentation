# 2. Delta Live Tables Basics

### 2.1 Table vs View Distinction

| Aspect          | Live Table                          | Live View                         |
| --------------- | ----------------------------------- | --------------------------------- |
| Materialization | Persistent Delta table              | Logical view (no storage)         |
| Performance     | Cached; fast queries                | Recomputed on each reference      |
| Typical use     | Data storage, sharing               | Intermediate transformations      |
| Syntax          | `@dlt.table` or `@dlt.create_table` | `@dlt.view` or `@dlt.create_view` |

**Interview note**: Views are cheaper because they don't persist; use them for intermediate transformations.

### 2.2 Streaming Table vs Materialized View

| Aspect                  | Streaming Table                      | Materialized View (LIVE TABLE)                       |
| ----------------------- | ------------------------------------ | ---------------------------------------------------- |
| Processing model        | Incremental (only new data)          | Full recompute (default) or incremental (serverless) |
| State management        | Stateful; maintains checkpoints      | Stateless (batch recompute)                          |
| Latency                 | Low; continuous or frequent triggers | Higher; depends on refresh schedule                  |
| Cost                    | Lower for high-volume ingestion      | Higher for large aggregations (unless serverless)    |
| Schema changes          | Streaming source must append-only    | Schema evolution via DDL supported                   |
| Idempotency requirement | High (deduplicate if replayed)       | Lower (deterministic recompute)                      |

**Key insight**: Streaming tables ≠ always running. They're triggered (continuous mode or scheduled); they maintain checkpoints to track processed data.

### 2.3 Managed vs Unmanaged Tables

- **Managed (default)**: DLT owns storage location; dropped when table dropped
- **Unmanaged**: You specify `location`; survives table drop

```python
@dlt.table(
    name="my_table",
    path="/mnt/external/data/my_table"  # Unmanaged
)
def my_table():
    return spark.read.parquet("...")
```

**Interview note**: Unity Catalog integration prefers managed tables for governance.

### 2.4 Schema Inference and Evolution

DLT auto-infers schema on first run (if not explicitly provided). Evolution:

- **Additive**: New columns appended to schema (safe)
- **Restrictive**: Column removal or type change requires manual intervention

```python
@dlt.table(
    schema="id LONG, name STRING, updated_at TIMESTAMP"  # Explicit schema
)
def customers():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .load("s3://bucket/customers/")
```

**Auto Loader schema evolution modes**:

- `ADDITIVE` (default): Accept new columns
- `FAILONCOLUMNJROPOUT`: Fail if column removed
- `RESCUE`: Unknown fields → `_rescue_data` JSON column

### 2.5 Table Dependencies and Lineage

DLT parses all `dlt.read()` calls to build dependency graph. Circular references cause `CircularDependencyError`.

```python
@dlt.table
def table_a():
    return dlt.read("table_b")  # Depends on B

@dlt.table
def table_b():
    return dlt.read("table_a")  # Depends on A
# ERROR: Circular dependency
```

**Lineage tracking**: DLT UI shows data flow. Event log contains `flow_definition` entries with input/output relationships.
