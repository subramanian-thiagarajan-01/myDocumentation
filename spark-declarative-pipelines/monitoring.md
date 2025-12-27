# 11. Monitoring & Observability

### 11.1 DLT Event Log Structure

Every pipeline writes events to `storage_location/_event_log` (Delta format, incremental).

**Key event types**:

- `create_update`: Pipeline run started
- `flow_definition`: Table metadata (schema, input/output datasets)
- `flow_progress`: Execution metrics (rows, duration, data quality)
- `update_end`: Pipeline run completed

### 11.2 Querying Event Logs

```sql
-- Get all tables in latest update
SELECT origin.update_id, origin.flow_name, details:flow_definition.schema
FROM event_log(table(my_table))
WHERE event_type = 'flow_definition'
ORDER BY timestamp DESC
LIMIT 100

-- Get row count metrics
SELECT
    origin.flow_name,
    TRY_CAST(details:flow_progress.metrics.num_output_rows AS BIGINT) as rows_output,
    TRY_CAST(details:flow_progress.metrics.num_upserted_rows AS BIGINT) as rows_upserted
FROM event_log(table(my_table))
WHERE event_type = 'flow_progress'
```

### 11.3 Data Quality Metrics

```sql
SELECT
    row_expectations.dataset,
    row_expectations.name,
    SUM(row_expectations.passed_records) as passed,
    SUM(row_expectations.failed_records) as failed
FROM (
    SELECT
        explode(
            from_json(
                details:flow_progress:data_quality:expectations,
                "array<struct<name: string, dataset: string, passed_records: int, failed_records: int>>"
            )
        ) row_expectations
    FROM event_log(table(my_table))
    WHERE event_type = 'flow_progress'
)
GROUP BY row_expectations.dataset, row_expectations.name
```

### 11.4 Lineage & Data Provenance

UI shows visual DAG. Event log contains:

- `flow_definition.input_datasets`: Table(s) this table reads from
- `flow_definition.output_dataset`: Name of this table
- Timestamp, duration, status (success/failure/skipped)
