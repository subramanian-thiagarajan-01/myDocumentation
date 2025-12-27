# 16 Query Optimization

### EXPLAIN and EXPLAIN ANALYZE

Understanding query execution plans is critical for optimization.

```sql
-- EXPLAIN: shows planned execution (doesn't run query)
EXPLAIN
SELECT e.*, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
WHERE e.salary > 50000;

-- Output components:
-- Seq Scan: sequential table scan (slow for large tables)
-- Index Scan: uses index (usually faster)
-- Nested Loop: join method (small joins)
-- Hash Join: join method (larger joins)
-- Merge Join: join method (pre-sorted data)

-- EXPLAIN ANALYZE: executes query and shows actual statistics
EXPLAIN ANALYZE
SELECT * FROM employees WHERE salary > 50000;

-- Output includes:
-- cost=X..Y: estimated cost (lower = faster)
-- actual time=X..Y ms: real execution time
-- rows=X: estimated rows / actual rows returned
-- Loops: how many times node executed

-- Full analysis with buffers
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM employees WHERE department_id = 1;

-- Buffers output shows:
-- Shared Blks Hit: data from cache
-- Shared Blks Read: data from disk
-- Shared Blks Dirtied: data modified

-- FORMAT options
EXPLAIN (FORMAT JSON) SELECT * FROM employees;
EXPLAIN (FORMAT XML) SELECT * FROM employees;
EXPLAIN (FORMAT YAML) SELECT * FROM employees;
```

### Common Performance Issues and Solutions

```sql
-- ISSUE 1: Sequential Scan on large table
-- Symptom: "Seq Scan on large_table"
EXPLAIN SELECT * FROM orders WHERE customer_id = 100;

-- Solution: Create index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- ISSUE 2: N+1 Query problem
-- Bad: Execute query in loop (application level)
-- SELECT * FROM orders WHERE id = 1;
-- SELECT * FROM orders WHERE id = 2;
-- ... (100 times)

-- Solution: Fetch all at once
SELECT * FROM orders WHERE id IN (1, 2, 3, ...);

-- ISSUE 3: Unnecessary joins
SELECT e.*, d.*, j.*
FROM employees e
JOIN departments d ON e.department_id = d.department_id
JOIN jobs j ON e.job_id = j.job_id;

-- Solution: Select only needed columns
SELECT e.employee_id, e.first_name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- ISSUE 4: Missing WHERE clause (full table scan)
EXPLAIN ANALYZE SELECT * FROM large_table;

-- Solution: Add filtering
SELECT * FROM large_table WHERE status = 'active';

-- ISSUE 5: Complex expressions in WHERE clause
SELECT * FROM employees
WHERE YEAR(hire_date) = 2020;  -- Can't use index!

-- Solution: Use range for indexability
SELECT * FROM employees
WHERE hire_date >= '2020-01-01' AND hire_date < '2021-01-01';

-- ISSUE 6: OR conditions preventing index usage
SELECT * FROM employees
WHERE first_name = 'John' OR first_name = 'Jane';

-- Solution: Use IN or UNION
SELECT * FROM employees WHERE first_name IN ('John', 'Jane');
```

### Query Tuning Techniques

```sql
-- 1. Use DISTINCT carefully (expensive operation)
-- Bad:
SELECT DISTINCT employee_id FROM orders;

-- Better (if goal is uniqueness):
SELECT employee_id FROM orders GROUP BY employee_id;

-- 2. Limit result set early
-- Bad:
SELECT * FROM large_table JOIN another_table...
-- Then limit in application

-- Good:
SELECT * FROM (
    SELECT * FROM large_table WHERE date > NOW() - INTERVAL '30 days'
) t
LIMIT 100;

-- 3. Rewrite correlated subqueries as joins
-- Slow (correlated):
SELECT e.* FROM employees e
WHERE salary > (
    SELECT AVG(salary) FROM employees WHERE department_id = e.department_id
);

-- Fast (join):
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department_id
)
SELECT e.* FROM employees e
JOIN dept_avg da ON e.department_id = da.department_id
WHERE e.salary > da.avg_sal;

-- 4. Use window functions instead of self-joins
-- Complex:
SELECT e1.*, e2.salary AS next_higher_salary
FROM employees e1
LEFT JOIN employees e2 ON e2.salary > e1.salary
WHERE e2.salary = (SELECT MIN(salary) FROM employees e3 WHERE e3.salary > e1.salary);

-- Simple:
SELECT *,
    LEAD(salary) OVER (ORDER BY salary) AS next_higher_salary
FROM employees;

-- 5. Batch operations
-- Slow:
FOR i IN 1..1000 LOOP
    INSERT INTO table (col1, col2) VALUES (val1, val2);
END LOOP;

-- Fast:
INSERT INTO table (col1, col2)
SELECT val1, val2 FROM source_table;

-- 6. Use COPY for bulk loading
COPY employees (employee_id, first_name, salary)
FROM '/path/to/file.csv'
WITH (FORMAT csv, HEADER true);
```

### Configuration Tuning

```sql
-- Check current settings
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW random_page_cost;

-- Common tuning parameters (postgresql.conf or SET command)
SET shared_buffers = '256MB';        -- 25% of RAM (restart needed)
SET work_mem = '64MB';               -- Per operation
SET maintenance_work_mem = '1GB';    -- For VACUUM, ANALYZE
SET effective_cache_size = '8GB';    -- Help query planner
SET random_page_cost = 1.1;          -- Favor index scans on SSD
SET seq_page_cost = 1.0;             -- Cost of sequential page read

-- Autovacuum settings
SET autovacuum = on;
SET autovacuum_vacuum_scale_factor = 0.1;
SET autovacuum_analyze_scale_factor = 0.05;
```
