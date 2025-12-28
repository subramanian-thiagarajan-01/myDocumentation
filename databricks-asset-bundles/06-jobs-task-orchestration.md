# 6. Jobs: Task Orchestration

**Concept**

Jobs define **multi-task workflows** in Databricks. A job orchestrates execution order, passes data between tasks, and manages retries/alerts.

**Key Concepts**

| Concept                | Definition                                                      |
| ---------------------- | --------------------------------------------------------------- |
| **Task**               | A single unit of work (notebook, Python script, JAR, SQL query) |
| **Task Key**           | Unique identifier within a job; used for dependencies           |
| **Task Parameters**    | Input values passed to a task                                   |
| **Job Parameters**     | Global parameters available to all tasks                        |
| **Task Dependencies**  | Define execution order and data passing                         |
| **Cluster Assignment** | Task runs on existing cluster or new cluster                    |

**Task Types (Pareto)**

```yaml
resources:
  jobs:
    multi_task_job:
      name: example_job
      tasks:
        # Notebook task
        - task_key: ingest_data
          notebook_task:
            notebook_path: ./notebooks/ingest.py
            base_parameters:
              source_path: "/mnt/data/raw/"
              target_schema: "bronze"

        # Python script task
        - task_key: transform
          python_task:
            python_file: ./scripts/transform.py
            parameters:
              - "--input=/mnt/data/bronze/"
              - "--output=/mnt/data/silver/"
          depends_on:
            - task_key: ingest_data

        # Spark JAR task
        - task_key: ml_model
          spark_jar_task:
            jar_uri: dbfs:/jars/ml_model.jar
            main_class_name: com.example.ModelTraining
          depends_on:
            - task_key: transform

      new_cluster:
        spark_version: 13.3.x-scala2.12
        node_type_id: i3.xlarge
        num_workers: 4
```

**Job Parameters Flow**

```
Bundle Variables → Job `base_parameters` → Notebook Widgets
```

Example:

```yaml
variables:
  env_name:
    default: "dev"

resources:
  jobs:
    my_job:
      tasks:
        - task_key: my_task
          notebook_task:
            notebook_path: ./my_notebook.py
            base_parameters:
              environment: ${var.env_name} # Variable interpolated here
              schema_name: "default_schema"
```

In the notebook:

```python
# Notebook receives base_parameters as widget values
environment = dbutils.widgets.get("environment")
schema_name = dbutils.widgets.get("schema_name")
```

---
