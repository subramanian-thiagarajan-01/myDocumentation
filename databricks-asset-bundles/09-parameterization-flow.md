# 9. Parameterization Model: The Complete Flow

**Pareto Pattern: How Parameters Flow Through the System**

```
┌─────────────────────────────────────────────────────────────┐
│ Parameter Sources                                           │
│ - GitHub Actions secrets → environment variables            │
│ - CLI: --var="my_var=value"                                 │
│ - Variable defaults in databricks.yml                       │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
        ┌─────────────────────┐
        │ Bundle Variables    │
        │ ${var.name}         │
        │ Resolved at         │
        │ deploy-time         │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────────────────────────────────┐
        │ Job Definition (databricks.yml)                 │
        │ - tasks.base_parameters                         │
        │ - new_cluster configuration                     │
        │ - schedule, timeout_seconds, etc.               │
        └──────────┬──────────────────────────────────────┘
                   │
                   │ Job deployed to workspace
                   │ (one-time deployment)
                   │
        ┌──────────▼──────────────────────────────────────┐
        │ Job Execution (Run-time)                        │
        │ - base_parameters → notebook widgets            │
        │ - Retrieved via dbutils.widgets.get()           │
        └──────────┬──────────────────────────────────────┘
                   │
                   ▼
        ┌─────────────────────┐
        │ Notebook            │
        │ Uses widget values  │
        │ for business logic  │
        └─────────────────────┘
```

**Example: Dev-to-Prod Promotion**

```yaml
# databricks.yml
bundle:
  name: etl_bundle

variables:
  cluster_size:
    description: "Cluster worker count"

  table_prefix:
    description: "Schema/table prefix"

targets:
  dev:
    default: true
    variables:
      cluster_size: "2"
      table_prefix: "dev_"

  prod:
    variables:
      cluster_size: "8"
      table_prefix: "prod_"

resources:
  jobs:
    etl_job:
      tasks:
        - task_key: transform
          notebook_task:
            notebook_path: ./notebooks/etl.py
            base_parameters:
              table_prefix: ${var.table_prefix}
          new_cluster:
            num_workers: ${var.cluster_size}
```

Deploy to dev:

```bash
databricks bundle deploy -t dev
```

Deploy to prod:

```bash
databricks bundle deploy -t prod
# Automatically uses cluster_size=8 and table_prefix=prod_
```

---
