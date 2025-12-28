# 8. Notebook Packaging and Synchronization

**Concept**

Notebooks are **first-class bundle artifacts**. They are checked into Git, deployed to the workspace, and versioned as part of the bundle.

**Notebook Resource Definition**

```yaml
resources:
  notebooks:
    shared_library:
      path: ./notebooks/shared_lib.py
      language: python
```

**File Synchronization: The `sync` Section**

By default, DAB syncs all files defined in bundle configuration to the workspace. Fine-tune synchronization:

```yaml
sync:
  include:
    - "notebooks/**/*.py"
    - "notebooks/**/*.sql"
    - "scripts/**/*.py"
    - "src/**/*.py"

  exclude:
    - ".git/"
    - "*.pyc"
    - "__pycache__/"
    - ".venv/"
    - "tests/" # Optional: exclude test files

  paths:
    - ./notebooks
    - ./scripts
    - ./src
```

**Deployment Path**

Files are deployed to a workspace path managed by the bundle:

```
/Users/<user>/.bundle/<bundle-name>/<target>/
```

Or with custom `root_path`:

```yaml
targets:
  prod:
    workspace:
      root_path: /Shared/prod/.bundle/${bundle.name}/${bundle.target}
```

---
