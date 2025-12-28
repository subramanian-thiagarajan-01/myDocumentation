# 19. Troubleshooting and Debugging

**Common Issues and Resolutions**

| Issue                                  | Cause                                                       | Fix                                                            |
| -------------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------- |
| `Variable '<var>' is not defined`      | Variable referenced in YAML but not declared                | Add `variables` section with definition                        |
| `Path does not exist`                  | Notebook path in job definition is incorrect                | Verify notebook exists in `notebooks/` and path is relative    |
| `Permission denied`                    | Service principal lacks permissions to resource             | Grant permissions via Databricks workspace admin               |
| `Workspace host is not configured`     | Missing `workspace.host` in target                          | Add `workspace: host: https://...` to target definition        |
| `Resource already exists in workspace` | Resource created outside bundle (UI); conflicts with deploy | Run `databricks bundle bind <resource>` or delete and redeploy |
| `Git branch does not match`            | Deployed from wrong Git branch in prod target               | Use `--force` to override (not recommended in prod)            |

**Debugging Commands**

```bash
# Validate and show resolved configuration
databricks bundle validate -t prod -vvv

# Show deployment plan without applying
databricks bundle plan -t prod

# Check bundle identity and state
databricks bundle summary -t prod

# List included files
databricks bundle summary -t prod --show-includes
```

---
