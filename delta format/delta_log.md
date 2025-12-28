# 2. Delta Lake Transaction Log & ACID Implementation

## Delta Log Overview

**What it is**: Ordered record of every transaction ever performed on a Delta table. Stored in `_delta_log/` subdirectory.

**Purpose**:

- Single source of truth for table state
- Enables ACID guarantees
- Powers time travel
- Coordinates concurrent readers/writers

---

## Delta Log File Structure

### Directory Layout

```
my_delta_table/
├── part-00000-xxx.parquet
├── part-00001-xxx.parquet
└── _delta_log/
    ├── 00000000000000000000.json
    ├── 00000000000000000001.json
    ├── 00000000000000000002.json
    ├── 00000000000000000002.crc
    ├── 00000000000000000003.json
    ├── ...
    ├── 00000000000000000010.checkpoint.parquet
    └── _last_checkpoint
```

**Key components**:

- **JSON files**: One per transaction (version), named with 20-digit zero-padded version number
- **CRC files**: Checksum for JSON integrity validation
- **Checkpoint files**: Aggregated state snapshots every N commits (default: 10)
- **`_last_checkpoint`**: Metadata file pointing to latest checkpoint

---

## Delta Log Files Explained

### 1. JSON Transaction Files (`.json`)

**Content**: Single transaction's actions encoded as newline-delimited JSON.

**Example**: Creating a table with 2 files

```python
# Create Delta table
from pyspark.sql.types import *

df = spark.createDataFrame([
    (1, "Alice", 25),
    (2, "Bob", 30),
    (3, "Charlie", 35)
], ["id", "name", "age"])

df.write.format("delta").mode("overwrite").save("/tmp/users")
```

**Resulting `00000000000000000000.json`**:

```json
{"commitInfo":{"timestamp":1703001234567,"operation":"WRITE","operationParameters":{"mode":"Overwrite","partitionBy":"[]"},"isBlindAppend":false}}
{"protocol":{"minReaderVersion":1,"minWriterVersion":2}}
{"metaData":{"id":"abc-123-def","format":{"provider":"parquet","options":{}},"schemaString":"{\"type\":\"struct\",\"fields\":[{\"name\":\"id\",\"type\":\"long\",\"nullable\":true,\"metadata\":{}},{\"name\":\"name\",\"type\":\"string\",\"nullable\":true,\"metadata\":{}},{\"name\":\"age\",\"type\":\"long\",\"nullable\":true,\"metadata\":{}}]}","partitionColumns":[],"configuration":{},"createdTime":1703001234000}}
{"add":{"path":"part-00000-xxx.parquet","partitionValues":{},"size":1234,"modificationTime":1703001234000,"dataChange":true,"stats":"{\"numRecords\":2,\"minValues\":{\"id\":1,\"age\":25},\"maxValues\":{\"id\":2,\"age\":30},\"nullCount\":{\"id\":0,\"name\":0,\"age\":0}}"}}
{"add":{"path":"part-00001-xxx.parquet","partitionValues":{},"size":987,"modificationTime":1703001234000,"dataChange":true,"stats":"{\"numRecords\":1,\"minValues\":{\"id\":3,\"age\":35},\"maxValues\":{\"id\":3,\"age\":35},\"nullCount\":{\"id\":0,\"name\":0,\"age\":0}}"}}
```

**Action Types**:

| Action           | Purpose                                                   |
| ---------------- | --------------------------------------------------------- |
| `commitInfo`     | Transaction metadata (timestamp, operation, user, params) |
| `protocol`       | Reader/writer version requirements                        |
| `metaData`       | Schema, partition columns, table properties, table ID     |
| `add`            | File added to table                                       |
| `remove`         | File removed from table (logical deletion)                |
| `txn`            | Application transaction ID for idempotency                |
| `cdc`            | Change Data Capture records                               |
| `domainMetadata` | Deletion vectors, column mapping, etc.                    |

---

### 2. CRC Files (`.crc`)

**Purpose**: Validate JSON file integrity.

**When created**: After JSON commit file is written.

**Content**: Checksum of the JSON file.

**Example**: `00000000000000000002.crc`

```
{"crc":1234567890}
```

**Validation**: On read, Delta recomputes JSON checksum and compares with CRC. Mismatch = corrupted transaction log.

**Note**: Not always present in cloud storage (S3, ADLS). Delta handles gracefully.

---

### 3. Checkpoint Files (`.checkpoint.parquet`)

**Purpose**: Avoid replaying entire log history. Aggregate all table state up to version N.

**When created**: Every 10 commits by default (configurable via `delta.checkpointInterval`).

**Structure**: Parquet file containing all active `add` actions, latest `metaData`, and `protocol`.

**Example**: After 10 commits, `00000000000000000010.checkpoint.parquet` created.

**Checkpoint content**:

- All `add` actions for current table files
- Current `metaData`
- Current `protocol`
- No `remove`, `commitInfo` (not needed for state reconstruction)

**Multi-part checkpoints**: For large tables, checkpoint split into multiple files:

```
00000000000000000010.checkpoint.0000000001.0000000010.parquet
00000000000000000010.checkpoint.0000000002.0000000010.parquet
...
00000000000000000010.checkpoint.0000000010.0000000010.parquet
```

---

### 4. `_last_checkpoint` File

**Purpose**: Quick discovery of latest checkpoint.

**Content**: JSON with checkpoint version and file count.

**Example**:

```json
{ "version": 10, "size": 5, "parts": 1 }
```

**Usage**: Readers start here to find checkpoint, then replay subsequent JSON commits.

---

## How Delta Ensures ACID Properties

### Atomicity

**Mechanism**: Optimistic concurrency control + atomic file operations.

**Process**:

1. Writer reads latest version from log
2. Performs operation (e.g., writes new Parquet files)
3. Creates JSON commit file for version `N+1`
4. Attempts atomic write (PUT-if-absent in S3, conditional create in ADLS/HDFS)
5. **If write succeeds**: Transaction committed
6. **If write fails**: Another writer committed first. Retry with conflict resolution.

**Key**: JSON file write is atomic. Either version N exists or doesn't—no partial state.

**Code Example - Concurrent Writes**:

```python
# Writer 1
df1 = spark.range(0, 10)
df1.write.format("delta").mode("append").save("/tmp/concurrent_test")

# Writer 2 (simultaneous)
df2 = spark.range(10, 20)
df2.write.format("delta").mode("append").save("/tmp/concurrent_test")
```

**Result**: Both succeed (append is commutative) or one retries. No data loss, no partial commits.

---

### Consistency

**Mechanism**: Single-version snapshot isolation.

**Guarantee**: Readers see complete snapshot at version N. Never see partial transaction.

**Example**:

```python
# Write version 0
spark.range(10).write.format("delta").save("/tmp/consistent")

# Read always sees complete version 0
df = spark.read.format("delta").load("/tmp/consistent")
df.count()  # Always 10, never partial
```

**Implementation**:

- Reader lists `_delta_log/`, finds latest version (or checkpoint)
- Constructs file list from `add` actions at that version
- Reads only those files
- Concurrent writes to version N+1 invisible until reader explicitly refreshes

---

### Isolation

**Level**: Serializable (strongest isolation).

**Mechanism**: Write conflict detection.

**Conflict types**:

| Scenario                                    | Conflict? | Resolution                                |
| ------------------------------------------- | --------- | ----------------------------------------- |
| Concurrent appends to non-partitioned table | No        | Both succeed                              |
| Concurrent appends to disjoint partitions   | No        | Both succeed                              |
| Concurrent appends to same partition        | Yes       | One retries, checks for logical conflicts |
| Update + Delete on same records             | Yes       | One retries                               |
| Schema evolution + Write                    | Yes       | One retries                               |

**Code Example - Conflict Detection**:

```python
from delta.tables import DeltaTable

# Create table
spark.range(10).withColumn("value", expr("id * 10")).write.format("delta").save("/tmp/conflict_test")

# Transaction 1: Update
dt1 = DeltaTable.forPath(spark, "/tmp/conflict_test")
dt1.update(condition="id < 5", set={"value": "999"})

# Transaction 2 (concurrent): Delete same rows
dt2 = DeltaTable.forPath(spark, "/tmp/conflict_test")
dt2.delete("id < 5")
```

**Behavior**: Second transaction detects conflict (overlapping predicate/partition), retries.

**Behind the scenes**:

1. Both read version N
2. Both write new Parquet files
3. Transaction 1 writes version N+1 successfully
4. Transaction 2 attempts N+1, **fails** (file exists)
5. Transaction 2 reads N+1, detects logical conflict, retries as version N+2

---

### Durability

**Mechanism**: Write-ahead log (Delta Log) committed before data visible.

**Process**:

1. Write new Parquet files to table directory
2. Write JSON commit file
3. **Only after JSON commit succeeds**, transaction visible

**Guarantee**: If JSON commit exists, data files exist. If system crashes before JSON write, data files ignored (orphaned).

**Example**:

```python
df = spark.range(1000000)
df.write.format("delta").mode("append").save("/tmp/durable")

# If crash happens:
# - Before JSON write: No version created, data files orphaned
# - After JSON write: Version exists, data guaranteed accessible
```

**Cleanup**: Orphaned files removed by `VACUUM` command.

---

## Schema Evolution, Enforcement, and Overwrite

### Schema Enforcement (Default Behavior)

**Rule**: Write schema must match existing table schema. Type mismatches or missing columns rejected.

**Example - Schema Enforcement Failure**:

```python
# Create table
spark.range(10).withColumn("name", lit("test")).write.format("delta").save("/tmp/schema_enforce")

# Try to append with different schema
df_bad = spark.range(10).withColumn("age", lit(25))  # Different column
df_bad.write.format("delta").mode("append").save("/tmp/schema_enforce")
```

**Error**:

```
AnalysisException: A schema mismatch detected when writing to the Delta table.
To enable schema migration, please set: '.option("mergeSchema", "true")'
```

**Enforcement at**: Write time, before Parquet files written.

---

### Schema Evolution

**Enable with**: `.option("mergeSchema", "true")`

**Behavior**: Add new columns, compatible type widening.

**Allowed changes**:

- Add new columns (nullable by default)
- Type widening (e.g., int → long, float → double)

**Not allowed**:

- Drop columns (use `ALTER TABLE DROP COLUMN`)
- Incompatible type changes (e.g., string → int)
- Change nullability (nullable → not null)

**Example - Add Column**:

```python
# Initial table
df1 = spark.createDataFrame([(1, "Alice"), (2, "Bob")], ["id", "name"])
df1.write.format("delta").save("/tmp/schema_evolve")

# Append with new column
df2 = spark.createDataFrame([(3, "Charlie", 30)], ["id", "name", "age"])
df2.write.format("delta").mode("append").option("mergeSchema", "true").save("/tmp/schema_evolve")

# Read
result = spark.read.format("delta").load("/tmp/schema_evolve")
result.show()
```

**Output**:

```
+---+-------+----+
| id|   name| age|
+---+-------+----+
|  1|  Alice|null|
|  2|    Bob|null|
|  3|Charlie|  30|
+---+-------+----+
```

**Delta Log Change**:

Version 1 JSON includes:

```json
{"metaData":{"schemaString":"{\"type\":\"struct\",\"fields\":[{\"name\":\"id\",\"type\":\"long\"},{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"age\",\"type\":\"long\"}]}",...}}
```

**Old data files**: Retain old schema. Delta fills missing `age` with `null` on read.

---

### Schema Overwrite Modes

**1. `overwriteSchema` = true**

**Usage**: Replace entire schema.

```python
# Original schema
df1 = spark.createDataFrame([(1, "Alice")], ["id", "name"])
df1.write.format("delta").save("/tmp/schema_overwrite")

# Overwrite schema completely
df2 = spark.createDataFrame([(100, 25)], ["user_id", "age"])
df2.write.format("delta").mode("overwrite").option("overwriteSchema", "true").save("/tmp/schema_overwrite")

# New schema: [user_id, age], old data removed
spark.read.format("delta").load("/tmp/schema_overwrite").show()
```

**Output**:

```
+-------+---+
|user_id|age|
+-------+---+
|    100| 25|
+-------+---+
```

**Use case**: Complete table restructure.

---

**2. `replaceWhere` (Partial Overwrite)**

**Usage**: Overwrite specific partitions while preserving schema and other partitions.

```python
# Partitioned table
df1 = spark.createDataFrame([
    (1, "2023-01-01", 100),
    (2, "2023-01-02", 200),
    (3, "2023-01-03", 300)
], ["id", "date", "amount"])

df1.write.format("delta").partitionBy("date").save("/tmp/replace_where")

# Overwrite only 2023-01-02 partition
df2 = spark.createDataFrame([
    (4, "2023-01-02", 999)
], ["id", "date", "amount"])

df2.write.format("delta").mode("overwrite") \
    .option("replaceWhere", "date = '2023-01-02'") \
    .save("/tmp/replace_where")

spark.read.format("delta").load("/tmp/replace_where").orderBy("id").show()
```

**Output**:

```
+---+----------+------+
| id|      date|amount|
+---+----------+------+
|  1|2023-01-01|   100|
|  4|2023-01-02|   999|  # Overwritten
|  3|2023-01-03|   300|
+---+----------+------+
```

**Delta Log**: Adds `remove` actions for old 2023-01-02 files, `add` actions for new file.

---

## Schema Changes in Delta Log

### Table Property: `delta.columnMapping.mode`

**Purpose**: Allow column renames, drops, type changes without rewriting data.

**Modes**:

- `none` (default): Physical column names must match schema
- `name`: Use field IDs, allow renames
- `id`: Full column mapping with IDs

**Example - Column Rename with Mapping**:

```python
# Enable column mapping
spark.sql("""
    CREATE TABLE mapping_test (id LONG, old_name STRING)
    USING delta
    TBLPROPERTIES ('delta.columnMapping.mode' = 'name')
""")

spark.sql("INSERT INTO mapping_test VALUES (1, 'Alice'), (2, 'Bob')")

# Rename column
spark.sql("ALTER TABLE mapping_test RENAME COLUMN old_name TO new_name")

spark.sql("SELECT * FROM mapping_test").show()
```

**Output**:

```
+---+--------+
| id|new_name|
+---+--------+
|  1|   Alice|
|  2|     Bob|
+---+--------+
```

**Delta Log**: Schema updated, but Parquet files unchanged (field ID mapping used).

---

## How Delta Operations Work Internally

### INSERT / APPEND

**Process**:

1. Write new Parquet files
2. Create JSON commit with `add` actions
3. No `remove` actions (pure append)

**Example**:

```python
df = spark.range(10)
df.write.format("delta").mode("append").save("/tmp/insert_test")
```

**Delta Log Version N**:

```json
{"commitInfo":{"operation":"WRITE","mode":"Append","isBlindAppend":true}}
{"add":{"path":"part-00000-xxx.parquet","size":1234,"dataChange":true}}
{"add":{"path":"part-00001-xxx.parquet","size":987,"dataChange":true}}
```

**Key**: `isBlindAppend=true` means no reads required, conflict-free.

---

### UPDATE

**Process**:

1. **Read phase**: Scan files matching `WHERE` clause (using stats)
2. **Rewrite phase**: Rewrite affected files with updated rows
3. **Commit phase**: Add `remove` for old files, `add` for new files

**Example**:

```python
from delta.tables import DeltaTable

# Create table
spark.range(100).withColumn("value", expr("id * 10")).write.format("delta").save("/tmp/update_test")

# Update
dt = DeltaTable.forPath(spark, "/tmp/update_test")
dt.update(condition="id >= 50", set={"value": "999"})
```

**Delta Log**:

```json
{"commitInfo":{"operation":"UPDATE","operationMetrics":{"numAddedFiles":"20","numRemovedFiles":"20","numUpdatedRows":"50"}}}
{"remove":{"path":"part-00000-xxx.parquet","deletionTimestamp":1703001234567}}
{"remove":{"path":"part-00001-xxx.parquet","deletionTimestamp":1703001234567}}
...
{"add":{"path":"part-00000-yyy.parquet","size":1234,"dataChange":true}}
{"add":{"path":"part-00001-yyy.parquet","size":987,"dataChange":true}}
...
```

**Optimization**: If predicate matches entire file, skipped (no rewrite). If matches no rows, skipped.

**Performance**:

- **Best case**: Partition pruning + Z-ordering reduce files scanned
- **Worst case**: Full table scan if no partition/stats pruning

---

### DELETE

**Two modes**: **Copy-on-Write** (default) vs. **Deletion Vectors** (Delta 2.0+).

#### Copy-on-Write DELETE

**Process**:

1. Scan files matching `WHERE` clause
2. Rewrite files, excluding deleted rows
3. Commit `remove` (old) + `add` (rewritten)

**Example**:

```python
dt = DeltaTable.forPath(spark, "/tmp/delete_test")
dt.delete("id < 10")
```

**Delta Log**:

```json
{"commitInfo":{"operation":"DELETE","operationMetrics":{"numDeletedRows":"10","numRemovedFiles":"5","numAddedFiles":"5"}}}
{"remove":{"path":"part-00000-xxx.parquet"}}
{"add":{"path":"part-00000-rewritten.parquet","size":800}}
```

**Cost**: Proportional to data rewritten (even if deleting 1 row).

---

#### Deletion Vectors (Preferred for Small Deletes)

**What**: Bitmap index marking deleted rows. No file rewrite.

**Enable**:

```python
spark.sql("""
    ALTER TABLE my_table
    SET TBLPROPERTIES ('delta.enableDeletionVectors' = 'true')
""")
```

**Process**:

1. Scan files matching `WHERE` clause
2. Generate deletion vector (RoaringBitmap) marking deleted row positions
3. Write DV file (`.bin` format)
4. Commit `add` action with `deletionVector` metadata—no `remove`

**Example**:

```python
# Enable DVs
spark.sql("CREATE TABLE dv_test (id LONG, name STRING) USING delta TBLPROPERTIES ('delta.enableDeletionVectors' = 'true')")
spark.sql("INSERT INTO dv_test VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Charlie')")

# Delete with DV
spark.sql("DELETE FROM dv_test WHERE id = 2")
```

**Delta Log**:

```json
{"commitInfo":{"operation":"DELETE","operationMetrics":{"numDeletionVectorsAdded":"1","numDeletionVectorsRemoved":"0"}}}
{"add":{"path":"part-00000-xxx.parquet","deletionVector":{"storageType":"u","pathOrInlineDv":"ab3efd67...","offset":1,"sizeInBytes":45,"cardinality":1},"dataChange":false}}
```

**`deletionVector` fields**:

- `storageType`: `u` (UUID path) or `i` (inline)
- `pathOrInlineDv`: DV file path or inline bitmap
- `cardinality`: Number of deleted rows
- `dataChange=false`: No new data, only metadata

**Deletion Vector File** (`deletion_vector_ab3efd67-xxxx-xxxx-xxxx-xxxxxxxxxxxx.bin`):

Binary RoaringBitmap with deleted row indices (e.g., `[1]` for row index 1).

**Read behavior**: Readers load DV, skip deleted rows.

**When to use**:

- **DVs**: Small deletes (<10% of file), frequent deletes
- **Copy-on-Write**: Large deletes, infrequent deletes, already rewriting

**Compaction**: `OPTIMIZE` merges DVs into rewritten files.

```python
spark.sql("OPTIMIZE dv_test")
```

**Result**: DV files removed, data files rewritten without deleted rows.

---

### MERGE (Upsert)

**Process**: Combines `UPDATE`, `INSERT`, `DELETE` in single transaction.

**Phases**:

1. **Match phase**: Join source with target using `ON` condition
2. **Action phase**: Execute matched/not-matched clauses
3. **Rewrite phase**: Rewrite affected files
4. **Commit phase**: Atomic commit with mixed `add`/`remove` actions

**Example**:

```python
# Target table
target = spark.createDataFrame([
    (1, "Alice", 25),
    (2, "Bob", 30)
], ["id", "name", "age"])
target.write.format("delta").save("/tmp/merge_target")

# Source data
source = spark.createDataFrame([
    (2, "Bob", 35),      # Update
    (3, "Charlie", 40)   # Insert
], ["id", "name", "age"])

# MERGE
from delta.tables import DeltaTable
target_dt = DeltaTable.forPath(spark, "/tmp/merge_target")

target_dt.alias("target").merge(
    source.alias("source"),
    "target.id = source.id"
).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()

spark.read.format("delta").load("/tmp/merge_target").show()
```

**Output**:

```
+---+-------+---+
| id|   name|age|
+---+-------+---+
|  1|  Alice| 25|  # Unchanged
|  2|    Bob| 35|  # Updated
|  3|Charlie| 40|  # Inserted
+---+-------+---+
```

**Delta Log**:

```json
{"commitInfo":{"operation":"MERGE","operationMetrics":{"numTargetRowsUpdated":"1","numTargetRowsInserted":"1","numTargetFilesAdded":"2","numTargetFilesRemoved":"1"}}}
{"remove":{"path":"part-00000-old.parquet"}}  # File containing id=2
{"add":{"path":"part-00000-rewritten.parquet","stats":"..."}}  # Updated id=2
{"add":{"path":"part-00001-new.parquet","stats":"..."}}  # Inserted id=3
```

---

## Key Interview Takeaways

### What Interviewers Test

1. **Log replay mechanism**: How readers construct table state from JSON + checkpoints
2. **Conflict resolution**: Write-write conflicts, retry logic
3. **File-level operations**: No row-level deletes in Parquet—always file rewrites (or DVs)
4. **Checkpoint usage**: Why needed, when created, how reduces read latency
5. **Schema evolution pitfalls**: `mergeSchema` vs. `overwriteSchema`, backward compatibility
6. **Deletion Vectors**: Trade-offs vs. copy-on-write, when to optimize
7. **ACID guarantees**: Atomicity via file atomicity, isolation via versioning, durability via WAL

### Common Pitfalls

- **Checkpoint lag**: If checkpoints disabled, readers replay 1000s of commits—slow reads
- **Small files problem**: Frequent appends create many small files—use `OPTIMIZE`
- **DV accumulation**: Many deletes create many DV files—run `OPTIMIZE` periodically
- **Schema evolution without `mergeSchema`**: Fails silently if schema mismatch not caught
- **Concurrent overwrites**: Two `mode("overwrite")` writers can conflict—use `replaceWhere` for partitions

### Quick Wins for Interviews

- **Know JSON actions**: `add`, `remove`, `metaData`, `protocol`, `txn`, `cdc`, `domainMetadata`
- **Explain checkpoint necessity**: "Avoids O(N) log replay for readers"
- **Articulate DV benefits**: "Efficient small deletes, deferred file rewrites, compaction required"
- **Describe conflict detection**: "Optimistic concurrency—writers check overlapping predicates/partitions on retry"
- **Schema evolution rules**: "Add columns, type widening allowed; drop/incompatible changes require explicit ALTER TABLE"

---

## Additional Code: Inspecting Delta Log

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col

# Read transaction log as table
dt = DeltaTable.forPath(spark, "/tmp/my_table")

# Show table history (commitInfo from all versions)
dt.history().select("version", "timestamp", "operation", "operationMetrics").show(truncate=False)

# Read specific version's JSON
log_df = spark.read.json("/tmp/my_table/_delta_log/00000000000000000005.json")
log_df.show(truncate=False)

# List all files at version 3
files_v3 = spark.read.format("delta").option("versionAsOf", 3).load("/tmp/my_table").inputFiles()
print(files_v3)

# Inspect deletion vectors (Delta 2.0+)
dt.detail().select("numFiles", "properties").show(truncate=False)
```

**Output Example**:

```
+-------+-------------------+---------+----------------------------------------------------+
|version|timestamp          |operation|operationMetrics                                    |
+-------+-------------------+---------+----------------------------------------------------+
|0      |2023-12-01 10:00:00|WRITE    |{numFiles: 2, numOutputBytes: 1234, ...}            |
|1      |2023-12-01 10:05:00|UPDATE   |{numUpdatedRows: 50, numAddedFiles: 2, ...}         |
|2      |2023-12-01 10:10:00|DELETE   |{numDeletedRows: 10, numDeletionVectorsAdded: 1}   |
+-------+-------------------+---------+----------------------------------------------------+
```
