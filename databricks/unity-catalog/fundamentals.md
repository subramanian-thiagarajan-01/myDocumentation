# 01 Unity Catalog

## 1. Unity Catalog Fundamentals

### 1.1 Definition and Purpose

**What It Is:**
Unity Catalog (UC) is Databricks' unified governance solution for data and AI assets across clouds. It provides a single place to manage access control, auditing, lineage, and data discovery.

**Key Problem It Solves:**
Before UC, Databricks used workspace-level Hive metastore with limited governance capabilities. UC centralizes governance across multiple workspaces and clouds.

**Core Properties:**

- **Access Control:** Fine-grained permissions (catalog, schema, table, column, row levels)
- **Auditing:** Comprehensive audit logs for all data access
- **Lineage:** Automated capture of data flow (table-to-table, column-to-column)
- **Data Discovery:** Search and explore data assets across the organization
- **Quality Monitoring:** Integration with Databricks Lakehouse Monitoring

**Interview Question:** _Why Unity Catalog over Hive Metastore?_

- Centralized governance across workspaces
- Built-in RBAC and ABAC
- Automated lineage capture
- Multi-cloud metadata layer
- Direct compatibility with Delta Sharing

---

## 2. Namespace & Object Model

### 2.1 Three-Level Namespace

**Structure:**

```
<catalog>.<schema>.<table>
```

Example:

```sql
SELECT * FROM prod_catalog.sales_schema.transactions_table;
```

**Comparison with Hive Metastore:**

- Hive: `<database>.<table>` (2-level)
- UC: `<catalog>.<schema>.<table>` (3-level)

**Why Three Levels?**

- **Catalog:** Logical boundary for environment (prod, dev, staging)
- **Schema:** Logical grouping of related tables (sales, marketing, finance)
- **Table:** Actual data object

### 2.2 Object Types

| Object Type           | Purpose                            | Example               |
| --------------------- | ---------------------------------- | --------------------- |
| **Catalog**           | Top-level container                | `prod_catalog`        |
| **Schema**            | Namespace for tables/views         | `sales_schema`        |
| **Table**             | Structured data (managed/external) | `transactions`        |
| **View**              | Virtual table                      | `vw_monthly_sales`    |
| **Materialized View** | Pre-computed view with refresh     | `mv_customer_summary` |
| **Volume**            | Non-tabular data storage           | `ml_models_volume`    |
| **Function**          | User-defined functions             | `mask_pii()`          |

### 2.3 Object Hierarchy

```
Metastore (1 per region)
└── Catalog (multiple)
    └── Schema (multiple)
        ├── Tables
        ├── Views
        ├── Materialized Views
        ├── Volumes
        └── Functions
```

**Code Example:**

```sql
-- Create catalog
CREATE CATALOG IF NOT EXISTS dev_catalog;

-- Create schema
CREATE SCHEMA IF NOT EXISTS dev_catalog.finance;

-- Create table
CREATE TABLE dev_catalog.finance.invoices (
  invoice_id BIGINT,
  amount DECIMAL(10,2),
  created_date DATE
) USING DELTA;

-- Query with full namespace
SELECT * FROM dev_catalog.finance.invoices;
```

**Interview Pitfall:**

- **Cannot skip levels:** Must specify `catalog.schema.table`, not just `catalog.table`
- **Default catalog/schema:** Can be set per session to avoid repetition
