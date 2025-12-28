# 7. Pipelines: Delta Live Tables (DLT)

**Concept**

Pipelines are **declarative data transformation frameworks**. Unlike jobs (imperative orchestration), pipelines define the desired state of data.

**Key Differences from Jobs**

| Aspect               | Jobs                      | Pipelines (DLT)                            |
| -------------------- | ------------------------- | ------------------------------------------ |
| **Paradigm**         | Imperative (how to do it) | Declarative (what state to reach)          |
| **Parameterization** | Task parameters + widgets | Pipeline configuration + Spark conf        |
| **Execution Mode**   | Triggered or continuous   | Triggered or continuous                    |
| **Error Handling**   | Task-level retries        | Quality checks; data validation            |
| **Use Case**         | General orchestration     | Data transformation, incremental ingestion |

**Pipeline Configuration**

```yaml
resources:
  pipelines:
    etl_pipeline:
      name: etl_pipeline
      storage: /Pipelines/etl_storage
      libraries:
        - notebook:
            path: ./notebooks/silver_layer.py
        - notebook:
            path: ./notebooks/gold_layer.py
      configuration:
        source_path: "/mnt/data/raw/"
        target_catalog: "main"
        target_schema: "gold"
      clusters:
        - label: default
          num_workers: 4
          node_type_id: i3.xlarge
      development: false
      continuous: false # Triggered mode; set true for continuous
      channel: CURRENT # Production-ready Databricks Runtime
```

**Environment-Specific Pipeline Parameterization**

```yaml
variables:
  pipeline_storage:
    default: /Pipelines/etl_dev

targets:
  dev:
    variables:
      pipeline_storage: /Pipelines/etl_dev

  prod:
    variables:
      pipeline_storage: /Pipelines/etl_prod
```

Pipeline code accesses configuration via Spark:

```python
config = spark.conf.get("pipeline.config.source_path", "default_value")
```

**Avoiding Widget Confusion in Pipelines**

⚠️ **DLT pipelines do NOT use notebook widgets** like jobs do. Use:

- Pipeline `configuration` section (key-value pairs)
- Bundle variables
- Spark session configuration
- Secrets (via `dbutils.secrets.get()`)

---
