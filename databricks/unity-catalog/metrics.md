# 05 Metrics, User Identity, Delta Sharing, Secret Management

## 11. UC Metrics (Semantic Layer - 2025 Feature)

### 11.1 Purpose

**What Are UC Metrics?**
Business-defined metrics stored in Unity Catalog. Provides consistent metric definitions across tools (Databricks SQL, BI tools, notebooks).

**Key Concepts:**

- **Metric View:** Container for metrics
- **Measures:** Aggregations (SUM, AVG, COUNT, etc.)
- **Dimensions:** Group-by attributes
- **Window Functions:** Time-based calculations

### 11.2 Creating Metric Views

**Example:**

```sql
CREATE METRIC VIEW prod_catalog.analytics.revenue_metrics
AS
SELECT
  order_date,
  product_category,
  region,
  SUM(amount) AS total_revenue,
  COUNT(DISTINCT customer_id) AS unique_customers,
  AVG(amount) AS avg_order_value
FROM prod_catalog.sales.orders
GROUP BY order_date, product_category, region;
```

**Querying Metric View:**

```sql
-- Use metric view like a regular table
SELECT
  product_category,
  SUM(total_revenue) AS revenue
FROM prod_catalog.analytics.revenue_metrics
WHERE order_date >= '2025-01-01'
GROUP BY product_category;
```

### 11.3 Benefits

- **Single source of truth:** Metrics defined once, used everywhere
- **Governance:** UC permissions apply to metrics
- **Lineage:** Track metric dependencies
- **BI tool integration:** Expose metrics to Tableau, Power BI

**Interview Question:** _UC Metrics vs Materialized Views?_

- **UC Metrics:** Business logic layer, consistent definitions, BI integration
- **Materialized Views:** Performance optimization, pre-computed results

## 12. User & Identity Management

### 12.1 Databricks Users

**User Types:**

- **Workspace users:** Created directly in Databricks
- **Federated users:** Synced from Entra ID (Azure AD), Okta, OneLogin

**Recommended Approach:**
Use identity federation for centralized user management.

### 12.2 Entra ID Integration (Azure)

**Setup:**

1. Configure SCIM provisioning in Azure Portal
2. Map Azure AD groups to Databricks groups
3. Users automatically synced to Databricks

**Group-Based Access:**

```sql
-- Create Databricks group
CREATE GROUP data_engineers;

-- Add users (via SCIM sync, not manual)
-- Users added in Azure AD automatically appear in Databricks

-- Grant permissions to group
GRANT USE CATALOG, USE SCHEMA ON CATALOG prod_catalog TO data_engineers;
GRANT SELECT ON SCHEMA prod_catalog.sales TO data_engineers;
```

### 12.3 Service Principals

**What Are Service Principals?**
Non-human identities for automation (pipelines, jobs, applications).

**Example:**

```sql
-- Create service principal (via Databricks Admin Console or API)
-- Then grant permissions

GRANT SELECT ON CATALOG prod_catalog TO `service-principal-id`;
```

**Interview Question:** _Users vs Service Principals?_

- **Users:** Human identities, interactive access
- **Service Principals:** Machine identities, automated workflows

## 13. Delta Sharing

### 13.1 Concept

**What Is Delta Sharing?**
Open protocol for secure data sharing. Share Delta tables with external organizations without copying data.

**Key Features:**

- **Open source protocol:** Works with non-Databricks recipients
- **UC-governed:** Permissions managed via UC
- **Read-only:** Recipients can query but not modify data
- **Audit logs:** Track who accessed shared data

### 13.2 Setup Example

**Step 1: Create Share**

```sql
-- Create share
CREATE SHARE sales_share;

-- Add table to share
ALTER SHARE sales_share ADD TABLE prod_catalog.sales.public_orders;
```

**Step 2: Create Recipient**

```sql
-- Create recipient (external organization)
CREATE RECIPIENT acme_corp;

-- Grant access to share
GRANT SELECT ON SHARE sales_share TO RECIPIENT acme_corp;
```

**Step 3: Generate Activation Link**

```sql
-- Generate sharing credential for recipient
-- (Done via Databricks UI or REST API)
-- Recipient uses link to access shared data
```

### 13.3 Querying Shared Data (Recipient Side)

**Non-Databricks Recipients:**
Use Delta Sharing connectors (Python, Pandas, PrestoDB, Trino).

**Example (Python):**

```python
import delta_sharing

# Profile file received from data provider
profile_file = "/path/to/config.share"

# List shared tables
client = delta_sharing.SharingClient(profile_file)
tables = client.list_all_tables()

# Load shared table as Pandas DataFrame
df = delta_sharing.load_as_pandas(f"{profile_file}#sales_share.prod_catalog.sales.public_orders")
```

**Interview Question:** _Delta Sharing vs API?_

- **Delta Sharing:** Standardized, governed, no custom API needed
- **API:** Custom code, harder to govern, more maintenance

---

## 14. Secret Management

### 14.1 Secret Scopes

**Purpose:**
Store sensitive information (passwords, API keys, tokens) securely. Secrets not exposed in code or notebooks.

**Types:**

- **Databricks-backed scope:** Secrets stored in Databricks control plane
- **Azure Key Vault-backed scope:** Secrets stored in Azure Key Vault (recommended for production)

### 14.2 Azure Key Vault Integration

**Setup:**

1. Create Azure Key Vault
2. Create secrets in Key Vault
3. Create Databricks secret scope linked to Key Vault

**CLI Example:**

```bash
# Create scope (Databricks CLI)
databricks secrets create-scope --scope production-secrets \
  --scope-backend-type AZURE_KEYVAULT \
  --resource-id /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{vault-name} \
  --dns-name https://{vault-name}.vault.azure.net/
```

**Accessing Secrets in Notebook:**

```python
# Retrieve secret
db_password = dbutils.secrets.get(scope="production-secrets", key="postgres-password")

# Use in connection
jdbc_url = f"jdbc:postgresql://prod-db.example.com:5432/sales_db"
df = spark.read \
  .format("jdbc") \
  .option("url", jdbc_url) \
  .option("dbtable", "orders") \
  .option("user", "readonly_user") \
  .option("password", db_password) \
  .load()
```

### 14.3 Secret Scope Permissions

```python
# Grant read access to secret scope
databricks secrets put-acl --scope production-secrets --principal data_engineers --permission READ

# Revoke access
databricks secrets delete-acl --scope production-secrets --principal data_engineers
```

**Interview Question:** _Why not hardcode secrets?_

- **Security risk:** Secrets exposed in code, version control, logs
- **Audit trail:** Secret access logged
- **Centralized management:** Rotate secrets without code changes
