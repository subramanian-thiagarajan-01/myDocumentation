# 04 Data Governance, Lineage, Observability, Lakehouse Federation

## 8. Data Governance Features

### 8.1 Object-Level Permissions

**Grant Syntax:**

```sql
GRANT <privilege> ON <object_type> <object_name> TO <principal>;
```

**Common Privileges:**

| Privilege        | Applies To | Meaning                           |
| ---------------- | ---------- | --------------------------------- |
| `USE CATALOG`    | Catalog    | View catalog, traverse to schemas |
| `USE SCHEMA`     | Schema     | View schema, traverse to tables   |
| `SELECT`         | Table/View | Query data                        |
| `MODIFY`         | Table      | Insert/Update/Delete data         |
| `CREATE TABLE`   | Schema     | Create tables in schema           |
| `CREATE SCHEMA`  | Catalog    | Create schemas in catalog         |
| `ALL PRIVILEGES` | Any object | All permissions                   |

**Example:**

```sql
-- Grant read access to table
GRANT SELECT ON TABLE prod_catalog.sales.orders TO marketing_group;

-- Grant write access to schema
GRANT CREATE TABLE, USE SCHEMA ON SCHEMA prod_catalog.sales TO data_engineers;

-- Grant full access to catalog
GRANT ALL PRIVILEGES ON CATALOG prod_catalog TO admin_group;

-- Show grants
SHOW GRANTS ON TABLE prod_catalog.sales.orders;
```

### 8.2 Row-Level Security (Dynamic Views)

**Mechanism:**
Use `current_user()` or `is_member()` functions in view definition to filter rows.

**Example:**

```sql
-- Base table (restricted access)
CREATE TABLE prod_catalog.hr.employees (
  emp_id INT,
  name STRING,
  department STRING,
  salary DECIMAL(10,2)
);

-- Dynamic view (row-level filtering)
CREATE VIEW prod_catalog.hr.employees_filtered AS
SELECT emp_id, name, department, salary
FROM prod_catalog.hr.employees
WHERE
  CASE
    WHEN is_member('hr_admins') THEN TRUE
    WHEN is_member('managers') THEN department = current_user()
    ELSE emp_id = CAST(current_user() AS INT)
  END;

-- Grant access to view, not base table
GRANT SELECT ON VIEW prod_catalog.hr.employees_filtered TO all_users;
```

**Interview Question:** _How does row-level security work?_

- View evaluates user context at query time
- `current_user()` returns logged-in user email
- `is_member('group')` checks group membership
- Base table should be restricted; grant access to view only

### 8.3 Column-Level Masking (UC Functions)

**Mechanism:**
Create SQL UDF that applies masking logic, use in views or enforce via policies.

**Example:**

```sql
-- Create masking function
CREATE FUNCTION prod_catalog.security.mask_ssn(ssn STRING)
RETURNS STRING
RETURN CONCAT('XXX-XX-', SUBSTRING(ssn, -4));

-- Create view with column masking
CREATE VIEW prod_catalog.hr.employees_masked AS
SELECT
  emp_id,
  name,
  CASE
    WHEN is_member('hr_admins') THEN ssn
    ELSE prod_catalog.security.mask_ssn(ssn)
  END AS ssn,
  salary
FROM prod_catalog.hr.employees;
```

**Databricks Runtime 13.3+ Feature:**
Column-level access control via `GRANT SELECT(column)` (limited preview as of 2025).

### 8.4 Audit Logging

**System Tables:**
Unity Catalog maintains audit logs in system tables.

**Example Query:**

```sql
-- Query audit logs (requires access to system.access)
SELECT
  event_time,
  user_identity.email AS user,
  action_name,
  request_params.full_name_arg AS object,
  response.status_code
FROM system.access.audit
WHERE action_name IN ('createTable', 'readTable', 'deleteTable')
  AND event_date >= current_date() - 7
ORDER BY event_time DESC;
```

**Interview Question:** _What is logged?_

- All data access (SELECT, INSERT, UPDATE, DELETE)
- DDL operations (CREATE, ALTER, DROP)
- Permission changes (GRANT, REVOKE)
- User identity, timestamp, object accessed

## 9. Lineage & Observability

### 9.1 Automated Lineage Capture

**What Is Captured:**

- **Table-to-table lineage:** Which tables are used to create other tables
- **Column-to-column lineage:** Which source columns feed into target columns
- **Notebook/job lineage:** Which notebooks/jobs accessed data

**Example:**

```sql
-- Create target table with lineage
CREATE TABLE prod_catalog.analytics.customer_summary AS
SELECT
  c.customer_id,
  c.name,
  SUM(o.amount) AS total_spent
FROM prod_catalog.sales.customers c
JOIN prod_catalog.sales.orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;

-- UC automatically tracks:
-- - customer_summary depends on customers and orders
-- - total_spent column depends on orders.amount
```

### 9.2 Viewing Lineage

**UI:**

- Databricks Data Explorer → Select table → Lineage tab
- Visual graph showing upstream and downstream dependencies

**API/SQL:**

```sql
-- Query lineage (system table - Databricks Runtime 13.0+)
SELECT
  source_table_full_name,
  target_table_full_name,
  source_column_name,
  target_column_name
FROM system.access.table_lineage
WHERE target_table_full_name = 'prod_catalog.analytics.customer_summary';
```

### 9.3 Column-Level Lineage

**Example:**
If `customer_summary.total_spent` is derived from `orders.amount`, UC tracks this relationship.

**Interview Question:** _How is lineage captured?_

- Databricks runtime intercepts Spark operations
- Parses SQL/DataFrame operations to extract dependencies
- Stores lineage metadata in Unity Catalog metastore
- **Limitation:** May not capture lineage from external tools (e.g., dbt, Airflow) unless they use Databricks APIs

---

## 10. Lakehouse Federation

### 10.1 Concept

**Definition:**
Query external data systems (Snowflake, PostgreSQL, Redshift, MySQL) directly from Databricks without copying data.

**Key Feature:**

- Query external system using Spark SQL syntax
- Data stays in source system
- UC governs access via connections

**Supported Systems (as of Databricks Runtime 13.3+):**

- Snowflake
- PostgreSQL
- MySQL
- Amazon Redshift
- Azure SQL Database
- Google BigQuery

### 10.2 Setup Example (PostgreSQL)

**Step 1: Create Connection**

```sql
CREATE CONNECTION postgres_prod
TYPE postgresql
OPTIONS (
  host 'prod-db.example.com',
  port '5432',
  database 'sales_db',
  user 'readonly_user',
  password 'encrypted_password'  -- Use secret scope in practice
);
```

**Step 2: Query External Table**

```sql
-- Query PostgreSQL table via federation
SELECT *
FROM postgres_prod.public.customer_transactions
WHERE order_date >= '2025-01-01'
LIMIT 100;
```

**Step 3: Join with Delta Table**

```sql
-- Join external data with UC table
SELECT
  t.transaction_id,
  t.amount,
  c.customer_name
FROM postgres_prod.public.customer_transactions t
JOIN prod_catalog.sales.customers c ON t.customer_id = c.customer_id;
```

### 10.3 Performance Considerations

**Pushdown Optimization:**
Databricks pushes filters to external system when possible.

```sql
-- Good: Filter pushed to PostgreSQL
SELECT * FROM postgres_prod.public.orders WHERE status = 'SHIPPED';

-- Bad: All data pulled, then filtered in Spark
SELECT * FROM postgres_prod.public.orders WHERE UPPER(status) = 'SHIPPED';
```

**Interview Question:** _When to use Federation vs ETL?_

- **Federation:** Real-time queries, small result sets, infrequent access
- **ETL:** Large data volumes, frequent queries, complex transformations
