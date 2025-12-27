# 15 Indexing Strategies

### Index Types in PostgreSQL

| Index Type | Use Case                             | Performance               |
| ---------- | ------------------------------------ | ------------------------- |
| B-tree     | Default; equality/range queries      | Fast for most queries     |
| Hash       | Equality comparisons only            | Fast for exact match      |
| GiST       | Complex data types, full-text search | Flexible but slower       |
| GIN        | Arrays, JSONB, full-text search      | Very fast for containment |
| BRIN       | Large sequential data, time-series   | Smallest index size       |
| SP-GiST    | Spatial, multidimensional data       | Specialized use           |

### Creating Indexes

```sql
-- Simple index (B-tree by default)
CREATE INDEX idx_employees_email ON employees(email);

-- Composite (multi-column) index
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Partial index (only non-active records)
CREATE INDEX idx_active_employees ON employees(salary)
WHERE is_active = TRUE;

-- Covering index (includes additional columns for index-only scans)
CREATE INDEX idx_emp_covering ON employees(department_id)
INCLUDE (first_name, salary);

-- Descending index (for ORDER BY DESC optimization)
CREATE INDEX idx_salary_desc ON employees(salary DESC);

-- Expression index (indexing computed values)
CREATE INDEX idx_upper_email ON employees(UPPER(email));

-- GIN index for full-text search
CREATE INDEX idx_article_search ON articles
USING GIN (to_tsvector('english', body));

-- GIN index for JSONB
CREATE INDEX idx_config_type ON settings
USING GIN (config);

-- BRIN index (for large tables with sequential ordering)
CREATE INDEX idx_events_time ON events
USING BRIN (event_timestamp);

-- Unique index (enforces uniqueness)
CREATE UNIQUE INDEX idx_username_unique ON users(username);

-- Conditional unique index (multiple NULLs allowed)
CREATE UNIQUE INDEX idx_email_not_deleted
ON users(email) WHERE is_deleted = FALSE;
```

### Index-Only Scans

```sql
-- Index-only scan: query satisfied entirely from index
CREATE INDEX idx_emp_covering ON employees(department_id)
INCLUDE (first_name, salary);

-- This query can use index-only scan (no table access needed)
SELECT first_name, salary
FROM employees
WHERE department_id = 2;

-- Check if index-only scan is used
EXPLAIN (ANALYZE, BUFFERS)
SELECT first_name, salary
FROM employees
WHERE department_id = 2;
-- Look for "Index-Only Scan" in output
```

### Index Maintenance

```sql
-- Analyze table statistics (helps query planner)
ANALYZE employees;

-- Vacuum (remove dead rows)
VACUUM FULL employees;  -- Aggressive, locks table

-- VACUUM ANALYZE (combined)
VACUUM ANALYZE employees;

-- List indexes on a table
SELECT * FROM pg_indexes WHERE tablename = 'employees';

-- Check index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes;

-- Find unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;

-- Drop unused index
DROP INDEX IF EXISTS idx_unused;

-- Rebuild index (removes bloat)
REINDEX INDEX idx_employees_email;

-- Check index size
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes;
```

### Indexing Best Practices

```sql
-- 1. Index columns frequently used in WHERE, JOIN, ORDER BY
CREATE INDEX idx_dept_id ON employees(department_id);
CREATE INDEX idx_hire_date ON employees(hire_date);

-- 2. Avoid indexing low-cardinality columns (boolean, status)
-- Index might be slower than sequential scan for many duplicates

-- 3. Consider column order in composite indexes
-- Most selective first (or follow query order)
CREATE INDEX idx_emp_composite ON employees(department_id, salary DESC);

-- 4. Partial indexes for filtered queries
CREATE INDEX idx_active_employees ON employees(salary)
WHERE is_active = TRUE;

-- 5. Monitor and remove unused indexes
-- Unused indexes slow down inserts/updates without helping queries

-- 6. Use EXPLAIN ANALYZE to verify index usage
EXPLAIN ANALYZE
SELECT * FROM employees WHERE salary > 50000;
```
