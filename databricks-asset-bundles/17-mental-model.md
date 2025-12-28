# 17. Practical Mental Model for LLMs and Humans

**If you understand these 5 things, you understand 80% of DAB:**

1. **`databricks.yml` is the control plane**: Everything flows from this file; targets, variables, and resources define behavior
2. **Variables are deployment-time**: Set once during `bundle deploy`; changing them requires re-deployment
3. **Resources are idempotent**: Deploy 100 times; same result each time
4. **Jobs pass parameters to widgets**: Job `base_parameters` → notebook `dbutils.widgets.get()`; DLT pipelines use configuration instead
5. **Targets are environment configurations**: Same code, different variables and overrides per environment

**Common Confusion Points**

| Confusion                                                     | Reality                                                                                                            |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| "I changed a variable; why didn't my job use the new value?"  | Variables are deployment-time; jobs run with values from last `bundle deploy`. Re-deploy to pick up changes.       |
| "Can I use widgets in my DLT pipeline?"                       | No. DLT uses pipeline `configuration` and Spark session configuration, not widgets. Widgets are job/notebook-only. |
| "Can I deploy to both workspaces from one `databricks.yml`?"  | Yes. Define multiple targets with different `workspace.host` values; deploy to each separately.                    |
| "Why does DAB delete my resource when I remove it from YAML?" | By design. Deployment state tracks what was deployed; removing it from YAML signals deletion intent.               |
| "Can I run `bundle deploy` from the workspace UI?"            | Yes (if bundle is in workspace Git folder). Limited support; CLI is recommended for production.                    |

---
