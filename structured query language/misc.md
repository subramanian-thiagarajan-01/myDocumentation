# 19 Misc

## Advanced Techniques & Tricky Interview Questions

### NULL Handling Edge Cases

```sql
-- CRITICAL: NULL in IN clause
SELECT * FROM employees
WHERE department_id IN (1, 2, NULL);  -- Returns rows with dept 1 or 2, NOT rows with NULL

-- Correct way (use OR with IS NULL)
SELECT * FROM employees
WHERE department_id IN (1, 2) OR department_id IS NULL;

-- CRITICAL: NULL in NOT IN
SELECT * FROM employees
WHERE department_id NOT IN (1, 2, NULL);  -- Returns NO rows!
-- Because: department_id NOT IN (1, 2, NULL) = department_id <> 1 AND <> 2 AND <> NULL
-- <> NULL is always UNKNOWN, so entire expression is UNKNOWN

-- Correct way (use NOT EXISTS)
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM departments d
    WHERE d.department_id IN (1, 2)
    AND e.department_id = d.department_id
);

-- NULL comparisons
SELECT
    NULL = NULL AS null_equals_null,        -- UNKNOWN (not TRUE!)
    NULL <> NULL AS null_not_equals,       -- UNKNOWN
    NULL IS NULL AS null_is_null,          -- TRUE
    NULL IS NOT NULL AS null_is_not_null;  -- FALSE

-- CASE with NULL
SELECT
    CASE
        WHEN salary IS NULL THEN 'No Salary'
        WHEN salary < 30000 THEN 'Low'
        ELSE 'OK'
    END AS salary_status
FROM employees;

-- COALESCE vs CASE
SELECT
    COALESCE(commission, 0) AS commission_filled,        -- Simple NULL replacement
    CASE WHEN commission IS NULL THEN 0 ELSE commission END AS commission_case;  -- Explicit
```

### Complex JOIN Scenarios

```sql
-- Interview Q: What if join columns have NULLs?
CREATE TABLE a (id INT, value VARCHAR);
CREATE TABLE b (id INT, description VARCHAR);

INSERT INTO a VALUES (1, 'A'), (2, 'B'), (NULL, 'C');
INSERT INTO b VALUES (1, 'Desc1'), (2, 'Desc2'), (NULL, 'Desc3');

-- INNER JOIN: Only returns 2 rows (1, A, Desc1) and (2, B, Desc2)
-- The NULL row doesn't match NULL row because NULL = NULL is UNKNOWN, not TRUE
SELECT * FROM a INNER JOIN b ON a.id = b.id;

-- LEFT JOIN: Returns 3 rows (includes NULL row from left table)
SELECT * FROM a LEFT JOIN b ON a.id = b.id;

-- Solution if you want to match NULLs:
SELECT * FROM a
FULL OUTER JOIN b ON a.id = b.id OR (a.id IS NULL AND b.id IS NULL);

-- Interview Q: Join with duplicate keys
CREATE TABLE orders (order_id INT, customer_id INT);
CREATE TABLE items (order_id INT, item_id VARCHAR);

INSERT INTO orders VALUES (1, 100), (2, 100);
INSERT INTO items VALUES (1, 'A'), (1, 'B'), (2, 'C');

-- Result: Cartesian product per order
-- Order 1 matches 2 items, order 2 matches 1 item = 3 rows (not 2!)
SELECT o.*, i.item_id
FROM orders o
JOIN items i ON o.order_id = i.order_id;

-- Interview Q: Multiple join conditions
SELECT e.*, p.project_name
FROM employees e
JOIN projects p ON e.employee_id = p.lead_id AND p.status = 'active';
-- Both conditions must be true

-- Interview Q: CROSS JOIN Cartesian product
SELECT e.employee_id, d.department_id
FROM employees e
CROSS JOIN departments d;
-- Result: 100 employees * 10 departments = 1000 rows
```

### Aggregation Edge Cases

```sql
-- SUM of NULL = NULL (not 0!)
SELECT SUM(bonus) FROM employees;  -- NULL if all bonus are NULL

-- COUNT(*) vs COUNT(column)
SELECT
    COUNT(*) AS total_rows,              -- 100
    COUNT(bonus) AS non_null_bonus;      -- 75 (NULLs excluded)

-- AVG ignores NULLs
SELECT AVG(commission) FROM employees;  -- Average of non-null commissions only

-- GROUP BY with HAVING and aggregate
SELECT
    department_id,
    COUNT(*) AS num_emp,
    AVG(salary) AS avg_sal
FROM employees
WHERE salary > 30000              -- WHERE: BEFORE grouping (filter rows)
GROUP BY department_id
HAVING COUNT(*) > 5;              -- HAVING: AFTER grouping (filter groups)

-- Interview Q: Empty GROUP BY
SELECT COUNT(*) FROM employees GROUP BY;  -- ERROR: GROUP BY requires columns

-- Interview Q: Non-aggregated column in SELECT
SELECT department_id, first_name, COUNT(*)  -- ERROR if first_name not in GROUP BY
FROM employees
GROUP BY department_id;

-- Correct:
SELECT department_id, COUNT(*) FROM employees GROUP BY department_id;
```

### Subquery Complexity

```sql
-- Scalar subquery in SELECT
SELECT
    e.employee_id,
    (SELECT AVG(salary) FROM employees) AS company_avg,
    (SELECT COUNT(*) FROM orders WHERE customer_id = e.employee_id) AS orders_count
FROM employees e;

-- Subquery in HAVING
SELECT
    department_id,
    AVG(salary)
FROM employees
GROUP BY department_id
HAVING AVG(salary) > (SELECT AVG(salary) FROM employees);

-- Nested subquery
SELECT * FROM employees
WHERE employee_id IN (
    SELECT employee_id FROM (
        SELECT e.employee_id
        FROM employees e
        WHERE salary > (
            SELECT AVG(salary) FROM employees
        )
    ) high_earners
);

-- Interview Q: What if subquery returns multiple rows?
SELECT * FROM employees
WHERE salary IN (SELECT salary FROM high_earners);  -- OK, IN handles multiple

SELECT * FROM employees
WHERE salary = (SELECT salary FROM high_earners);   -- ERROR if multiple rows!

-- Correct:
SELECT * FROM employees
WHERE salary = (SELECT MAX(salary) FROM high_earners);
```

### Window Function Tricky Cases

```sql
-- Interview Q: Window function with empty OVER()
SELECT
    employee_id,
    salary,
    AVG(salary) OVER () AS company_avg  -- Empty OVER() = all rows
FROM employees;

-- Interview Q: ROW_NUMBER vs RANK vs DENSE_RANK
SELECT
    employee_id,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,     -- 1, 2, 3, 4
    RANK() OVER (ORDER BY salary DESC) AS rank,              -- 1, 2, 2, 4 (gap)
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank   -- 1, 2, 2, 3 (no gap)
FROM employees;

-- Top N per group with window functions
WITH ranked AS (
    SELECT *,
        RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
    FROM employees
)
SELECT * FROM ranked WHERE dept_rank <= 3;  -- Top 3 per department

-- Interview Q: Frame with default vs explicit
SELECT
    date,
    value,
    SUM(value) OVER (ORDER BY date) AS default_frame,  -- ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    SUM(value) OVER (ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS full_window
FROM data;
```

### Complex CASE Statements

```sql
-- Simple CASE (compare to single value)
SELECT
    CASE department_id
        WHEN 1 THEN 'Sales'
        WHEN 2 THEN 'IT'
        WHEN 3 THEN 'HR'
        ELSE 'Other'
    END AS dept_name
FROM employees;

-- Searched CASE (complex conditions)
SELECT
    CASE
        WHEN salary < 30000 THEN 'Entry'
        WHEN salary BETWEEN 30000 AND 60000 THEN 'Mid'
        WHEN salary > 60000 AND years_employed > 5 THEN 'Senior'
        WHEN salary > 100000 THEN 'Executive'
        ELSE 'Unknown'
    END AS level
FROM employees;

-- Nested CASE
SELECT
    CASE
        WHEN salary > 100000 THEN
            CASE department_id
                WHEN 1 THEN 'Senior Sales'
                WHEN 2 THEN 'Senior IT'
                ELSE 'Senior Other'
            END
        ELSE 'Standard'
    END AS role_level
FROM employees;

-- CASE in WHERE clause
SELECT * FROM employees
WHERE CASE
    WHEN department_id = 1 THEN salary > 50000
    WHEN department_id = 2 THEN salary > 60000
    ELSE salary > 40000
END;

-- CASE for conditional aggregation
SELECT
    department_id,
    COUNT(*) AS total,
    COUNT(CASE WHEN salary > 50000 THEN 1 END) AS high_earners,
    COUNT(CASE WHEN salary <= 50000 THEN 1 END) AS low_earners,
    SUM(CASE WHEN status = 'active' THEN salary ELSE 0 END) AS active_payroll
FROM employees
GROUP BY department_id;
```

---

## Full-Text Search

PostgreSQL provides powerful full-text search capabilities for efficient text querying.

### Full-Text Search Components

```sql
-- 1. Create tsvector (text search vector)
SELECT to_tsvector('english', 'The quick brown fox') AS vector;
-- Result: 'brown':3 'fox':4 'quick':2

-- 2. Create tsquery (text search query)
SELECT to_tsquery('english', 'quick & fox') AS query;

-- 3. @@ operator (matches)
SELECT 'The quick brown fox'::tsvector @@ to_tsquery('quick & fox');

-- Example table
CREATE TABLE articles (
    article_id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    body TEXT,
    search_vector TSVECTOR
);

-- Create trigger to maintain search vector
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector('english',
        COALESCE(NEW.title, '') || ' ' || COALESCE(NEW.body, ''));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_trigger
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW
EXECUTE FUNCTION update_search_vector();
```

### Full-Text Search Queries

```sql
-- Basic search
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'database & performance');

-- AND query
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgres & tuning');

-- OR query
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgres | mysql');

-- NOT query (exclude)
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgres & !deprecated');

-- Phrase search
SELECT * FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'query optimization');

-- Ranking results
SELECT
    article_id,
    title,
    ts_rank(search_vector, query) AS rank
FROM articles,
    to_tsquery('english', 'database performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Better ranking with normalization
SELECT
    article_id,
    title,
    ts_rank_cd(search_vector, query, 32) AS rank  -- 32 = all options
FROM articles,
    to_tsquery('english', 'database') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### Search Index and Optimization

```sql
-- Create GIN index for FTS
CREATE INDEX idx_article_search ON articles
USING GIN (search_vector);

-- Search with GIN index (faster)
SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'optimization')
ORDER BY ts_rank(search_vector, to_tsquery('english', 'optimization')) DESC;

-- Configuration with stemming
CREATE TEXT SEARCH CONFIGURATION english_custom (COPY = english);

-- Use custom configuration
SELECT to_tsvector('english_custom', 'databases running quickly');
-- 'databas':1 'quickli':3 'run':2 (stemmed)
```

---

## Azure PostgreSQL Integration

### Connecting to Azure Database for PostgreSQL

```sql
-- Connection string format
postgresql://username@servername:password@servername.postgres.database.azure.com:5432/databasename

-- Using psql
psql -h servername.postgres.database.azure.com -U username@servername -d databasename -p 5432

-- Connection parameters
Host: your-server-name.postgres.database.azure.com
Port: 5432 (default)
Database: postgres (or your database name)
User: username@your-server-name
Password: your-password
SSL: Required (use sslmode=require)

-- Sample connection (psql)
psql -h myserver.postgres.database.azure.com \
     -U myuser@myserver \
     -d mydb \
     --set=sslmode=require
```

### Firewall Rules & Network Security

```sql
-- Configure firewall rules in Azure Portal or CLI
-- Allow connection from your IP address

-- Connection test
telnet servername.postgres.database.azure.com 5432  -- Verify port access

-- Check connection status
\conninfo  -- In psql, shows current connection

-- Common connection issues
-- 1. Firewall blocking: Configure firewall rule to allow client IP
-- 2. SSL required: Add sslmode=require to connection string
-- 3. Wrong credentials: Verify username@servername format
```

### Performance Considerations for Azure PostgreSQL

```sql
-- 1. Connection pooling (essential for Azure)
-- Use pgBouncer or application-level pooling
-- Azure supports up to 5000 connections per server

-- 2. Query optimization
-- Monitor slow queries using Azure Portal
-- Use EXPLAIN ANALYZE for optimization

-- 3. Scaling
-- Use Azure CLI or Portal to scale vCores
-- Monitor CPU, memory, I/O metrics

-- 4. Backups
-- Azure PostgreSQL provides automatic backups (7-35 days)
-- Geo-redundant backup available

-- Check server version and settings
SHOW server_version;
SHOW shared_preload_libraries;

-- Common tuning for Azure PostgreSQL
SET shared_buffers = '262144';    -- 25% of available RAM
SET effective_cache_size = '1048576';  -- 75% of available RAM
SET work_mem = '16384';           -- Per query operation
SET maintenance_work_mem = '65536';
SET random_page_cost = 1.1;       -- Favor indexes on SSD
```

### Azure PostgreSQL Advanced Features

```sql
-- Private endpoint connectivity (VNet integration)
-- Configured in Azure Portal
-- Highly secure, no public internet access

-- Replica setup for HA
-- Read replicas automatically created in Azure Portal
-- Replication lag monitoring via metrics

-- Extensions supported
CREATE EXTENSION pgcrypto;        -- Cryptographic functions
CREATE EXTENSION pg_trgm;         -- Trigram support
CREATE EXTENSION uuid-ossp;       -- UUID generation
CREATE EXTENSION postgis;         -- Geospatial queries

-- Monitor Azure PostgreSQL
SELECT * FROM pg_stat_statements;  -- Query statistics (if enabled)
SELECT * FROM pg_stat_database;    -- Database statistics
```

### Troubleshooting Azure PostgreSQL Connections

```sql
-- Common error: "Unknown host"
-- Solution: Verify server name format (servername.postgres.database.azure.com)

-- Common error: "no pg_hba.conf entry for host"
-- Solution: Configure firewall rules in Azure Portal

-- Common error: "SSL SYSCALL error: Connection refused"
-- Solution: Add --set=sslmode=require to connection

-- Connection pool exhaustion
-- Solution: Implement connection pooling, increase connection limit

-- Monitor current connections
SELECT
    datname,
    usename,
    state,
    query,
    query_start
FROM pg_stat_activity
WHERE datname IS NOT NULL
ORDER BY query_start;

-- Terminate problematic connection
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'mydb' AND state = 'active';
```

---

## Additional Optimization Techniques

### Query Caching and Materialization

```sql
-- Materialized views for heavy aggregations
CREATE MATERIALIZED VIEW sales_summary AS
SELECT
    DATE_TRUNC('month', order_date)::DATE AS month,
    product_id,
    SUM(quantity) AS total_qty,
    SUM(amount) AS total_revenue,
    COUNT(DISTINCT customer_id) AS unique_customers
FROM orders
GROUP BY DATE_TRUNC('month', order_date), product_id;

-- Refresh MV
REFRESH MATERIALIZED VIEW sales_summary;

-- Non-blocking refresh (maintains read access)
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
-- Requires unique index on MV for concurrent refresh

-- Create unique index for concurrent refresh
CREATE UNIQUE INDEX idx_sales_summary
ON sales_summary (month, product_id);
```

### Partitioning for Large Tables

```sql
-- Range partitioning (most common)
CREATE TABLE orders (
    order_id INT,
    order_date DATE,
    customer_id INT,
    amount NUMERIC
) PARTITION BY RANGE (YEAR(order_date));

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- List partitioning
CREATE TABLE customers (
    customer_id INT,
    region VARCHAR(20),
    name VARCHAR(100)
) PARTITION BY LIST (region);

CREATE TABLE customers_us PARTITION OF customers
    FOR VALUES IN ('US_WEST', 'US_EAST', 'US_CENTRAL');

CREATE TABLE customers_emea PARTITION OF customers
    FOR VALUES IN ('EUROPE', 'MIDDLE_EAST', 'AFRICA');

-- Benefits:
-- 1. Faster queries on specific partitions
-- 2. Easier archival of old data
-- 3. Parallel query execution
-- 4. Faster index creation/deletion

-- Query performance improvement with partitions
EXPLAIN ANALYZE
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31';
-- Only scans 2024 partition, not entire table
```

### Concurrency and Locking

```sql
-- Monitor locks
SELECT
    l.locktype,
    l.database,
    l.relation,
    l.page,
    l.tuple,
    l.granted,
    a.usename,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted
ORDER BY a.query_start;

-- Lock levels
-- AccessShare: SELECT
-- RowShare: SELECT FOR SHARE
-- RowExclusive: UPDATE, DELETE, INSERT
-- ShareUpdateExclusive: VACUUM, ANALYZE
-- Share: CREATE INDEX
-- ShareRowExclusive: Complex transactions
-- Exclusive: LOCK TABLE ... IN EXCLUSIVE MODE
-- AccessExclusive: ALTER TABLE, DROP TABLE

-- Explicit locking
BEGIN;
    LOCK TABLE employees IN EXCLUSIVE MODE;
    -- No other transactions can access this table
    UPDATE employees SET salary = salary * 1.1 WHERE department_id = 1;
COMMIT;

-- SELECT with locking
SELECT * FROM employees
WHERE employee_id = 1
FOR UPDATE;  -- Row-level exclusive lock

SELECT * FROM employees
WHERE employee_id = 1
FOR SHARE;   -- Row-level shared lock (other SELECT FOR SHARE allowed)

SELECT * FROM employees
WHERE employee_id = 1
FOR UPDATE SKIP LOCKED;  -- Skip locked rows (useful for job queues)
```

---

## Conclusion

This comprehensive guide covers expert-level SQL concepts for PostgreSQL interviews. Key takeaways:

**Master These Concepts:**

1. **Joins** - Understand NULL handling, duplicate keys, performance implications
2. **Window Functions** - Powerful for ranking, running totals, comparisons
3. **Subqueries** - Know when to use correlated vs non-correlated
4. **Transactions** - ACID properties and isolation levels
5. **Indexing** - Choose right types, monitor usage, avoid premature optimization
6. **Query Optimization** - Use EXPLAIN ANALYZE, identify bottlenecks
7. **CTEs & Recursive Queries** - Clean code for hierarchical data
8. **Stored Procedures** - Encapsulate business logic, improve security

**Interview Preparation:**

- Practice complex JOINs with NULL values
- Explain execution plans using EXPLAIN ANALYZE
- Discuss normalization trade-offs
- Know when to use window functions vs GROUP BY
- Understand isolation levels and transaction handling
- Be ready for tricky NULL handling questions

**Azure PostgreSQL Specifics:**

- Connection pooling is essential
- Firewall and SSL configuration
- Monitoring via Azure Portal
- Scaling vCore capacity as needed

This guide should serve as your complete reference for SQL interviews. The key is deep understanding, not rote memorization.
