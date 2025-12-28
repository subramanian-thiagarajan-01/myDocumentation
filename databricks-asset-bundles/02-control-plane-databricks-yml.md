# 2. The Control Plane: `databricks.yml`

**Concept**

`databricks.yml` is the **single source of truth** for a bundle. It defines bundle identity, deployment targets, variables, resources, and behavior.

**Minimum Valid Configuration**

```yaml
bundle:
  name: my-bundle

targets:
  dev:
    default: true
```

**Core Top-Level Sections**

| Section       | Purpose                                                               |
| ------------- | --------------------------------------------------------------------- |
| `bundle`      | Bundle metadata: `name` (required), `git`, `deployment`, `cluster_id` |
| `targets`     | Environment definitions (dev, staging, prod)                          |
| `variables`   | Parameterized values; resolved at deployment time                     |
| `resources`   | Deployable assets (jobs, pipelines, notebooks, clusters)              |
| `include`     | Path globs to include additional YAML files                           |
| `artifacts`   | Build instructions for Python wheels, JARs, etc.                      |
| `sync`        | File synchronization rules to workspace                               |
| `permissions` | Access control (CAN_VIEW, CAN_RUN, CAN_MANAGE)                        |
| `run_as`      | Identity for execution (user_name or service_principal_name)          |

---
