# 20. Quick Reference: From Concept to Deployment

**Minimal DAB Project (10 minutes to first deploy)**

```bash
# 1. Initialize bundle from template
databricks bundle init default-python

# 2. Edit databricks.yml with your job definition
# (customize name, notebook path, cluster config)

# 3. Create your notebook
mkdir notebooks
# Create notebooks/my_notebook.py

# 4. Validate
databricks bundle validate -t dev

# 5. Deploy
databricks bundle deploy -t dev

# 6. Run
databricks bundle run -t dev hello_job
```

**Template-Generated `databricks.yml`**

```yaml
bundle:
  name: my-bundle

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://my-workspace.cloud.databricks.com

resources:
  jobs:
    sample_job:
      name: Sample Job
      tasks:
        - task_key: hello_world
          notebook_task:
            notebook_path: ./notebooks/hello.py
          new_cluster:
            spark_version: 13.3.x-scala2.12
            node_type_id: i3.xlarge
            num_workers: 2
```

---
