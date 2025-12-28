# 3. Bundle Targets (Environments)

**Concept**

Targets are **named deployment contexts**. Each target can have its own workspace, configuration, and resource overrides.

**Typical Target Structure**

```yaml
targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://dev-workspace.cloud.databricks.com

  staging:
    mode: development
    workspace:
      host: https://staging-workspace.cloud.databricks.com

  prod:
    mode: production
    workspace:
      host: https://prod-workspace.cloud.databricks.com
    root_path: /Shared/prod/.bundle/${bundle.name}/${bundle.target}
    run_as:
      service_principal_name: <sp-id>
```

**Target-Level Overrides**

Each target can override:

- `variables` (environment-specific values)
- `resources` (target-specific job configurations, cluster sizes, schedules)
- `workspace` (host URL, root_path for artifact storage)
- `mode` (development or production)
- `run_as` (execution identity)
- `presets` (name_prefix, pipeline_development, jobs_max_concurrent_runs, etc.)

**Deployment Mode Behaviors**

- **Development Mode** (`mode: development`):
  - Pauses all job schedules
  - Enables concurrent job runs
  - Disables deployment lock (faster iteration)
  - Marks DLT pipelines as `development: true`
  - Allows cluster override via `--cluster-id` flag
- **Production Mode** (`mode: production`):
  - Enforces Git branch validation
  - Requires service principal for `run_as`
  - Keeps deployment lock enabled
  - Respects configured cluster definitions

---
