# 10. Orchestration & Dependencies

### 10.1 Automatic Dependency Resolution

DLT builds DAG from `dlt.read()` calls:

```python
@dlt.table
def bronze():
    return spark.read.parquet("...")

@dlt.table
def silver():
    return dlt.read("bronze").filter(...)  # DLT knows: silver depends on bronze

@dlt.table
def gold():
    return dlt.read("silver").groupBy(...).agg(...)  # DLT knows: gold depends on silver
```

Execution order: bronze → silver → gold (automatically scheduled).

### 10.2 Multi-Pipeline Dependencies

DLT pipelines are **isolated**. To depend on tables from other pipelines:

```python
@dlt.table
def my_table():
    # Read from another pipeline's output location
    return spark.read.table("other_catalog.other_schema.other_table")
```

**Not natively supported**: DLT does not expose cross-pipeline dependency tracking. Use Databricks Workflows for multi-pipeline orchestration.

### 10.3 Circular Dependency Detection

DLT validates DAG at pipeline startup. Circular refs cause immediate failure:

```python
# Error: Cannot create circular dependencies
@dlt.table
def table_a():
    return dlt.read("table_b")

@dlt.table
def table_b():
    return dlt.read("table_a")
```
