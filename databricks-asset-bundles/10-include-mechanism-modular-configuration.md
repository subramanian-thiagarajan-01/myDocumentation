# 10. The Include Mechanism: Modular Configuration

**Concept**

`include` allows you to split `databricks.yml` across multiple files for modularity.

**Include Syntax**

```yaml
# databricks.yml (minimal)
bundle:
  name: my_bundle

include:
  - "resources/*.yml"
  - "targets/*.yml"
  - "bundle*.yml"

variables:
  env: dev
```

**Glob Patterns**

```yaml
include:
  - "resources/*.yml" # All YML files in resources/
  - "resources/**/*.yml" # Recursive; all YML in subdirs
  - "bundle.*.yml" # bundle.jobs.yml, bundle.pipelines.yml, etc.
  - "targets/prod.yml" # Specific file
```

**Common Structure**

```
my-bundle/
├── databricks.yml            # Bundle definition
├── resources/
│   ├── jobs.yml             # All job definitions
│   ├── pipelines.yml        # All pipeline definitions
│   └── clusters.yml         # Reusable cluster configs
├── targets/
│   ├── dev.yml              # Dev target overrides
│   ├── staging.yml          # Staging target overrides
│   └── prod.yml             # Prod target overrides
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

---
