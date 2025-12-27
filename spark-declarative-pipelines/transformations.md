# 5. Transformations

### 5.1 SQL vs Python DLT Pipelines

**SQL Pipeline (notebook .sql file)**:

```sql
CREATE STREAMING TABLE raw_orders AS
SELECT * FROM stream(catalog.schema.external_table);

CREATE LIVE TABLE cleaned_orders (
  CONSTRAINT no_negative_amount EXPECT (amount > 0) ON VIOLATION DROP ROW
) AS
SELECT id, customer_id, amount, order_date
FROM STREAM(raw_orders)
WHERE amount > 0 AND order_date <= current_date();

CREATE MATERIALIZED VIEW daily_revenue AS
SELECT DATE(order_date) as order_date, SUM(amount) as revenue
FROM live.cleaned_orders
GROUP BY DATE(order_date);
```

**Python Pipeline**:

```python
import dlt

@dlt.create_streaming_table(name="raw_orders")
def raw_orders():
    return spark.readStream.table("external_orders")

@dlt.table
@dlt.expect_or_drop("no_negative_amount", "amount > 0")
def cleaned_orders():
    return dlt.read_stream("raw_orders") \
        .filter("amount > 0") \
        .filter("order_date <= current_date()")

@dlt.table
def daily_revenue():
    return dlt.read("cleaned_orders") \
        .groupBy("order_date") \
        .agg({"amount": "sum"})
```

**Key difference**: Python allows for complex logic, loops, conditionals; SQL is pure declarative.

### 5.2 Incremental Transformations

For stateless operations (filters, selects), all table types are inherently incremental. For stateful operations (aggregations, joins), only streaming tables guarantee true incremental processing on classic compute.

**Serverless compute** (as of 2025): Materialized views get incremental refresh via cost-based optimizer.

```python
@dlt.table
def incremental_aggregation():
    # On serverless, only processes new/changed data (if underlying table changed)
    return dlt.read("silver_events") \
        .groupBy("user_id", "date") \
        .agg(count("*").alias("event_count"))
```

### 5.3 Aggregations and Joins

**Aggregations in streaming tables**:

```python
@dlt.create_streaming_table(name="streaming_user_events")
def streaming_user_events():
    return spark.readStream.format("cloudFiles") \
        .load("s3://events/").withWatermark("event_time", "1 hour")

@dlt.create_streaming_table(name="user_event_count")
def user_event_count():
    return dlt.read_stream("streaming_user_events") \
        .groupBy(
            F.window("event_time", "5 minutes"),
            "user_id"
        ) \
        .agg(count("*").alias("event_count"))
```

**Joins**: Streaming table + streaming table requires watermarks. Streaming + static is safe.
