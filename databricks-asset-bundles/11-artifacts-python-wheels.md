# 11. Artifacts: Python Wheels and Build Artifacts

**Concept**

Artifacts are **built dependencies** packaged as part of a bundle (e.g., Python wheels, JAR files).

**Python Wheel Artifacts**

```yaml
artifacts:
  my_wheel:
    type: whl # Artifact type
    build: "poetry build" # Build command
    path: . # Relative to databricks.yml
    files:
      - source: pyproject.toml
        path: pyproject.toml
      - source: src/
        path: src/

resources:
  jobs:
    my_wheel_job:
      tasks:
        - task_key: wheel_task
          python_wheel_task:
            package_name: my_package
            entry_point: main
          libraries:
            - whl: ${artifacts.my_wheel.path}
```

**Build Workflow**

1. Run `databricks bundle deploy`
2. CLI executes: `poetry build`
3. Wheel file is created locally
4. Wheel is uploaded to workspace
5. Job references the uploaded wheel

**Supported Artifact Types**

- `whl` (Python wheels)
- `jar` (Java/Scala JARs)

---
