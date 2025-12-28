# 02 Metastore, Metadata

## 3. Metastore Architecture

### 3.1 Metastore Concept

**What It Is:**
The metastore is the top-level container for Unity Catalog metadata. It stores:

- Catalog definitions
- Schema definitions
- Table metadata
- Permissions
- Lineage information
- Audit logs

**Key Constraint:** One metastore per region (not per workspace)

### 3.2 Metastore Creation (Azure Example)

**Prerequisites:**

1. ADLS Gen2 storage account
2. Storage container for metastore root
3. Access Connector for Azure Databricks (managed identity)

**Creation Steps:**

```sql
-- Step 1: Create storage credential (links to Access Connector)
CREATE STORAGE CREDENTIAL adls_credential
USING AZURE_MANAGED_IDENTITY
WITH (
  managed_identity_id = '/subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Databricks/accessConnectors/{connector-name}'
);

-- Step 2: Create metastore (done via Databricks Account Console or REST API)
-- Not directly via SQL - requires account admin privileges

-- Step 3: Verify metastore
DESCRIBE METASTORE;
```

**Metastore Assignment:**

- Multiple workspaces can be assigned to the same metastore
- Workspaces in the same region share the metastore
- Enables cross-workspace data sharing and governance

**Interview Question:** _Can you have multiple metastores in one region?_

- **No.** One metastore per region is the design constraint.
- Workspaces in the region must use that metastore or none at all.

### 3.3 Managed Storage Locations

**Hierarchy of Managed Storage:**

1. **Metastore-level:** Default location for all catalogs
2. **Catalog-level:** Overrides metastore default for specific catalog
3. **Schema-level:** Overrides catalog default for specific schema

**Example:**

```sql
-- Metastore default: abfss://metastore@storage.dfs.core.windows.net/

-- Create catalog with custom location
CREATE CATALOG prod_catalog
MANAGED LOCATION 'abfss://prod@storage.dfs.core.windows.net/';

-- Create schema with custom location
CREATE SCHEMA prod_catalog.sales
MANAGED LOCATION 'abfss://prod@storage.dfs.core.windows.net/sales/';

-- Managed table inherits schema location
CREATE TABLE prod_catalog.sales.orders (...);
-- Stored at: abfss://prod@storage.dfs.core.windows.net/sales/orders/
```

**Interview Pitfall:**

- If no location specified, uses parent's managed location
- Changing managed location **does not migrate** existing data

---

## 4. Storage & Security Abstractions

### 4.1 Storage Credentials

**Purpose:**
Define authentication to cloud storage without exposing secrets in table definitions.

**Types by Cloud:**

**Azure:**

```sql
-- Managed Identity (recommended)
CREATE STORAGE CREDENTIAL azure_mi
USING AZURE_MANAGED_IDENTITY
WITH (managed_identity_id = '...');

-- Service Principal
CREATE STORAGE CREDENTIAL azure_sp
USING AZURE_SERVICE_PRINCIPAL
WITH (
  directory_id = 'tenant-id',
  application_id = 'client-id',
  client_secret = 'secret'
);
```

**AWS:**

```sql
-- IAM Role
CREATE STORAGE CREDENTIAL aws_role
USING AWS_IAM_ROLE
WITH (role_arn = 'arn:aws:iam::account:role/role-name');
```

**GCP:**

```sql
-- Service Account
CREATE STORAGE CREDENTIAL gcp_sa
USING GCP_SERVICE_ACCOUNT
WITH (email = 'sa@project.iam.gserviceaccount.com');
```

### 4.2 External Locations

**Purpose:**
Map storage credentials to specific storage paths. Enables governed access to external data.

**Example:**

```sql
-- Create external location
CREATE EXTERNAL LOCATION s3_raw_data
URL 's3://my-bucket/raw-data/'
WITH (STORAGE CREDENTIAL aws_role);

-- Grant access to external location
GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION s3_raw_data TO data_engineers;
```

**Interview Question:** _Storage Credential vs External Location?_

- **Storage Credential:** Authentication mechanism (HOW to authenticate)
- **External Location:** Specific path + credential (WHERE + HOW)
- External Location references a Storage Credential

### 4.3 Access Connector (Azure-Specific)

**What It Is:**
Azure-managed identity that Databricks uses to access ADLS Gen2 without storing credentials.

**Setup Flow:**

1. Create Access Connector in Azure Portal
2. Grant Access Connector RBAC permissions on ADLS Gen2 (Storage Blob Data Contributor)
3. Create Storage Credential in UC referencing the Access Connector
4. Create External Location using the Storage Credential

**Interview Pitfall:**

- Access Connector must have proper RBAC on storage account
- Common error: "403 Forbidden" due to missing RBAC
