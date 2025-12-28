# 16. Opinionated Best Practices (The DAB Philosophy)

**Implicit Assumptions Built Into DAB**

1. **Git is Source of Truth**: All infrastructure defined in version control; no UI-driven resource creation
2. **Declarative Over Imperative**: YAML configuration defines desired state; CLI executes it
3. **Environment Parity**: Same code, different configurations via targets
4. **Reproducibility**: Deployments are deterministic and repeatable across workspaces
5. **Separation of Concerns**: Deployment identity ≠ Execution identity (via `run_as`)

**Recommended Directory Structure**

```
my-dab-project/
├── .github/
│   └── workflows/
│       └── deploy.yml              # CI/CD pipeline
├── databricks.yml                  # Bundle definition
├── include:
│   - resources/*.yml
│   - targets/*.yml
├── resources/
│   ├── jobs.yml
│   ├── pipelines.yml
│   ├── clusters.yml
│   └── dashboards.yml
├── targets/
│   ├── dev.yml
│   ├── staging.yml
│   └── prod.yml
├── notebooks/
│   ├── etl/
│   │   ├── ingest.py
│   │   └── transform.py
│   ├── validation/
│   │   └── quality_checks.py
│   └── shared/
│       └── utils.py
├── src/
│   └── my_package/
│       ├── __init__.py
│       ├── transforms.py
│       └── schemas.py
├── tests/
│   ├── unit/
│   │   └── test_transforms.py
│   └── integration/
│       └── test_jobs.py
├── pyproject.toml                  # Python dependencies (for wheels)
├── .gitignore
└── README.md                        # Bundle onboarding
```

**Deployment Checklist**

- [ ] `databricks bundle validate -t <target>` passes
- [ ] All variables have values (no missing required vars)
- [ ] Git branch matches `bundle.git.branch` (prevents accidental deployments from wrong branch)
- [ ] Service principal (for `run_as`) exists and has necessary permissions
- [ ] Secrets are stored in Databricks Secret Scopes, not hardcoded
- [ ] CI/CD pipeline uses service principal credentials, not user PATs
- [ ] `sync.exclude` filters out `.git`, `__pycache__`, `.venv`, etc.

---
