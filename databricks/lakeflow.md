# 3. Data Engineering & ETL (Lakeflow Era)

> **Context shift (important for interviews)**
> Databricks is moving from _imperative Spark pipelines_ → **declarative, optimized-by-default pipelines**.
> **Lakeflow** is the umbrella for this evolution.

---

## 3.1 Lakeflow (2025+)

### What is Lakeflow?

**Lakeflow** is Databricks’ **unified ingestion + transformation + orchestration framework** for data engineering.

It:

- Standardizes ingestion, ETL, and orchestration
- Uses **declarative definitions**
- Automatically applies incremental processing and optimizations

Lakeflow **does not replace Spark** — it **abstracts execution decisions**.

---

### What Lakeflow Replaces / Augments

| Older Tool                     | Status           |
| ------------------------------ | ---------------- |
| Custom Spark jobs              | Augmented        |
| DLT (classic)                  | Evolved into SDP |
| Complex orchestration logic    | Reduced          |
| Hand-written incremental logic | Eliminated       |

Interview phrasing:

> Lakeflow shifts responsibility from developers to the platform.

---

### Key Characteristics

- Declarative pipelines
- SQL-first, Python-supported
- Built-in quality, lineage, retries
- Tight integration with Jobs & Unity Catalog

---

## 3.2 File Ingestion – Auto Loader (`cloudFiles`)

### What is Auto Loader?

Auto Loader is a **structured streaming–based ingestion mechanism** for cloud object storage.

Supported formats:

- JSON
- CSV
- Parquet
- Avro
- ORC
- Text

Supported storage:

- AWS S3
- Azure ADLS Gen2
- GCP GCS

---

### Notification Mode vs Directory Listing Mode

#### Notification Mode (Preferred)

- Uses cloud-native notifications (SQS, Event Grid, Pub/Sub)
- Near real-time
- Scales to millions of files

Requirements:

- Cloud permissions
- Event infrastructure

---

#### Directory Listing Mode

- Periodic file listing
- Slower
- Higher storage API cost

Use only when:

- Notifications cannot be configured

---

### Example: Auto Loader Ingestion (PySpark)

```python
df = (
  spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/schema/events")
    .load("/mnt/raw/events")
)

df.writeStream \
  .format("delta") \
  .option("checkpointLocation", "/mnt/chk/events") \
  .table("raw.events")
```

Key interview points:

- Uses **Structured Streaming**
- Schema is tracked externally
- Exactly-once semantics via checkpointing

---

## 3.3 Declarative ETL

### Declarative vs Imperative

Declarative = define **what the table should look like**, not how to compute it.

Databricks handles:

- Dependency graph
- Incremental recomputation
- Retry logic
- Backfills

---

### Supported Languages

- SQL (primary)
- Python (secondary)

SQL-first is intentional:

- Easier lineage
- Easier optimization
- Easier governance

---

## 3.4 Data Quality (Expectations)

### What are Expectations?

Expectations are **data quality rules** enforced at write time.

They:

- Validate incoming data
- Track violations
- Control failure behavior

---

### Example Expectation

```sql
CONSTRAINT valid_id
EXPECT (id IS NOT NULL)
ON VIOLATION DROP ROW
```

Violation actions:

- `DROP ROW`
- `FAIL UPDATE`
- `WARN`

Interview insight:

> Expectations are enforced **before data is committed**.

---

### Observability

- Metrics per expectation
- Failures visible in pipeline UI
- Integrated with lineage

---

## 3.5 Enzyme Engine (Internal)

### What is Enzyme?

**Enzyme** is Databricks’ internal optimization engine for **incremental processing**.

It:

- Tracks data dependencies
- Minimizes recomputation
- Applies state-aware optimizations

---

### Why Enzyme Matters

Without Enzyme:

- Small upstream changes → full recompute

With Enzyme:

- Only affected downstream data is recomputed

Interview phrasing:

> Enzyme makes declarative pipelines cost-efficient at scale.

Status:

- **Internal**
- **Not directly configurable**
- Interviewers expect conceptual understanding only

---

## 3.6 Data Layout Optimization – Liquid Clustering (🔥 Very High Yield)

### What is Liquid Clustering?

Liquid Clustering is a **dynamic, adaptive data layout optimization** for Delta tables.

It:

- Eliminates static partitioning
- Replaces Z-Ordering
- Continuously optimizes layout

---

### Why Partitioning Is Problematic

- Requires choosing columns upfront
- Causes data skew
- Hard to evolve

Liquid Clustering:

- No fixed partition columns
- Adapts based on query patterns
- Optimizes automatically

---

### Key Interview Statement

> Liquid Clustering removes the need to design partitions manually.

---

### Usage Example

```sql
CREATE TABLE sales
USING DELTA
CLUSTER BY (customer_id);
```

Notes:

- Column list is **advisory**
- Databricks may change clustering internally
- Works best with large tables

---

### Trade-offs

- Not ideal for very small tables
- Background optimization cost

---

## 3.7 Ingestion Patterns

### COPY INTO vs Auto Loader

| Aspect        | COPY INTO | Auto Loader |
| ------------- | --------- | ----------- |
| Mode          | Batch     | Streaming   |
| Incremental   | Yes       | Yes         |
| File tracking | Metadata  | Checkpoints |
| Latency       | Higher    | Low         |
| Complexity    | Low       | Medium      |

---

### When to Use COPY INTO

- One-time backfills
- Scheduled batch ingestion
- Simple pipelines

---

### When to Use Auto Loader

- Continuous ingestion
- Event-driven pipelines
- High file arrival rate

---

### COPY INTO Example

```sql
COPY INTO bronze.events
FROM '/mnt/raw/events'
FILEFORMAT = JSON;
```

Internals:

- Tracks loaded files
- Idempotent
- No streaming state

---

### Job Integration

- COPY INTO → scheduled Jobs
- Auto Loader → continuous or triggered Jobs

---

## Common Interview Pitfalls (Section 3)

- Treating Lakeflow as a product, not a framework
- Confusing Liquid Clustering with partitioning
- Overusing directory listing mode
- Writing manual incremental logic when declarative is available

---

### Next Section

Reply **“Proceed to Section 4 (Databricks SQL)”** when ready.
