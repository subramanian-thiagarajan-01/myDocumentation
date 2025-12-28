# 12. Authentication and Identity

**Concept**

Bundle authentication determines:

- Who deploys the bundle
- Who runs the deployed job/pipeline (via `run_as`)

**Authentication Scenarios**

| Scenario                | Type       | Method            | Use Case                      |
| ----------------------- | ---------- | ----------------- | ----------------------------- |
| Local development       | Attended   | OAuth U2M or PAT  | Developer iterating locally   |
| CI/CD (GitHub Actions)  | Unattended | OAuth M2M or PAT  | Automated deployments         |
| CI/CD (multi-workspace) | Unattended | Service Principal | Deploy to multiple workspaces |

**Authentication Setup**

Databricks CLI reads from `~/.databrickscfg` (local) or environment variables:

```bash
# Environment variables (CI/CD)
export DATABRICKS_HOST=https://my-workspace.cloud.databricks.com
export DATABRICKS_CLIENT_ID=<client-id>        # OAuth M2M
export DATABRICKS_CLIENT_SECRET=<client-secret>
```

Or:

```bash
export DATABRICKS_TOKEN=<personal-access-token>
```

**Run Identity (`run_as`)**

Separate the identity that **deploys** from the identity that **runs**:

```yaml
bundle:
  name: my_bundle

targets:
  dev:
    mode: development
    run_as:
      user_name: ${workspace.current_user.userName} # Current user

  prod:
    mode: production
    run_as:
      service_principal_name: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

**Best Practices**

- **Local development**: Run as your user (`user_name`)
- **Staging**: Run as a staging service principal
- **Production**: Always run as a production service principal
- **CI/CD**: Use service principal with minimal permissions for the environment it accesses

---
