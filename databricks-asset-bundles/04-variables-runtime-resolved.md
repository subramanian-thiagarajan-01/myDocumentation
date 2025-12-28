# 4. Variables: Runtime-Resolved Parameters

**Concept**

Bundle variables are **deployment-time parameters** injected into resource definitions. They enable parameterized deployments without hardcoding environment-specific values.

**Sources of Variable Values** (in order of precedence)

1. CLI flag: `--var="my_var=value"`
2. Environment variable: `BUNDLE_VAR_my_var=value`
3. Target override in `databricks.yml`
4. `variable-overrides.json` file
5. `default` value in variable definition

**Variable Definition**

```yaml
variables:
  cluster_id:
    description: "Cluster ID for job tasks"
    default: "1234-567890-abcde123"

  environment:
    description: "Deployment environment"
    default: "dev"

  instance_profile_arn:
    description: "AWS instance profile ARN for Glue catalog"
    # No default; must be provided at deployment
```

**Variable Reference Syntax**

Variables are referenced using `${var.variable_name}` throughout `databricks.yml`:

```yaml
resources:
  jobs:
    my_job:
      tasks:
        - task_key: my_task
          existing_cluster_id: ${var.cluster_id}
          notebook_task:
            notebook_path: ./my_notebook.py
```

**Target-Specific Variable Overrides**

```yaml
targets:
  dev:
    variables:
      cluster_id: "1111-111111-dev-cluster"
      environment: "dev"

  prod:
    variables:
      cluster_id: "2222-222222-prod-cluster"
      environment: "prod"
```

**Critical Distinction: Deployment-Time vs. Runtime**

- **Bundle variables** are resolved at deployment time (when `databricks bundle deploy` runs)
- **Job parameters** and **notebook widgets** are resolved at runtime (when the job executes)
- Once deployed, changing a variable requires re-deployment; changing parameters at run-time does NOT reflect the deployment

---
