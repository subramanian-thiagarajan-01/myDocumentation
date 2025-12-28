# 13. Bundle Lifecycle Commands

**Concept**

The Databricks CLI is the execution engine for bundle operations.

**Core Commands**

| Command           | Purpose                               | Notes                                    |
| ----------------- | ------------------------------------- | ---------------------------------------- |
| `bundle validate` | Syntax check; outputs bundle identity | Safe to run anytime                      |
| `bundle plan`     | Preview changes (diff)                | Shows create/update/delete actions       |
| `bundle deploy`   | Deploy resources; upload files        | Idempotent; safe to re-run               |
| `bundle run`      | Execute a job or pipeline             | Runs deployed job; not a deployment step |
| `bundle destroy`  | Delete all deployed resources         | **Permanent; use carefully**             |

**Validate Before Deploy**

```bash
databricks bundle validate -t prod
# Output:
# Name: my_bundle
# Target: prod
# Workspace: https://prod-workspace.cloud.databricks.com
# Validation OK!
```

**Deploy with Approval**

```bash
databricks bundle deploy -t prod
# Shows diff; prompts for confirmation

# Skip prompt:
databricks bundle deploy -t prod --auto-approve
```

**Run a Job**

```bash
# Run deployed job
databricks bundle run -t prod my_job

# With timeout
databricks bundle run -t prod my_job --timeout 3600
```

**List Deployed Resources**

```bash
databricks bundle summary -t prod
```

**Common Flag Options**

| Flag                    | Purpose                              |
| ----------------------- | ------------------------------------ |
| `-t, --target <target>` | Target environment (dev, prod, etc.) |
| `--var "key=value"`     | Override variable at deploy-time     |
| `--cluster-id <id>`     | Override cluster (dev mode only)     |
| `--auto-approve`        | Skip confirmation prompts            |
| `--force`               | Override Git branch validation       |

---
