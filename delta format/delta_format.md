# 1 Delta Lake Interview Preparation Notes

## 1. Delta Lake / Delta Format Introduction

**What is Delta Lake?**

- Open-source storage framework that brings ACID transactions to data lakes
- Built on top of Parquet files with a transaction log
- Not a separate storage system—it's a layer on top of existing object stores (S3, ADLS, GCS, HDFS)
- Provides table abstraction over data lake files

**What is Delta Format?**

- File format specification for storing data with ACID guarantees
- Consists of:
  - **Data files**: Parquet files containing actual data
  - **Transaction log**: JSON files tracking all changes (the `_delta_log` directory)
- Delta Table = Delta Format + APIs to read/write

**Key Point for Interviews:**

- Delta Lake is NOT a database—it's a storage layer
- No separate compute engine required—works with Spark, Presto, Trino, Flink, etc.

---

## 2. Data Storage Evolution

**Traditional Data Warehouse (Pre-2010)**

- Structured data only
- Proprietary formats
- Expensive, limited scalability
- Strong ACID guarantees

**Data Lake Era (2010-2015)**

- Store raw data (structured, semi-structured, unstructured)
- Open formats (Parquet, ORC, Avro)
- Cheap storage (S3, ADLS)
- **Problems:**
  - No ACID transactions
  - No schema enforcement
  - Data quality issues
  - Difficult to maintain consistency
  - Small file problem

**Lakehouse Era (2019-present)**

- Combines data lake flexibility with warehouse reliability
- Delta Lake, Apache Iceberg, Apache Hudi
- ACID transactions + schema enforcement on data lakes
- Delta Lake specifically introduced by Databricks in 2019, open-sourced immediately

**Interview Trap:**

- Be clear: Delta Lake doesn't replace data lakes—it enhances them
- Still uses same underlying storage (S3/ADLS/etc.)

---

## 3. Why Delta Lake?

**Problems Delta Lake Solves:**

1. **No ACID Transactions in Data Lakes**

   - Reading while writing = inconsistent data
   - Failed jobs leave partial writes
   - No rollback mechanism

2. **Data Quality Issues**

   - No schema enforcement
   - Schema evolution without validation
   - Bad data ingestion with no guarantees

3. **Performance Problems**

   - Small file problem (millions of tiny files)
   - No data clustering
   - Full table scans for updates/deletes

4. **Operational Complexity**
   - Manual compaction needed
   - Difficult to handle late-arriving data
   - Time travel not possible
   - No audit trail

**Delta Lake Solutions:**

| Problem            | Delta Solution                              |
| ------------------ | ------------------------------------------- |
| No ACID            | Transaction log with optimistic concurrency |
| Bad data quality   | Schema validation, constraints, enforcement |
| Small files        | Auto-optimize, Z-ordering                   |
| No updates/deletes | MERGE, UPDATE, DELETE support               |
| No time travel     | Version history in transaction log          |
| No audit           | DESCRIBE HISTORY command                    |

---

## 4. Features of Delta Lake

### 4.1 ACID Transactions

- **Atomicity**: All or nothing writes
- **Consistency**: Schema validation before write
- **Isolation**: Serializable isolation level (strongest)
- **Durability**: Once committed, guaranteed to persist

**Code Example:**

```python
# Two concurrent writes - both succeed or both fail
# Writer 1
df1.write.format("delta").mode("append").save("/data/table")

# Writer 2 (concurrent)
df2.write.format("delta").mode("append").save("/data/table")

# Both commits recorded in transaction log atomically
```

### 4.2 Schema Enforcement & Evolution

**Schema Enforcement (default):**

```python
# Initial schema: id (INT), name (STRING)
df1 = spark.createDataFrame([(1, "Alice")], ["id", "name"])
df1.write.format("delta").save("/data/users")

# This FAILS - mismatched schema
df2 = spark.createDataFrame([(2, "Bob", 30)], ["id", "name", "age"])
df2.write.format("delta").mode("append").save("/data/users")
# Error: Schema mismatch detected
```

**Schema Evolution:**

```python
# Allow schema evolution
df2.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .save("/data/users")

# Now schema includes 'age' column (NULL for existing rows)
```

**Interview Point:**

- `mergeSchema=true` adds new columns, doesn't remove or change types
- Type changes require explicit ALTER TABLE

### 4.3 Time Travel (Data Versioning)

**Every write creates a new version:**

```python
# Version 0: Initial data
df1.write.format("delta").save("/data/orders")

# Version 1: Append
df2.write.format("delta").mode("append").save("/data/orders")

# Version 2: Update
spark.sql("UPDATE delta.`/data/orders` SET status='shipped' WHERE id=1")

# Read specific version
df_v0 = spark.read.format("delta").option("versionAsOf", 0).load("/data/orders")
df_v1 = spark.read.format("delta").option("versionAsOf", 1).load("/data/orders")

# Read as of timestamp
df_historical = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-01") \
    .load("/data/orders")
```

**Interview Trap:**

- Versions retained based on retention period (default: 30 days)
- After VACUUM, old versions unreadable
- Time travel uses transaction log, not data file copies

### 4.4 MERGE (UPSERT) Operations

```python
from delta.tables import DeltaTable

# Target table
target = DeltaTable.forPath(spark, "/data/customers")

# Source updates
updates = spark.createDataFrame([
    (1, "Alice Updated", "alice@new.com"),
    (3, "Charlie New", "charlie@new.com")
], ["id", "name", "email"])

# MERGE logic
target.alias("t").merge(
    updates.alias("s"),
    "t.id = s.id"
).whenMatchedUpdate(set={
    "name": "s.name",
    "email": "s.email"
}).whenNotMatchedInsert(values={
    "id": "s.id",
    "name": "s.name",
    "email": "s.email"
}).execute()
```

**SQL Equivalent:**

```sql
MERGE INTO delta.`/data/customers` AS t
USING updates AS s
ON t.id = s.id
WHEN MATCHED THEN UPDATE SET t.name = s.name, t.email = s.email
WHEN NOT MATCHED THEN INSERT (id, name, email) VALUES (s.id, s.name, s.email)
```

### 4.5 DELETE and UPDATE

```python
from delta.tables import DeltaTable

dt = DeltaTable.forPath(spark, "/data/orders")

# DELETE
dt.delete("status = 'cancelled'")

# UPDATE
dt.update(
    condition="order_date < '2023-01-01'",
    set={"status": "'archived'"}
)
```

**SQL:**

```sql
DELETE FROM delta.`/data/orders` WHERE status = 'cancelled';
UPDATE delta.`/data/orders` SET status = 'archived' WHERE order_date < '2023-01-01';
```

**Interview Point:**

- DELETE/UPDATE not in-place—creates new Parquet files
- Old files marked as removed in transaction log
- Requires rewriting affected data files

### 4.6 Unified Batch and Streaming

```python
# Batch write
df_batch.write.format("delta").save("/data/events")

# Streaming write to same table
df_stream = spark.readStream.format("kafka").load()
df_stream.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", "/checkpoints/events") \
    .start("/data/events")

# Streaming read
df_stream_read = spark.readStream.format("delta").load("/data/events")
```

**Interview Point:**

- Same table accessed by batch and streaming—no separate pipelines
- Streaming writes are also ACID
- No "exactly-once" issues with proper checkpointing

### 4.7 Automatic File Management

**OPTIMIZE (Compaction):**

```python
from delta.tables import DeltaTable

dt = DeltaTable.forPath(spark, "/data/sales")

# Compact small files
dt.optimize().executeCompaction()
```

**Z-ORDER (Data Clustering):**

```python
# Z-order by frequently filtered columns
dt.optimize().executeZOrderBy("date", "region")
```

**VACUUM (Cleanup):**

```python
# Remove files older than retention period
dt.vacuum(168)  # 168 hours = 7 days (default: 7 days)
```

**Interview Trap:**

- OPTIMIZE doesn't run automatically (unless Auto-Optimize enabled)
- VACUUM deletes physical files—time travel stops working for vacuumed versions
- Minimum retention: 7 days (safety check to prevent breaking time travel)

### 4.8 Audit History

```sql
DESCRIBE HISTORY delta.`/data/orders`;
```

**Output Example:**

```
version | timestamp           | operation | operationParameters
--------|---------------------|-----------|--------------------
3       | 2024-12-26 10:00:00 | MERGE     | predicate: [id=1]
2       | 2024-12-26 09:00:00 | UPDATE    | predicate: [status='pending']
1       | 2024-12-26 08:00:00 | WRITE     | mode: Append
0       | 2024-12-26 07:00:00 | WRITE     | mode: Overwrite
```

### 4.9 Data Quality Constraints

**NOT NULL:**

```sql
ALTER TABLE delta.`/data/users` ALTER COLUMN email SET NOT NULL;
```

**CHECK Constraints:**

```sql
ALTER TABLE delta.`/data/orders`
ADD CONSTRAINT valid_amount CHECK (amount >= 0);
```

**Code Example:**

```python
# This write will FAIL
df_invalid = spark.createDataFrame([(-100, "order1")], ["amount", "order_id"])
df_invalid.write.format("delta").mode("append").save("/data/orders")
# Error: CHECK constraint valid_amount violated
```

**Interview Point:**

- Constraints enforced at write time
- Constraint violations fail the entire transaction
- Not all SQL constraints supported (e.g., FOREIGN KEY not supported)

---

## 5. Delta Lake Architecture

### 5.1 Physical Storage Layout

**Directory Structure:**

```
/data/users/
├── _delta_log/
│   ├── 00000000000000000000.json    # Version 0
│   ├── 00000000000000000001.json    # Version 1
│   ├── 00000000000000000002.json    # Version 2
│   ├── 00000000000000000010.checkpoint.parquet  # Checkpoint
│   └── _last_checkpoint                         # Pointer to last checkpoint
├── part-00000-<uuid>.snappy.parquet  # Data file 1
├── part-00001-<uuid>.snappy.parquet  # Data file 2
└── part-00002-<uuid>.snappy.parquet  # Data file 3
```

### 5.2 Transaction Log (`_delta_log`)

**Core Concept:**

- Single source of truth for table state
- Ordered sequence of JSON files
- Each file = one atomic transaction
- Version number = sequential commit

**Transaction Log Entry Structure:**

```json
{
  "commitInfo": {
    "timestamp": 1672531200000,
    "operation": "WRITE",
    "operationParameters": { "mode": "Append" },
    "readVersion": 0,
    "isBlindAppend": true
  },
  "add": {
    "path": "part-00000-<uuid>.snappy.parquet",
    "size": 534,
    "partitionValues": {},
    "modificationTime": 1672531200000,
    "dataChange": true,
    "stats": "{\"numRecords\":100,\"minValues\":{\"id\":1},\"maxValues\":{\"id\":100}}"
  }
}
```

**Key Actions in Log:**

- `add`: New file added
- `remove`: File logically deleted (not physically)
- `metaData`: Schema changes
- `protocol`: Protocol version
- `commitInfo`: Transaction metadata

### 5.3 Checkpoints

**Why Checkpoints?**

- Reading 10,000 JSON files = slow
- Checkpoint = snapshot of entire table state at version N
- Parquet format (faster to read)

**Checkpoint Creation:**

- Automatic every 10 commits by default
- Contains all `add` actions (active files) at that version

**Example:**

```
Version 0-9: JSON files
Version 10: Checkpoint created (00000000000000000010.checkpoint.parquet)
Version 11-19: JSON files
Version 20: Checkpoint created
```

**How Table State is Reconstructed:**

1. Read latest checkpoint
2. Replay JSON commits after checkpoint
3. Build current list of active files

**Interview Point:**

- Checkpoint interval configurable: `delta.checkpointInterval`
- Without checkpoints, reads get slower as versions grow
- Checkpoints don't replace JSON files—both coexist

### 5.4 Optimistic Concurrency Control

**How Concurrent Writes Work:**

1. **Writer A** reads current version (v5)
2. **Writer B** reads current version (v5)
3. **Writer A** attempts commit to v6:
   - Checks if anyone committed v6 already → No
   - Writes transaction log file `00000000000000000006.json`
   - Success
4. **Writer B** attempts commit to v6:
   - Checks if anyone committed v6 already → Yes (Writer A did)
   - Checks for conflicts
   - If conflict: Retry from latest version (v6)
   - If no conflict: Write as v7

**Conflict Detection:**

- Two writes conflict if they modify the same data files
- Append-only writes rarely conflict
- UPDATE/DELETE/MERGE on same partitions = conflict

**Code Example - Conflict:**

```python
# Writer 1 and Writer 2 both try to update same partition
# Only one succeeds initially, other retries

# Writer 1
spark.sql("UPDATE delta.`/data/sales` SET status='processed' WHERE date='2024-01-01'")

# Writer 2 (concurrent)
spark.sql("UPDATE delta.`/data/sales` SET amount=amount*1.1 WHERE date='2024-01-01'")

# One commits first, other detects conflict and retries
```

**Interview Trap:**

- "Optimistic" means: assume no conflicts, check at commit time
- NOT pessimistic locking (no table locks during write)
- Retries are automatic, transparent to user

### 5.5 Read and Write Flow

**Write Flow:**

1. Schema validation
2. Write data as Parquet files
3. Compute file statistics (min/max, count)
4. Check for conflicts (read latest version)
5. Write transaction log JSON file
6. Atomic commit by successfully writing log file

**Read Flow (Current Version):**

1. List `_delta_log` directory
2. Find latest checkpoint
3. Read checkpoint (active files at checkpoint version)
4. Replay commits after checkpoint
5. Build final list of active files
6. Read Parquet files from list

**Read Flow (Historical Version):**

1. Replay transaction log from v0 to requested version
2. Build active files list at that version
3. Read those Parquet files

**Interview Point:**

- Reads don't lock table
- Multiple readers can read different versions simultaneously
- "Active files list" = files added but not removed in log

### 5.6 Metadata and Statistics

**File-level Statistics (in Transaction Log):**

```json
"stats": {
  "numRecords": 1000,
  "minValues": {"id": 1, "date": "2024-01-01"},
  "maxValues": {"id": 1000, "date": "2024-12-31"},
  "nullCount": {"id": 0, "date": 0}
}
```

**Usage:**

- Data skipping: Skip files not matching query predicates
- Example: `SELECT * FROM table WHERE id = 500`
  - Delta reads stats, skips files where max(id) < 500 or min(id) > 500

**Interview Point:**

- Stats collected automatically during write
- Only for first 32 columns by default
- Critical for query performance—enables data skipping without reading files

### 5.7 Data Skipping

**How It Works:**

1. Query: `SELECT * FROM orders WHERE date = '2024-01-15'`
2. Delta reads transaction log stats
3. Skips files where:
   - `max(date) < '2024-01-15'` OR
   - `min(date) > '2024-01-15'`
4. Only reads files where `'2024-01-15'` falls in `[min, max]` range

**Z-Ordering Impact:**

- Co-locates similar values in same files
- Improves data skipping effectiveness
- Example: Z-order by `date` → all records for same date mostly in same file

**Code Example:**

```python
# Without Z-order: 100 files, each has mixed dates → reads many files
# With Z-order by date: 100 files, each has narrow date range → reads few files

dt = DeltaTable.forPath(spark, "/data/orders")
dt.optimize().executeZOrderBy("date")

# Query now reads fewer files
spark.sql("SELECT * FROM delta.`/data/orders` WHERE date = '2024-01-15'")
```

---

## Key Interview Points Summary

1. **Delta Lake = Parquet + Transaction Log**

   - Not a separate storage system
   - Transaction log in `_delta_log` is the single source of truth

2. **ACID via Optimistic Concurrency**

   - No locks during write
   - Conflict detection at commit time
   - Automatic retries on conflict

3. **Time Travel via Versions**

   - Every commit = new version
   - Retained for 30 days by default
   - VACUUM deletes old data files

4. **Schema Enforcement vs Evolution**

   - Default: strict enforcement
   - `mergeSchema=true`: allow evolution
   - Constraints enforced at write time

5. **MERGE/UPDATE/DELETE Not In-Place**

   - Rewrite affected files
   - Mark old files as removed in log
   - VACUUM deletes physical files later

6. **Performance Features**

   - Data skipping via file statistics
   - Z-ordering for better clustering
   - Checkpoints for faster reads
   - OPTIMIZE for small file compaction

7. **Common Pitfall:**
   - VACUUM before retention period → breaks time travel
   - Forgetting to OPTIMIZE → small file problem
   - Not Z-ordering frequently filtered columns → poor data skipping
