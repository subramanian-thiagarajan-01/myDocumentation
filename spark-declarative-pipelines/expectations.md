# 7. Data Quality & Expectations

### 7.1 Expectations (EXPECT, EXPECT_OR_DROP, EXPECT_OR_FAIL)

**EXPECT** (default): Log violations; write valid + invalid records

```python
@dlt.expect("email_valid", "email LIKE '%@%.%'")
def users():
    return spark.read.table("raw_users")
```

**EXPECT_OR_DROP**: Remove violating records

```python
@dlt.expect_or_drop("non_negative_balance", "balance >= 0")
def accounts():
    return spark.read.table("raw_accounts")
```

**EXPECT_OR_FAIL**: Fail entire pipeline on violation

```python
@dlt.expect_or_fail("primary_key_not_null", "customer_id IS NOT NULL")
def customers():
    return spark.read.table("raw_customers")
```

### 7.2 Advanced: Group Multiple Expectations

```python
@dlt.table
@dlt.expect_all({
    "email_valid": "email LIKE '%@%.%'",
    "email_not_null": "email IS NOT NULL",
    "age_reasonable": "age BETWEEN 0 AND 150"
})
def users():
    return spark.read.table("raw_users")
```

Use `expect_all_or_drop`, `expect_all_or_fail` for grouped actions.

### 7.3 Metrics & Monitoring

Event log stores expectation results in `details:flow_progress.data_quality.expectations`:

```sql
SELECT
    row_expectations.dataset,
    row_expectations.name,
    SUM(row_expectations.passed_records) as passing,
    SUM(row_expectations.failed_records) as failing
FROM event_log_raw,
    LATERAL FLATTEN(
        input => from_json(
            details:flow_progress:data_quality:expectations,
            "array<struct<name: string, dataset: string, passed_records: int, failed_records: int>>"
        )
    ) row_expectations
WHERE event_type = 'flow_progress'
GROUP BY row_expectations.dataset, row_expectations.name
```

### 7.4 Quarantine Pattern (Advanced)

```python
@dlt.table
def users_quarantine():
    df = spark.read.table("raw_users")
    # Capture invalid rows in a separate table
    return df.filter(
        NOT (email LIKE '%@%.%' AND age BETWEEN 0 AND 150)
    )

@dlt.table
@dlt.expect_all({
    "email_valid": "email LIKE '%@%.%'",
    "age_reasonable": "age BETWEEN 0 AND 150"
})
def users_valid():
    return spark.read.table("raw_users") \
        .filter(email LIKE '%@%.%' AND age BETWEEN 0 AND 150)
```
