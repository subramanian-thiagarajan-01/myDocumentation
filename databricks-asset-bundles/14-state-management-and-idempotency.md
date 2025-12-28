# 14. State Management and Idempotency

**Concept**

DAB maintains **deployment state** to ensure idempotent, repeatable deployments.

**State File Location**

```
.databricks/
└── bundle/
    └── <target>/
        ├── .state               # Deployment state
        ├── variables-overrides.json
        └── sync-excludes.txt
```

**Idempotency Guarantees**

- **First deploy**: Creates all resources
- **Nth deploy with no changes**: No Databricks API calls made
- **Deploy after edit**: Only changed resources updated
- **Remove from YAML, redeploy**: Resource deleted from workspace

**Resource Identity**

A bundle resource is uniquely identified by:

1. Bundle name
2. Bundle target
3. Workspace host
4. Resource key (e.g., `jobs.my_job`)

Changing any of these is considered a **new resource**:

```bash
# These are different resources
databricks bundle deploy -t dev
databricks bundle deploy -t prod

# Renaming bundle invalidates deployment
bundle:
  name: old_name  # Changes to new_name → new deployment identity
```

---
