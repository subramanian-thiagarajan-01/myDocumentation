# 1. Introduction

## 1. Core Concepts

### 1.1 Declarative vs Imperative Paradigm

**Imperative (traditional Spark jobs):**

```python
# Traditional approach: explicitly manage orchestration
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("ETL").getOrCreate()

# Read, transform, write - manual sequencing
raw_df = spark.read.parquet("s3://bucket/raw")
cleaned_df = raw_df.filter("id IS NOT NULL")
cleaned_df.write.mode("overwrite").parquet("s3://bucket/cleaned")

result_df = spark.read.parquet("s3://bucket/cleaned").groupBy("category").count()
result_df.write.mode("overwrite").parquet("s3://bucket/results")
```

**Declarative (DLT):**

```python
import dlt
from pyspark.sql import functions as F

@dlt.table
def raw_data():
    return spark.read.parquet("s3://bucket/raw")

@dlt.table
def cleaned_data():
    return dlt.read("raw_data").filter("id IS NOT NULL")

@dlt.table
def results():
    return dlt.read("cleaned_data").groupBy("category").count()
```

Key difference: DLT infers dependency graph and handles orchestration. You declare _what_, not _how_.

### 1.2 Streaming-First Design

DLT is built on Spark Structured Streaming fundamentals, enabling:

- Exactly-once semantics via checkpointing
- Incremental processing of append-only data
- Unified batch/stream execution model

**Critical insight:** Streaming tables process each record once (assuming append-only source). Materialized views compute full state on each trigger.

### 1.3 Pipeline Lifecycle

1. **Create Update**: DLT analyzes code, discovers tables/views, builds DAG
2. **Initialize**: First run processes all historical data (full refresh)
3. **Incremental Updates**: Subsequent runs process only new/changed data
4. **State Persistence**: Checkpoints saved at `storage_location/system/checkpoints/{table_name}`

**Not officially guaranteed**: Exactly how DLT decides when to do full vs incremental refresh for materialized views depends on cost-based optimizer (varies by version).

### 1.4 Medallion Architecture in DLT

Bronze → Silver → Gold pattern fits naturally:

- **Bronze**: Raw ingestion via streaming tables with minimal validation
- **Silver**: Cleaned, deduplicated tables with Expectations; mix of streaming tables and MVs
- **Gold**: Aggregated/curated tables; typically MVs for performance
