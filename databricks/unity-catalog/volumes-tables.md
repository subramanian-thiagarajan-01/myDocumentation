# 03 Tables, Volumes, Views and M.Views

## 5. Tables in Unity Catalog

### 5.1 Managed Tables

**Definition:**
UC manages both metadata and underlying data files. When table is dropped, both metadata and data are deleted.

**Storage:**
Data stored in managed location (metastore/catalog/schema level).

**Example:**

```python
# PySpark
df = spark.createDataFrame([
  (1, "Alice", 100),
  (2, "Bob", 200)
], ["id", "name", "amount"])

df.write.saveAsTable("dev_catalog.sales.transactions")
```

**Underlying Storage:**

```
abfss://metastore@storage.dfs.core.windows.net/
  └── catalogs/
      └── dev_catalog/
          └── schemas/
              └── sales/
                  └── tables/
                      └── transactions/
                          ├── part-00000.snappy.parquet
                          └── _delta_log/
```

**Interview Question:** _What happens when you DROP a managed table?_

- **Metadata deleted:** Table definition removed from metastore
- **Data deleted:** Underlying files deleted from storage
- **Cannot recover** unless you have storage-level backups

### 5.2 External Tables

**Definition:**
UC manages metadata only. User manages underlying data files. When table is dropped, only metadata is removed.

**Example:**

```sql
-- Create external table
CREATE EXTERNAL TABLE dev_catalog.raw.customer_data
LOCATION 's3://my-bucket/customers/'
AS SELECT * FROM VALUES (1, 'John'), (2, 'Jane');

-- Drop external table
DROP TABLE dev_catalog.raw.customer_data;
-- Metadata removed, but s3://my-bucket/customers/ remains intact
```

**Supported Formats:**

- **Delta Lake** (recommended)
- **Parquet**
- **Apache Iceberg** (Databricks Runtime 13.0+)
- CSV, JSON, ORC (read-only in UC)

**Interview Question:** _Why use External Tables?_

- Data shared across platforms (Databricks, Snowflake, Presto)
- Data lifecycle managed independently
- Compliance requirements for data retention

### 5.3 Managed vs External Comparison

| Aspect               | Managed Table      | External Table          |
| -------------------- | ------------------ | ----------------------- |
| **Metadata**         | UC controls        | UC controls             |
| **Data**             | UC controls        | User controls           |
| **DROP behavior**    | Deletes data       | Keeps data              |
| **Storage location** | Managed location   | User-specified path     |
| **Use case**         | Standard workflows | Shared data, compliance |

**Code Example - Checking Table Type:**

```sql
DESCRIBE EXTENDED dev_catalog.sales.transactions;

-- Look for:
-- Type: MANAGED or EXTERNAL
-- Location: abfss://... or s3://...
```

---

## 6. Volumes (Non-Tabular Data)

### 6.1 Purpose

**What Are Volumes?**
Storage for non-tabular data (PDFs, images, ML models, binaries) with UC governance.

**Why Volumes?**

- Replaces legacy DBFS mounts
- Governed access via UC permissions
- Lineage tracking for non-tabular files

### 6.2 Managed Volumes

**Definition:**
UC manages lifecycle and storage. Deleting volume deletes files.

**Example:**

```sql
-- Create managed volume
CREATE VOLUME dev_catalog.ml.model_artifacts;

-- Python: Write file to volume
with open("/Volumes/dev_catalog/ml/model_artifacts/model_v1.pkl", "wb") as f:
    pickle.dump(model, f)

-- Python: Read file from volume
with open("/Volumes/dev_catalog/ml/model_artifacts/model_v1.pkl", "rb") as f:
    model = pickle.load(f)
```

**Path Structure:**

```
/Volumes/<catalog>/<schema>/<volume>/<file_path>
```

### 6.3 External Volumes

**Definition:**
Points to external storage path. Deleting volume does not delete files.

**Example:**

```sql
-- Create external volume
CREATE EXTERNAL VOLUME dev_catalog.raw.external_files
LOCATION 's3://my-bucket/unstructured/';

-- List files
%python
dbutils.fs.ls("/Volumes/dev_catalog/raw/external_files/")
```

### 6.4 Permissions on Volumes

```sql
-- Grant read/write access
GRANT READ VOLUME, WRITE VOLUME ON VOLUME dev_catalog.ml.model_artifacts TO data_scientists;

-- Revoke access
REVOKE WRITE VOLUME ON VOLUME dev_catalog.ml.model_artifacts FROM data_scientists;
```

**Interview Question:** _Volumes vs DBFS?_

- **Volumes:** Governed, UC-integrated, recommended for UC-enabled workspaces
- **DBFS:** Legacy, no UC governance, being deprecated

## 7. Views & Materialized Views

### 7.1 Standard Views in UC

**Definition:**
Virtual table defined by a query. No data stored; query executed each time view is accessed.

**Example:**

```sql
CREATE VIEW prod_catalog.analytics.monthly_revenue AS
SELECT
  DATE_TRUNC('MONTH', order_date) AS month,
  SUM(amount) AS revenue
FROM prod_catalog.sales.orders
GROUP BY DATE_TRUNC('MONTH', order_date);
```

**Governance:**

- Views respect UC permissions
- User needs SELECT on underlying tables
- Views can enforce row/column-level security

### 7.2 Materialized Views

**Definition:**
Pre-computed view where results are stored and incrementally refreshed.

**Key Features (Databricks Runtime 12.2+):**

- **Incremental refresh:** Only processes new data since last refresh
- **UC-governed:** Permissions and lineage tracked
- **Delta Lake backed:** Results stored as Delta table

**Example:**

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW prod_catalog.analytics.daily_summary
AS
SELECT
  DATE(order_timestamp) AS order_date,
  product_id,
  SUM(quantity) AS total_quantity,
  SUM(amount) AS total_revenue
FROM prod_catalog.sales.orders
GROUP BY DATE(order_timestamp), product_id;

-- Refresh materialized view (manual)
REFRESH MATERIALIZED VIEW prod_catalog.analytics.daily_summary;

-- Check refresh status
DESCRIBE EXTENDED prod_catalog.analytics.daily_summary;
-- Look for: Last Refresh Time, Refresh Status
```

**Automatic Refresh (Databricks SQL Warehouses):**

```sql
CREATE MATERIALIZED VIEW prod_catalog.analytics.hourly_summary
SCHEDULE CRON '0 * * * *'  -- Every hour
AS SELECT ...;
```

**Interview Question:** _Materialized View vs Table?_

- **Materialized View:** Auto-refreshed, lineage-tracked, tied to source query
- **Table:** Manual load, no automatic refresh, no query dependency

**When to Use Materialized Views:**

- Expensive aggregations queried frequently
- Incremental refresh from streaming sources
- Dashboard backend requiring low-latency access

**Limitation:**

- Cannot be directly written to (read-only from user perspective)
- Refresh may lag behind source data
