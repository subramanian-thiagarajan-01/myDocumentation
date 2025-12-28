# 1. Foundational Concept

**What is a Databricks Asset Bundle (DAB)?**

A Databricks Asset Bundle is a **declarative, version-controlled packaging format** that enables **Infrastructure-as-Code (IaC)** for Databricks. It is a single deployable unit containing:

- Notebook and Python code (source files)
- Databricks resource definitions (jobs, pipelines, clusters, dashboards, etc.)
- Environment configurations (dev, staging, prod)
- Parameterized variables
- Deployment metadata and identity settings

**Why It Matters**

- **Environment Portability**: Deploy the same code to multiple environments with different configurations
- **CI/CD Integration**: Automate deployments via GitHub Actions, Azure DevOps, or similar systems
- **Version Control**: All assets tracked in Git; no manual UI-based resource creation
- **Reproducibility**: Identical deployments across workspaces; no configuration drift
- **Single Source of Truth**: Git becomes the authoritative state for your Databricks infrastructure

---
