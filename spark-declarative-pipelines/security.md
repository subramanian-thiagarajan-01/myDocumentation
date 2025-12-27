# 13. Security & Governance

### 13.1 Unity Catalog Integration

DLT tables automatically reside in Unity Catalog (if workspace enabled). Ownership and permissions:

```sql
-- DLT creates table owned by current user/service principal
GRANT SELECT ON TABLE my_catalog.my_schema.my_table TO `data-engineers`;
GRANT MODIFY ON TABLE my_catalog.my_schema.my_table TO `data-platform-team`;
```

### 13.2 Object Ownership

Every table has an owner (user, service principal, or group). Owner can grant/revoke privileges:

```sql
ALTER TABLE my_table OWNER TO `platform-admin`;
```

### 13.3 Row-Level and Column-Level Security

**Row-level security**: Dynamic views with `current_user()`:

```python
@dlt.table
def users_filtered():
    df = spark.read.table("users_raw")
    if spark.sql("SELECT current_user()").collect()[0][0] == "analyst@company.com":
        return df.filter("region = 'US'")  # Analyst sees only US data
    return df
```

**Column-level security**: Column masking via SQL UDF (not native to DLT, requires wrapper function).

### 13.4 Secrets Management

Use Databricks Secrets API (not environment variables):

```python
@dlt.table
def secure_data():
    api_key = dbutils.secrets.get(scope="my-scope", key="api-key")
    return spark.read.option("api_key", api_key).parquet("...")
```
