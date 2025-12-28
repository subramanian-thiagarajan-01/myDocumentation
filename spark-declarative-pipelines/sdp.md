# Databricks Declarative Pipelines - Interview Preparation Notes

**Knowledge Current Through:** December 2025

---

## 6.1 SDP Overview

### What is SDP?

**Lakeflow Spark Declarative Pipelines (SDP)** is a declarative framework for building batch and streaming data pipelines in SQL and Python.

**Key characteristics:**

- Runs on performance-optimized Databricks Runtime
- Extends and is interoperable with Apache Spark Declarative Pipelines (available in Spark 4.1+)
- Code written for open-source Apache Spark pipelines runs without modification on Databricks
- Formerly known as Delta Live Tables (DLT) — **no migration required** for existing DLT code
- Requires **Premium plan** on Databricks

**Relationship to DLT:**

- DLT was rebranded/evolved into Lakeflow SDP
- Old `dlt` module → new `pyspark.pipelines` module (alias `dp`)
- Existing DLT pipelines run seamlessly within Lakeflow SDP
- Classic SKUs still begin with "DLT" prefix
- Event log schemas with "dlt" name unchanged

### Why SDP?

**Core benefits over manual Spark/Structured Streaming:**

1. **Automatic Orchestration**

   - Analyzes dependencies automatically
   - Determines optimal execution order
   - Maximizes parallelism for performance
   - Hierarchical retry logic: Spark task → Flow → Pipeline
   - No manual orchestration via Lakeflow Jobs required

2. **Declarative Processing**

   - Reduces hundreds/thousands of lines to just a few
   - Focus on **what** to compute, not **how**
   - Built-in best practices

3. **Incremental Processing**

   - Materialized views process only new data/changes when possible
   - Eliminates manual incremental processing code
   - Automatic watermark management (no manual watermark configuration needed)

4. **Simplified CDC**
   - AUTO CDC API handles out-of-order events automatically
   - Supports SCD Type 1 and Type 2 out-of-the-box
   - No complex merge logic required

**Common use cases:**

- Incremental data ingestion from cloud storage (S3, ADLS Gen2, GCS)
- Message bus ingestion (Kafka, Kinesis, EventHub, Pub/Sub, Pulsar)
- Incremental batch and streaming transformations
- Real-time stream processing between message buses and databases

### SDP Package Introduction

**Python Module:**

```python
from pyspark import pipelines as dp
```

**Critical constraints:**

- `pyspark.pipelines` module only available **within pipeline execution context**
- Cannot be used in notebooks/scripts outside pipelines
- Code is evaluated **multiple times** during planning and execution

**SQL Interface:**

- New keywords: `CREATE OR REFRESH`, `STREAM`, `CONSTRAINT EXPECT`
- Syntax: `CREATE OR REFRESH STREAMING TABLE` / `MATERIALIZED VIEW`

**Architecture:**

- Pipelines separate dataset definitions from update processing
- Not intended for interactive execution
- Source files stored in Databricks workspace or synced from local IDE

**Databricks-only features (not in Apache Spark):**

- AUTO CDC flows
- AUTO CDC FROM SNAPSHOT
- Sinks (Kafka, EventHub, custom)
- ForEachBatch sink
- Enhanced expectation actions

---

## 6.2 Core Components

### Pipelines

**Definition:**
A pipeline is the **unit of development and execution** in SDP.

**Contains:**

- Flows (streaming and batch)
- Streaming tables
- Materialized views
- Sinks

**Execution mechanics:**

1. Starts a cluster with correct configuration
2. Discovers all defined tables/views
3. Checks for analysis errors (invalid columns, missing dependencies, syntax)
4. Creates or updates tables/views with latest data

**Pipeline source code:**

- Python or SQL (can mix both in one pipeline)
- Each file contains only one language
- Evaluated to build dataflow graph **before** query execution
- Order in files defines **evaluation order**, not **execution order**

**Configuration categories:**

1. Source code collection definition
2. Infrastructure, dependencies, update processing, table storage

**Critical configurations:**

- **Target schema** (Hive metastore) or **catalog + schema** (Unity Catalog) — required for external data access
- Storage location
- Compute settings
- Dependency management

**Key interview insight:**

- Pipeline orchestration is **automatic** — SDP analyzes dependencies and parallelizes optimally
- Unlike Lakeflow Jobs, no manual task dependencies required

### Flows

**Definition:**
A flow is the **foundational data processing concept** in SDP — reads data from source, applies transformations, writes to target.

**Flow types:**

| Flow Type             | Semantics | Target                  | Use Case                       |
| --------------------- | --------- | ----------------------- | ------------------------------ |
| **Append**            | Streaming | Streaming table or sink | Continuous streaming ingestion |
| **AUTO CDC**          | Streaming | Streaming table only    | CDC with SCD Type 1/2          |
| **Materialized View** | Batch     | Materialized view       | Incremental batch processing   |

**Append Flow characteristics:**

- Uses Spark Structured Streaming append mode
- Writes to streaming tables or sinks
- Can be defined explicitly (separate from target) or implicitly (within table definition)
- Supports `once=True` parameter for one-time backfills

**AUTO CDC Flow characteristics:**

- **Databricks-only** (not in Apache Spark)
- Handles out-of-order events automatically
- Requires sequencing column (monotonically increasing)
- Only writes to streaming tables

**Materialized View Flow characteristics:**

- Batch semantics
- Incremental processing engine — processes only new data/changes
- Always defined implicitly within materialized view definition
- No separate flow object

**Flow retry logic:**

1. Spark task level (most granular)
2. Flow level
3. Pipeline level (entire pipeline)

**Python example - Explicit append flow:**

```python
from pyspark import pipelines as dp

dp.create_streaming_table("bronze_table")

@dp.append_flow(
    target="bronze_table",
    name="ingestion_flow"
)
def ingest_data():
    return (
        spark.readStream
        .format("cloudFiles")
        .option("cloudFiles.format", "json")
        .load("/path/to/data")
    )
```

**Python example - Implicit flow in streaming table:**

```python
@dp.table(
    name="silver_table",
    comment="Cleansed data"
)
def silver_data():
    return spark.readStream.table("bronze_table").filter("id IS NOT NULL")
```

---

## 6.3 Datasets

### Streaming Tables

**Definition:**
Unity Catalog managed table that serves as a **streaming target** for SDP flows.

**Key characteristics:**

- Can have **one or more** streaming flows (Append, AUTO CDC) writing to it
- Supports both **explicit** and **implicit** flow definitions
- Always backed by Delta format
- Created with `CREATE OR REFRESH STREAMING TABLE` (SQL) or `dp.create_streaming_table()` / `@dp.table()` (Python)

**Use the `STREAM` keyword to read with streaming semantics:**

- Throws error if source has changes/deletions to existing records

**SQL - Explicit creation:**

```sql
CREATE OR REFRESH STREAMING TABLE bronze_events
COMMENT 'Raw event data';

CREATE FLOW bronze_ingest AS
  SELECT *
  FROM STREAM read_files(
    '/path/to/data',
    format => 'json'
  );
```

**SQL - Implicit creation:**

```sql
CREATE OR REFRESH STREAMING TABLE silver_events
COMMENT 'Cleansed events'
AS
  SELECT event_id, event_type, timestamp
  FROM STREAM(bronze_events)
  WHERE event_id IS NOT NULL;
```

**Python - Explicit:**

```python
dp.create_streaming_table(
    name="bronze_events",
    comment="Raw event data"
)

@dp.append_flow(target="bronze_events", name="ingest_flow")
def ingest():
    return spark.readStream.format("json").load("/path/to/data")
```

**Python - Implicit:**

```python
@dp.table(
    name="silver_events",
    comment="Cleansed events"
)
def silver_data():
    return (
        spark.readStream.table("bronze_events")
        .filter("event_id IS NOT NULL")
    )
```

**Critical interview points:**

- Streaming tables are **always streaming** — use streaming semantics for reads
- Changes/deletions to records in source cause **errors**
- Support AUTO CDC flows (Databricks-only feature)

### Materialized Views

**Definition:**
Unity Catalog managed table that serves as a **batch target** with **incremental processing**.

**Key characteristics:**

- Batch semantics (not streaming)
- **Incremental processing engine** — only processes new data/changes when possible
- Flows always defined **implicitly** within view definition
- Write transformation logic with batch semantics; engine handles incremental processing
- Created with `CREATE OR REFRESH MATERIALIZED VIEW` (SQL) or `@dp.materialized_view()` (Python)

**SQL syntax:**

```sql
CREATE OR REFRESH MATERIALIZED VIEW gold_aggregates
COMMENT 'Daily aggregates'
AS
  SELECT
    date,
    event_type,
    COUNT(*) as event_count,
    AVG(duration) as avg_duration
  FROM silver_events
  GROUP BY date, event_type;
```

**Python syntax:**

```python
@dp.materialized_view(
    name="gold_aggregates",
    comment="Daily aggregates"
)
def aggregate_events():
    return (
        spark.read.table("silver_events")
        .groupBy("date", "event_type")
        .agg(
            count("*").alias("event_count"),
            avg("duration").alias("avg_duration")
        )
    )
```

**Incremental processing mechanics:**

- Automatically detects changes in source tables
- Processes only modified partitions/data
- Eliminates need for manual change tracking
- Reduces cost and latency vs. full recomputation

**When materialized views recompute fully:**

- Source schema changes
- View definition changes
- Full refresh update triggered
- Source doesn't support change tracking

**Critical interview points:**

- Write **batch** logic; SDP handles incremental execution
- Cannot be target of explicit flows (always implicit)
- Best for aggregations, joins, complex transformations with batch semantics

### Views

**Definition:**
Temporary datasets within a pipeline — not persisted as tables.

**Key characteristics:**

- Exist only during pipeline execution
- Not stored in metastore
- Used for intermediate transformations
- Save compute/storage for ephemeral data
- Cannot have expectations applied (only streaming tables and materialized views support expectations)

**SQL syntax:**

```sql
CREATE OR REFRESH VIEW cleaned_data
AS
  SELECT
    id,
    TRIM(name) as name,
    CAST(age AS INT) as age
  FROM STREAM(raw_data)
  WHERE id IS NOT NULL;
```

**Python syntax:**

```python
@dp.view(name="cleaned_data")
def clean_data():
    return (
        spark.readStream.table("raw_data")
        .filter("id IS NOT NULL")
        .select("id", trim(col("name")).alias("name"))
    )
```

**Use cases:**

- Intermediate transformations before loading to target
- Data quality filtering
- Schema transformations
- Deduplication logic

**Comparison:**

| Feature       | Streaming Table | Materialized View   | View               |
| ------------- | --------------- | ------------------- | ------------------ |
| Persisted     | Yes             | Yes                 | No                 |
| Processing    | Streaming       | Batch (incremental) | Either             |
| Expectations  | Yes             | Yes                 | No                 |
| Unity Catalog | Yes             | Yes                 | No                 |
| Use case      | Real-time data  | Aggregations, batch | Intermediate steps |

---

## 6.4 Data Movement

### Sinks

**Definition:**
Streaming targets for writing data **outside** Databricks managed tables.

**Supported sink types:**

1. Delta tables (external Unity Catalog tables)
2. Apache Kafka topics
3. Azure EventHub topics
4. Google Pub/Sub topics
5. Custom Python data sources (via `ForEachBatch`)

**Critical constraints:**

- Only **append_flow** can write to sinks
- AUTO CDC flows **not supported** for sinks
- Cannot use sink in dataset definition (only as target)
- Python-only API (no SQL support for sink creation)
- Full refresh **does not clean up** sink data (appends only)

**Python - Delta sink:**

```python
from pyspark import pipelines as dp

# By path
dp.create_sink(
    name="external_delta_sink",
    format="delta",
    options={"path": "/Volumes/catalog/schema/volume/data"}
)

# By table name (fully qualified)
dp.create_sink(
    name="uc_delta_sink",
    format="delta",
    options={"tableName": "catalog.schema.table"}
)

@dp.append_flow(target="external_delta_sink", name="to_delta")
def write_to_delta():
    return spark.readStream.table("silver_events")
```

**Python - Kafka sink:**

```python
dp.create_sink(
    name="kafka_sink",
    format="kafka",
    options={
        "kafka.bootstrap.servers": "broker:9092",
        "topic": "events_topic"
    }
)

@dp.append_flow(target="kafka_sink", name="to_kafka")
def write_to_kafka():
    return (
        spark.readStream.table("silver_events")
        .selectExpr(
            "event_id as key",
            "to_json(struct(*)) as value"
        )
    )
```

**Kafka/EventHub required columns:**

- `value` — **mandatory** (message payload)
- `key` — optional
- `partition` — optional
- `headers` — optional
- `topic` — optional (overrides sink-level topic)

**Python - EventHub sink:**

```python
eh_namespace = "my-eventhub"
bootstrap = f"{eh_namespace}.servicebus.windows.net:9093"

dp.create_sink(
    name="eventhub_sink",
    format="kafka",  # EventHub uses Kafka protocol
    options={
        "kafka.bootstrap.servers": bootstrap,
        "kafka.sasl.mechanism": "PLAIN",
        "kafka.security.protocol": "SASL_SSL",
        "kafka.sasl.jaas.config": f'org.apache.kafka.common.security.plain.PlainLoginModule required username="$ConnectionString" password="{connection_string}";',
        "topic": "events-topic"
    }
)
```

**ForEachBatch custom sink:**

```python
def process_batch(batch_df, batch_id):
    # Custom logic: merge, write to multiple targets, etc.
    batch_df.write.format("jdbc").options(...).save()

dp.create_foreach_batch_sink(
    name="custom_sink",
    foreach_batch_function=process_batch
)

@dp.append_flow(target="custom_sink", name="custom_flow")
def write_custom():
    return spark.readStream.table("silver_events")
```

**ForEachBatch characteristics:**

- Processes stream as micro-batches
- Full Spark DataFrame API available per batch
- Supports merge, upsert, multi-target writes
- Checkpoints managed per-flow automatically
- Not compatible with Databricks Connect

### Append Flows

**Definition:**
Streaming flows that continuously append new data to targets.

**Key capabilities:**

- Target: streaming tables or sinks
- Semantics: Spark Structured Streaming append output mode
- Can be explicit (separate decorator) or implicit (within table)
- Supports one-time execution with `once=True`

**Explicit append flow:**

```python
dp.create_streaming_table("target_table")

@dp.append_flow(
    target="target_table",
    name="append_data",
    once=False,  # Continuous streaming (default)
    spark_conf={"spark.sql.shuffle.partitions": "200"},
    comment="Appends events continuously"
)
def append_events():
    return spark.readStream.table("source_table")
```

**One-time backfill:**

```python
@dp.append_flow(
    target="target_table",
    name="backfill",
    once=True  # Executes once then stops
)
def backfill_historical():
    return spark.read.format("delta").load("/historical/data")
```

**Multiple append flows to same table:**

```python
# Streaming flow
@dp.append_flow(target="combined_table", name="streaming_data")
def stream_data():
    return spark.readStream.table("live_source")

# One-time backfill
@dp.append_flow(target="combined_table", name="backfill", once=True)
def backfill_data():
    return spark.read.table("historical_source")
```

**Critical interview points:**

- Multiple append flows can write to same streaming table
- Useful for backfilling + continuous streaming scenarios
- Use `once=True` for one-time historical loads
- Append mode only — no updates or deletes

### Jobs

SDP pipelines are **orchestrated automatically** — no manual Lakeflow Jobs configuration for internal dependencies.

**When to use Lakeflow Jobs with SDP:**

- Scheduling pipeline updates (daily, hourly, triggered)
- Coordinating pipelines with other workflows (notebooks, SQL queries, ML models)
- Conditional logic (if/else, for-each loops)
- Notifications and alerting
- Multi-pipeline orchestration

**Key distinction:**

- **Within pipeline:** Flows orchestrated automatically by SDP
- **Across pipelines:** Use Lakeflow Jobs for coordination

**Example Lakeflow Job with pipeline:**

```json
{
  "name": "ETL Pipeline Job",
  "tasks": [
    {
      "task_key": "run_sdp_pipeline",
      "pipeline_task": {
        "pipeline_id": "abc123",
        "full_refresh": false
      }
    },
    {
      "task_key": "send_notification",
      "depends_on": [{"task_key": "run_sdp_pipeline"}],
      "notebook_task": {...}
    }
  ]
}
```

---

## 6.5 Data Quality

### Expectations

**Definition:**
Optional clauses in dataset definitions that apply **data quality checks** on each record.

**Key characteristics:**

- Use standard SQL Boolean expressions
- Can combine multiple expectations per dataset
- Collect metrics on pass/fail rates
- **Only supported on streaming tables and materialized views** (not views)
- Three actions: retain (default), drop, fail

**Expectation components:**

1. **Name** — identifies the constraint
2. **Constraint** — SQL Boolean expression
3. **Action** — what to do when constraint violated

### Actions

| Action               | SQL Syntax                               | Python Syntax                        | Behavior                          |
| -------------------- | ---------------------------------------- | ------------------------------------ | --------------------------------- |
| **Retain** (default) | `EXPECT (expr)`                          | `@dp.expect("name", "expr")`         | Keep invalid records, log metrics |
| **Drop**             | `EXPECT (expr) ON VIOLATION DROP ROW`    | `@dp.expect_or_drop("name", "expr")` | Drop invalid records              |
| **Fail**             | `EXPECT (expr) ON VIOLATION FAIL UPDATE` | `@dp.expect_or_fail("name", "expr")` | Fail entire update                |

**SQL examples:**

```sql
-- Simple constraint
CONSTRAINT non_negative_price
EXPECT (price >= 0)

-- With drop action
CONSTRAINT valid_email
EXPECT (email LIKE '%@%.%')
ON VIOLATION DROP ROW

-- Multiple constraints
CONSTRAINT valid_dates
EXPECT (start_date <= end_date AND end_date <= current_date()),
CONSTRAINT non_null_id
EXPECT (id IS NOT NULL)
ON VIOLATION FAIL UPDATE

-- Complex business logic
CONSTRAINT valid_order_state
EXPECT (
  (status = 'ACTIVE' AND balance > 0)
  OR (status = 'PENDING' AND created_date > current_date() - INTERVAL 7 DAYS)
)
```

**Python examples:**

```python
@dp.table(
    name="validated_table",
    expect={"valid_id": "id IS NOT NULL"},
    expect_or_drop={"valid_email": "email LIKE '%@%.%'"}
)
def validated_data():
    return spark.readStream.table("source")
```

**Python - Grouped expectations (Python-only):**

```python
# Retain all invalid records
@dp.table(
    name="table1",
    expect_all={
        "non_negative_price": "price >= 0",
        "valid_date": "order_date <= current_date()"
    }
)
def data1():
    return spark.readStream.table("source")

# Drop all invalid records
@dp.table(
    name="table2",
    expect_all_or_drop={
        "valid_id": "id IS NOT NULL",
        "valid_email": "email LIKE '%@%.%'"
    }
)
def data2():
    return spark.readStream.table("source")

# Fail on any invalid record
@dp.table(
    name="table3",
    expect_all_or_fail={
        "critical_field": "amount > 0",
        "required_status": "status IN ('ACTIVE', 'PENDING')"
    }
)
def data3():
    return spark.readStream.table("source")
```

**Advanced pattern - Centralized expectation repository:**

```python
# Define rules in a table or dict
expectation_rules = {
    "validity": {
        "non_null_id": "id IS NOT NULL",
        "valid_email": "email LIKE '%@%.%'"
    },
    "accuracy": {
        "positive_amount": "amount > 0",
        "recent_date": "date >= '2020-01-01'"
    }
}

@dp.table(
    name="validated_data",
    expect_all_or_drop=expectation_rules["validity"]
)
def apply_expectations():
    return spark.readStream.table("source")
```

**Viewing data quality metrics:**

- UI: Pipeline → Dataset → "Data quality" tab
- API: Query SDP event log (Delta table)

**When expectations don't generate metrics:**

- No expectations defined
- Flow doesn't support expectations (e.g., sinks)
- Flow type doesn't support expectations
- No updates to table/materialized view in flow run
- Metrics reporting disabled in pipeline config

**Critical interview points:**

- Expectations are **soft constraints** (unlike CHECK constraints in databases)
- Default behavior: retain invalid records (enables processing messy data)
- Only streaming tables and materialized views support expectations
- Python provides `expect_all*` decorators for grouping (SQL doesn't)

---

## 6.6 Change Data Capture

### Auto CDC

**Definition:**
Databricks-only streaming flow that processes CDC events with automatic out-of-order handling.

**Key capabilities:**

- Handles out-of-order events automatically
- Supports SCD Type 1 and Type 2
- Eliminates need for complex merge logic
- Requires sequencing column (monotonically increasing)
- **Only available in Lakeflow SDP** (not Apache Spark)
- **Only writes to streaming tables**

**Requirements:**

- Sequencing column — represents proper ordering of source data
- One distinct update per key at each sequencing value
- NULL sequencing values unsupported
- Sortable data type for sequencing column

**Sequencing column characteristics:**

- Must be monotonically increasing
- Can be timestamp, sequence number, version, etc.
- For multiple columns, use STRUCT: orders by first field, then second on ties

**SQL syntax:**

```sql
CREATE OR REFRESH STREAMING TABLE target;

CREATE FLOW cdc_flow AS
AUTO CDC INTO target
FROM stream(source_table)
KEYS (user_id)
SEQUENCE BY sequence_num
COLUMNS * EXCEPT (operation, sequence_num)
APPLY AS DELETE WHEN operation = "DELETE"
STORED AS SCD TYPE 1;  -- or TYPE 2
```

**Python syntax:**

```python
dp.create_streaming_table(name="target")

dp.create_auto_cdc_flow(
    name="cdc_flow",
    target="target",
    source="source_table",
    keys=["user_id"],
    sequence_by=col("sequence_num"),
    except_column_list=["operation", "sequence_num"],
    apply_as_deletes=expr("operation = 'DELETE'"),
    stored_as_scd_type="1"  # or "2"
)
```

**APPLY AS DELETE clause:**

- Specifies condition for treating event as DELETE
- If omitted, no deletes processed (only inserts/updates)

**COLUMNS clause:**

- `COLUMNS *` — include all columns
- `COLUMNS * EXCEPT (col1, col2)` — exclude specific columns
- Excluded columns typically: operation, sequence number

### SCD Type 1

**Definition:**
Overwrite existing records — no history retention.

**Use case:**
Correcting errors, updating non-critical fields (e.g., email address).

**SQL example:**

```sql
CREATE OR REFRESH STREAMING TABLE customers;

CREATE FLOW customers_cdc AS
AUTO CDC INTO customers
FROM stream(cdc_feed.customers)
KEYS (customer_id)
SEQUENCE BY event_timestamp
COLUMNS * EXCEPT (operation, event_timestamp)
APPLY AS DELETE WHEN operation = "DELETE"
STORED AS SCD TYPE 1;
```

**Python example:**

```python
dp.create_streaming_table("customers")

dp.create_auto_cdc_flow(
    name="customers_cdc",
    target="customers",
    source="cdc_feed.customers",
    keys=["customer_id"],
    sequence_by=col("event_timestamp"),
    except_column_list=["operation", "event_timestamp"],
    apply_as_deletes=expr("operation = 'DELETE'"),
    stored_as_scd_type="1"
)
```

**Behavior:**

- New record → INSERT
- Existing record → UPDATE (overwrites all columns)
- Delete event → DELETE

**Example data flow:**

| Event | customer_id | name  | city | operation | sequence_num |
| ----- | ----------- | ----- | ---- | --------- | ------------ |
| 1     | 123         | Alice | NYC  | INSERT    | 1            |
| 2     | 123         | Alice | LA   | UPDATE    | 2            |
| 3     | 123         | NULL  | NULL | DELETE    | 3            |

**Resulting table (SCD Type 1):**

- After event 1: (123, Alice, NYC)
- After event 2: (123, Alice, LA) — **overwrites**
- After event 3: **empty** — deleted

### SCD Type 2

**Definition:**
Retain full history of changes by creating new records for each version.

**Use case:**
Audit trails, historical analysis, compliance requirements.

**SQL syntax:**

```sql
CREATE OR REFRESH STREAMING TABLE customers_history;

CREATE FLOW customers_scd2 AS
AUTO CDC INTO customers_history
FROM stream(cdc_feed.customers)
KEYS (customer_id)
SEQUENCE BY event_timestamp
COLUMNS * EXCEPT (operation, event_timestamp)
APPLY AS DELETE WHEN operation = "DELETE"
STORED AS SCD TYPE 2
TRACK HISTORY ON *;  -- or TRACK HISTORY ON * EXCEPT (city)
```

**Python syntax:**

```python
dp.create_streaming_table("customers_history")

dp.create_auto_cdc_flow(
    name="customers_scd2",
    target="customers_history",
    source="cdc_feed.customers",
    keys=["customer_id"],
    sequence_by=col("event_timestamp"),
    except_column_list=["operation", "event_timestamp"],
    apply_as_deletes=expr("operation = 'DELETE'"),
    stored_as_scd_type="2",
    track_history_column_list=None  # None = track all columns
)
```

**SCD Type 2 metadata columns:**

- `__START_AT` — sequence value when record became active
- `__END_AT` — sequence value when record became inactive (NULL = current)

**TRACK HISTORY clause:**

- `TRACK HISTORY ON *` — track changes on all columns
- `TRACK HISTORY ON * EXCEPT (col1, col2)` — track all except specified
- Only creates new version if tracked columns change

**Example data flow:**

| Event | customer_id | name        | city | operation | event_timestamp  |
| ----- | ----------- | ----------- | ---- | --------- | ---------------- |
| 1     | 123         | Alice       | NYC  | INSERT    | 2024-01-01 10:00 |
| 2     | 123         | Alice       | LA   | UPDATE    | 2024-01-02 11:00 |
| 3     | 123         | Alice Smith | LA   | UPDATE    | 2024-01-03 12:00 |

**Resulting table (SCD Type 2, TRACK HISTORY ON \*):**

| customer_id | name        | city | \_\_START_AT     | \_\_END_AT       |
| ----------- | ----------- | ---- | ---------------- | ---------------- |
| 123         | Alice       | NYC  | 2024-01-01 10:00 | 2024-01-02 11:00 |
| 123         | Alice       | LA   | 2024-01-02 11:00 | 2024-01-03 12:00 |
| 123         | Alice Smith | LA   | 2024-01-03 12:00 | NULL             |

**Resulting table (SCD Type 2, TRACK HISTORY ON \* EXCEPT (city)):**

| customer_id | name        | city | \_\_START_AT     | \_\_END_AT       |
| ----------- | ----------- | ---- | ---------------- | ---------------- |
| 123         | Alice       | LA   | 2024-01-01 10:00 | 2024-01-03 12:00 |
| 123         | Alice Smith | LA   | 2024-01-03 12:00 | NULL             |

Note: City change (NYC → LA) didn't create new version because city is excluded from tracking.

**Querying current records only:**

```sql
SELECT * FROM customers_history WHERE __END_AT IS NULL;
```

**Querying as of specific time:**

```sql
SELECT *
FROM customers_history
WHERE __START_AT <= '2024-01-02 11:30'
  AND (__END_AT > '2024-01-02 11:30' OR __END_AT IS NULL);
```

**Primary key for SCD Type 2:**

- Keys + `__START_AT` (composite primary key)
- Ensures uniqueness per version

### SCD Type 3

**Not officially supported in AUTO CDC.**

SCD Type 3 (add columns for previous values) must be implemented manually using:

- Custom merge logic in ForEachBatch sink
- Manual DataFrame operations

**Manual Type 3 pattern:**

```python
# Not supported natively - requires custom code
from delta.tables import DeltaTable

def scd_type3_merge(batch_df, batch_id):
    target = DeltaTable.forName(spark, "target_table")

    target.alias("target").merge(
        batch_df.alias("source"),
        "target.id = source.id"
    ).whenMatchedUpdate(set={
        "current_value": "source.value",
        "previous_value": "target.current_value",
        "effective_date": "source.timestamp"
    }).whenNotMatchedInsert(values={
        "id": "source.id",
        "current_value": "source.value",
        "previous_value": "NULL",
        "effective_date": "source.timestamp"
    }).execute()

# Use in ForEachBatch sink
dp.create_foreach_batch_sink(
    name="scd3_sink",
    foreach_batch_function=scd_type3_merge
)
```

---

## Auto CDC FROM SNAPSHOT

**Definition:**
Processes CDC from **snapshots** rather than change streams.

**Use case:**

- Source system provides periodic full snapshots
- No incremental change feed available
- Historical backfill from snapshots

**Requirements:**

- Snapshots must be in **ascending order by version**
- Out-of-order snapshots throw error

**Python syntax:**

```python
dp.create_streaming_table("target_from_snapshot")

dp.create_auto_cdc_from_snapshot_flow(
    name="snapshot_cdc",
    target="target_from_snapshot",
    snapshot=spark.read.table("snapshots"),  # DataFrame
    keys=["user_id"],
    sequence_by=col("snapshot_version"),
    stored_as_scd_type="2"
)
```

**Critical difference vs. AUTO CDC:**

- AUTO CDC: processes change events (INSERT/UPDATE/DELETE)
- AUTO CDC FROM SNAPSHOT: compares snapshots to infer changes

---

## Critical Interview Insights

### SDP vs. Manual Spark

| Aspect                 | SDP                               | Manual Spark/Streaming |
| ---------------------- | --------------------------------- | ---------------------- |
| Orchestration          | Automatic                         | Manual (Lakeflow Jobs) |
| Dependency management  | Automatic                         | Manual                 |
| Retry logic            | Hierarchical (task→flow→pipeline) | Manual                 |
| Incremental processing | Built-in (materialized views)     | Manual code            |
| CDC handling           | AUTO CDC API                      | Complex merge logic    |
| Watermarks             | Automatic                         | Manual configuration   |
| Code volume            | 10-100 lines                      | 100-1000+ lines        |

### When NOT to Use SDP

- Need non-Delta outputs (use sinks or ForEachBatch)
- Complex custom stateful operations (use manual Structured Streaming)
- Interactive/ad-hoc queries (use notebooks)
- Batch-only with no incremental processing needs
- Need fine-grained control over checkpoints/watermarks

### Common Pitfalls

1. **Using `dlt` module instead of `pyspark.pipelines`**

   - Old: `import dlt`
   - New: `from pyspark import pipelines as dp`

2. **Applying expectations to views**

   - Only streaming tables and materialized views support expectations

3. **Using AUTO CDC with sinks**

   - AUTO CDC only writes to streaming tables
   - Use append_flow for sinks

4. **Running pipeline code interactively**

   - `pyspark.pipelines` only available in pipeline context
   - Test logic separately, not pipeline decorators

5. **Forgetting STREAM keyword in SQL**

   - Streaming reads require `STREAM(table)` or `STREAM read_files(...)`

6. **Multiple flows to materialized views**

   - Materialized views only support implicit flows (within definition)
   - Use streaming tables for multiple explicit flows

7. **Not handling NULL sequencing values**
   - AUTO CDC requires non-NULL sequencing column
   - Filter or handle NULLs before CDC processing

---

## Performance Considerations

**Materialized view optimization:**

- Incremental processing most effective with:
  - Partitioned source tables
  - Append-only or low-update-rate sources
  - Delta change data feed enabled
- Full recomputation triggers:
  - Schema changes
  - Definition changes
  - Non-trackable sources

**Streaming table optimization:**

- Use Auto Loader for cloud storage ingestion
- Enable schema evolution when needed
- Monitor streaming metrics (latency, throughput)

**Expectation performance:**

- Expectations evaluated per-record (can impact throughput)
- Use `expect_or_drop` to reduce downstream processing
- Complex constraints increase evaluation time

**CDC performance:**

- Sequencing column should be indexed in source
- Minimize tracked columns in SCD Type 2
- Use appropriate key cardinality (not too fine-grained)

---

## December 2025 Updates Summary

- **Lakeflow SDP** is the current name (formerly DLT)
- `pyspark.pipelines` module (alias `dp`) replaces `dlt` module
- Apache Spark 4.1+ includes open-source declarative pipelines
- Databricks extends with AUTO CDC, sinks, ForEachBatch
- Full backward compatibility — no migration required
- Enhanced Lakeflow integration for end-to-end workflows
- Premium plan required for Lakeflow SDP

---

**End of Interview Preparation Notes**
