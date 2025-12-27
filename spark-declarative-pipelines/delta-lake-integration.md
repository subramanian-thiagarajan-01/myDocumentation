# 9. Delta Lake Integration

### 9.1 ACID Transactions & Time Travel

DLT writes to Delta tables, inheriting ACID guarantees:

- **Atomicity**: Entire table version succeeds or fails
- **Consistency**: No partial updates visible
- **Isolation**: Snapshot isolation; concurrent readers see consistent snapshot
- **Durability**: Transaction committed to storage

Time travel:

```sql
SELECT * FROM my_table TIMESTAMP AS OF '2025-12-27 10:00:00'
SELECT * FROM my_table VERSION AS OF 10
```

### 9.2 Optimize and Z-Ordering

DLT does **not** automatically call `OPTIMIZE`. Manual maintenance required:

```sql
OPTIMIZE my_table ZORDER BY (customer_id, order_date)
```

Z-ordering clusters data by columns for faster pruning.

### 9.3 Vacuum

Remove old data files (default: 7-day retention):

```sql
VACUUM my_table RETAIN 7 DAYS
```

**Critical**: Cannot vacuum data referenced by time-travel queries. Retention period protects concurrent readers.

### 9.4 Schema Enforcement vs Evolution

**Enforcement**: Strict mode; new columns rejected

```sql
ALTER TABLE my_table SET TBLPROPERTIES ('delta.columnMapping.mode' = 'none')
```

**Evolution**: New columns appended (default in DLT)

```sql
ALTER TABLE my_table SET TBLPROPERTIES ('delta.schema.autoMerge.enabled' = 'true')
```

### 9.5 MERGE Operations in DLT

```python
@dlt.table
def customer_gold():
    updates = dlt.read("customer_silver_updates")
    gold = dlt.read("customer_gold_current")

    # DLT does not expose Spark merge directly; must use spark.sql()
    return spark.sql("""
        MERGE INTO gold USING updates
        ON gold.customer_id = updates.customer_id
        WHEN MATCHED THEN UPDATE SET *
        WHEN NOT MATCHED THEN INSERT *
    """)
```

**Limitation**: DLT tables read-only within DLT. MERGE requires raw Spark SQL outside DLT or within `spark.sql()`.
