# 8. Change Data Capture (CDC)

### 8.1 APPLY CHANGES INTO (SCD Type 1 & 2)

```python
import dlt
from pyspark.sql.functions import col, expr

@dlt.create_streaming_table(name="customer_cdc_stream")
def customer_cdc_stream():
    return spark.readStream.format("cloudFiles") \
        .option("cloudFiles.format", "json") \
        .load("s3://cdc-stream/customers/")

# SCD Type 1: Overwrite historical values
@dlt.apply_changes(
    target="customers_scd1",
    source="customer_cdc_stream",
    keys=["customer_id"],
    sequence_by="change_timestamp",
    ignore_null_updates=False
)
def apply_scd1():
    pass  # Function body empty; @dlt.apply_changes handles the logic

# SCD Type 2: Maintain historical records
@dlt.apply_changes(
    target="customers_scd2",
    source="customer_cdc_stream",
    keys=["customer_id"],
    sequence_by="change_timestamp",
    stored_as_scd_type=2,
    ignore_null_updates=False
)
def apply_scd2():
    pass
```

### 8.2 Handling Deletes and Truncates

```python
from pyspark.sql.functions import col, expr

@dlt.apply_changes(
    target="customers_scd1",
    source="customer_cdc_stream",
    keys=["customer_id"],
    sequence_by="change_timestamp",
    apply_as_deletes=expr("change_type = 'DELETE'"),  # Condition for row deletion
    apply_as_truncates=expr("change_type = 'TRUNCATE'")  # Condition for table truncate
)
def apply_with_deletes():
    pass
```

**Important**: `apply_as_deletes` applies at row level; `apply_as_truncates` clears entire table.

### 8.3 Out-of-Sequence Record Handling

DLT automatically deduplicates out-of-order updates using `sequence_by` column:

```python
# Stream receives: [customer_id=1, ts=100], [customer_id=1, ts=50], [customer_id=1, ts=75]
# DLT processes in order of ts, ignoring ts=50 (earlier than 100)
@dlt.apply_changes(
    target="customers",
    source="cdc_stream",
    keys=["customer_id"],
    sequence_by="event_timestamp"  # Must be monotonic per key
)
```

**Non-monotonic sequences**: If `sequence_by` values jump backward (clock skew, late-arriving updates), DLT ignores the out-of-sequence record.
