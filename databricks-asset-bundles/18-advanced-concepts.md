# 18. Advanced Concepts (Beyond the Pareto 80/20)

**Listed for completeness; less critical for productivity**

- **Dynamic Value References**: `{{job.run_id}}`, `{{job.parameters.x}}` for context-aware job parameters
- **Permissions**: `permissions` section for granular ACLs on bundle resources
- **Source-Linked Deployments**: Dev mode option to reference workspace code instead of synced copies
- **Binding**: `databricks bundle bind` to associate existing workspace resources with bundle management
- **Environments**: Different cloud providers (AWS, Azure, GCP) supported; configuration differs slightly
- **MLflow Experiments & Models**: Bundles can manage registered models and experiments
- **Dashboards & SQL Assets**: Bundles support AI/BI Dashboards and SQL queries as resources
- **Workspace Git Folders**: Deploy bundles directly from workspace Git integration (experimental)

---
