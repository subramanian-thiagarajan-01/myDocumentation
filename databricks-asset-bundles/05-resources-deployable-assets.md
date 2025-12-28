# 5. Resources: The Deployable Assets

**Concept**

Resources are **Databricks objects managed by the bundle**. They are declarative definitions of jobs, pipelines, notebooks, clusters, etc.

**Pareto Core Resources**

| Resource Type | Purpose                                            | Typical Use Case                                       |
| ------------- | -------------------------------------------------- | ------------------------------------------------------ |
| `jobs`        | Orchestrate tasks: notebooks, Python scripts, JARs | Production workflows, scheduled ETL                    |
| `pipelines`   | Delta Live Tables (DLT) workflows                  | Declarative data transformations, continuous ingestion |
| `notebooks`   | Versioned notebook artifacts                       | Code checked into Git, deployed to workspace           |
| `clusters`    | Reusable compute definitions                       | Optional; usually referenced by job tasks              |

**Resource Declaration Structure**

```yaml
resources:
  jobs:
    my_job: # Unique resource key
      name: my_job_name # Display name in workspace
      tasks:
        - task_key: my_task
          notebook_task:
            notebook_path: ./my_notebook.py
          new_cluster:
            spark_version: 13.3.x-scala2.12
            node_type_id: i3.xlarge
            num_workers: 4
      schedule:
        quartz_cron_expression: "0 0 * * *" # Daily at midnight
        pause_status: "UNPAUSED"
      max_concurrent_runs: 1
      timeout_seconds: 3600
```

**Resource Uniqueness**

Each resource must have a **unique programmatic key** within its type:

- `jobs.my_job` (key: `my_job`)
- `pipelines.etl_pipeline` (key: `etl_pipeline`)
- `notebooks.shared_lib` (key: `shared_lib`)

The key is used internally by the CLI; the `name` field is what users see in Databricks UI.

**Idempotency**

Resources are **idempotent**:

- First deployment creates the resource
- Subsequent deployments update existing resource (no duplication)
- Removed resources are deleted from workspace if previously deployed

---
