# 06 Misc

## 15. Common Interview Questions & Scenarios

### Q1: How do you migrate from Hive Metastore to Unity Catalog?

**Answer:**

1. **Enable UC on workspace:** Assign metastore to workspace
2. **Create UC catalogs/schemas:** Mirror existing Hive database structure
3. **Sync external tables:** Use `SYNC` command or manual `CREATE EXTERNAL TABLE`
4. **Sync managed tables:** Use `DEEP CLONE` or `CREATE TABLE AS SELECT`
5. **Update code:** Change table references from `database.table` to `catalog.schema.table`
6. **Migrate permissions:** Recreate `GRANT` statements in UC
7. **Validate lineage and audit logs:** Ensure governance is working

**Code Example:**

```sql
-- Hive table
SELECT * FROM sales_db.orders;

-- Sync to UC external table
CREATE EXTERNAL TABLE prod_catalog.sales.orders
LOCATION 'abfss://data@storage.dfs.core.windows.net/sales/orders/'
AS SELECT * FROM sales_db.orders;

-- Deep clone for managed table
CREATE TABLE prod_catalog.sales.orders_managed
DEEP CLONE sales_db.orders;
```

### Q2: How do you implement multi-region data sharing with UC?

**Answer:**

- **Problem:** One metastore per region, but need to share data across regions
- **Solution:** Use Delta Sharing
  - Create share in Region A
  - Add tables to share
  - Create recipient in Region B
  - Recipient queries data via Delta Sharing protocol (no data replication)

### Q3: What happens if you drop a catalog with existing tables?

**Answer:**

```sql
DROP CATALOG dev_catalog;
-- ERROR: Cannot drop catalog with existing schemas
```

**Correct Approach:**

```sql
-- Drop with CASCADE
DROP CATALOG dev_catalog CASCADE;
-- Deletes catalog, all schemas, and all tables (including data for managed tables)
```

**Interview Pitfall:**

- `CASCADE` deletes ALL data in managed tables
- Use with extreme caution in production

### Q4: How do you troubleshoot "Permission Denied" errors in UC?

**Answer:**

1. Check if user has `USE CATALOG` and `USE SCHEMA` permissions
2. Check if user has `SELECT` permission on table
3. For external tables, check if user has access to External Location
4. For external tables, check if Storage Credential has valid cloud permissions
5. Check audit logs to see which permission check failed

**Code Example:**

```sql
-- Check current user permissions
SHOW GRANTS ON CATALOG prod_catalog;
SHOW GRANTS ON SCHEMA prod_catalog.sales;
SHOW GRANTS ON TABLE prod_catalog.sales.orders;

-- Check external location access
SHOW GRANTS ON EXTERNAL LOCATION s3_raw_data;
```

### Q5: How do you enforce data retention policies in UC?

**Answer:**

- **Table-level:** Use Delta table properties `delta.deletedFileRetentionDuration`
- **Governance-level:** Use UC permissions to restrict DELETE/TRUNCATE
- **Audit-level:** Monitor audit logs for data deletion

**Example:**

```sql
-- Set retention to 30 days
ALTER TABLE prod_catalog.sales.orders
SET TBLPROPERTIES ('delta.deletedFileRetentionDuration' = 'interval 30 days');

-- Vacuum old files
VACUUM prod_catalog.sales.orders RETAIN 720 HOURS;  -- 30 days
```

---

## 16. Critical Exam Topics Summary

**High-Priority Areas:**

1. **Three-level namespace:** Understand catalog → schema → table hierarchy
2. **Managed vs External tables:** Lifecycle, DROP behavior, use cases
3. **Metastore architecture:** One per region, multi-workspace assignment
4. **Storage Credentials & External Locations:** How they work together
5. **Permissions model:** GRANT/REVOKE, object hierarchy, row/column security
6. **Volumes:** Purpose, managed vs external, replacing DBFS
7. **Lineage:** Automated capture, column-level lineage
8. **Delta Sharing:** Protocol, shares, recipients
9. **Materialized Views:** Incremental refresh, use cases
10. **Lakehouse Federation:** Query external systems without ETL

**Common Pitfalls:**

- Forgetting `USE CATALOG` permission when granting `SELECT`
- Mixing up Storage Credential (auth) and External Location (path + auth)
- Not understanding DROP behavior for managed vs external tables
- Assuming DBFS and Volumes are interchangeable (they're not)
- Expecting multi-region metastore (constraint: one per region)

---

## 17. Hands-On Practice Exercises

### Exercise 1: Create End-to-End UC Catalog

```sql
-- 1. Create catalog
CREATE CATALOG practice_catalog;

-- 2. Create schema with managed location
CREATE SCHEMA practice_catalog.sales
MANAGED LOCATION 'abfss://practice@storage.dfs.core.windows.net/sales/';

-- 3. Create managed table
CREATE TABLE practice_catalog.sales.orders (
  order_id BIGINT,
  customer_id BIGINT,
  amount DECIMAL(10,2),
  order_date DATE
) USING DELTA;

-- 4. Insert data
INSERT INTO practice_catalog.sales.orders VALUES
(1, 101, 250.00, '2025-01-15'),
(2, 102, 300.00, '2025-01-16');

-- 5. Create view with row-level security
CREATE VIEW practice_catalog.sales.orders_filtered AS
SELECT * FROM practice_catalog.sales.orders
WHERE
  CASE
    WHEN is_member('admin') THEN TRUE
    ELSE amount < 500
  END;

-- 6. Grant permissions
GRANT SELECT ON VIEW practice_catalog.sales.orders_filtered TO analysts;
```

### Exercise 2: Implement Column Masking

```sql
-- 1. Create masking function
CREATE FUNCTION practice_catalog.security.mask_email(email STRING)
RETURNS STRING
RETURN CONCAT(SUBSTRING(email, 1, 3), '***@', SPLIT(email, '@')[1]);

-- 2. Create masked view
CREATE VIEW practice_catalog.sales.customers_masked AS
SELECT
  customer_id,
  name,
  CASE
    WHEN is_member('pii_access') THEN email
    ELSE practice_catalog.security.mask_email(email)
  END AS email
FROM practice_catalog.sales.customers;
```

### Exercise 3: Setup Delta Sharing

```sql
-- 1. Create share
CREATE SHARE analytics_share;

-- 2. Add tables
ALTER SHARE analytics_share ADD TABLE practice_catalog.sales.orders;

-- 3. Create recipient
CREATE RECIPIENT partner_company;

-- 4. Grant access
GRANT SELECT ON SHARE analytics_share TO RECIPIENT partner_company;
```

---

**End of Unity Catalog Interview Preparation Notes**
