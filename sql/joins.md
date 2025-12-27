# 05 Joins: The Complete Deep-Dive

Joins combine rows from multiple tables based on related columns.

### 1. INNER JOIN (Intersection)

Returns only matching rows from both tables.

```sql
-- Basic INNER JOIN
SELECT
    e.employee_id,
    e.first_name,
    d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;

-- Multiple INNER JOINs
SELECT
    e.employee_id,
    e.first_name,
    d.department_name,
    j.job_title
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id
INNER JOIN jobs j ON e.job_id = j.job_id;

-- INNER JOIN with WHERE clause
SELECT e.*, d.department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id
WHERE d.department_name = 'Sales';

-- Self INNER JOIN (joining table to itself)
SELECT
    e1.employee_id,
    e1.first_name AS employee,
    e2.first_name AS manager
FROM employees e1
INNER JOIN employees e2 ON e1.manager_id = e2.employee_id;
```

**Interview Question:** _What rows does INNER JOIN return if the join column contains NULL values?_
**Answer:** INNER JOIN does NOT return rows where the join condition is NULL, because NULL != NULL (NULL comparisons are always UNKNOWN). This is a common gotcha!

### 2. LEFT JOIN (LEFT OUTER JOIN)

Returns all rows from left table + matching rows from right table (NULLs for non-matches).

```sql
-- Basic LEFT JOIN
SELECT
    e.employee_id,
    e.first_name,
    d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;

-- Finding unmatched rows (LEFT JOIN with WHERE IS NULL)
SELECT e.*
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
WHERE d.department_id IS NULL;  -- Employees with no valid department

-- Multiple LEFT JOINs (common pattern)
SELECT
    e.employee_id,
    e.first_name,
    d.department_name,
    m.first_name AS manager_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

### 3. RIGHT JOIN (RIGHT OUTER JOIN)

Returns all rows from right table + matching rows from left table (NULLs for non-matches).

```sql
-- RIGHT JOIN (equivalent to LEFT JOIN with tables reversed)
SELECT
    e.employee_id,
    e.first_name,
    d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;

-- This is typically rewritten as LEFT JOIN for consistency:
SELECT
    e.employee_id,
    e.first_name,
    d.department_name
FROM departments d
LEFT JOIN employees e ON e.department_id = d.department_id;
```

### 4. FULL OUTER JOIN (FULL JOIN)

Returns all rows from both tables (NULLs where no match).

```sql
-- FULL OUTER JOIN
SELECT
    COALESCE(e.employee_id, 0) AS employee_id,
    e.first_name,
    d.department_id,
    d.department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.department_id;

-- Finding unmatched rows in either table
SELECT
    e.employee_id,
    e.first_name,
    d.department_id,
    d.department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.department_id
WHERE e.employee_id IS NULL OR d.department_id IS NULL;
```

### 5. CROSS JOIN

Cartesian product: combines every row from left table with every row from right table.

```sql
-- CROSS JOIN (no ON clause)
SELECT * FROM employees CROSS JOIN departments;
-- Returns: rows_in_employees * rows_in_departments

-- Use cases (rare but important):
-- 1. Generate all combinations
SELECT e.employee_id, d.department_id
FROM employees e
CROSS JOIN departments d
LIMIT 10;

-- 2. Generate dates
SELECT
    d::DATE,
    (d + INTERVAL '1 day')::DATE
FROM GENERATE_SERIES('2025-01-01', '2025-12-31', '1 day'::INTERVAL) d;
```

### 6. Self-Join

Joining a table to itself for hierarchical or comparative data.

```sql
-- Manager-Employee hierarchy (most common)
SELECT
    e.employee_id,
    e.first_name || ' ' || e.last_name AS employee,
    m.employee_id AS manager_id,
    m.first_name || ' ' || m.last_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
ORDER BY m.employee_id, e.first_name;

-- Finding colleagues in the same department
SELECT
    e1.employee_id,
    e1.first_name AS employee1,
    e2.employee_id,
    e2.first_name AS employee2
FROM employees e1
INNER JOIN employees e2
    ON e1.department_id = e2.department_id
    AND e1.employee_id < e2.employee_id;  -- Avoid duplicates
```

### JOIN Tricky Interview Scenarios

#### Scenario 1: Join on NULL values

```sql
-- Tables with NULLs in join column
CREATE TABLE table_a (id INT, value VARCHAR);
CREATE TABLE table_b (id INT, description VARCHAR);

INSERT INTO table_a VALUES (1, 'A'), (2, 'B'), (NULL, 'C');
INSERT INTO table_b VALUES (1, 'Desc1'), (2, 'Desc2'), (NULL, 'Desc3');

-- INNER JOIN result: only (1, 'A', 'Desc1') and (2, 'B', 'Desc2')
-- NULL row is NOT matched with NULL row (NULL = NULL is UNKNOWN, not TRUE)
SELECT a.*, b.description
FROM table_a a
INNER JOIN table_b b ON a.id = b.id;

-- LEFT JOIN result: includes (NULL, 'C', NULL)
SELECT a.*, b.description
FROM table_a a
LEFT JOIN table_b b ON a.id = b.id;
```

#### Scenario 2: Duplicate join keys

```sql
CREATE TABLE orders (order_id INT, customer_id INT);
CREATE TABLE order_items (order_id INT, item_id INT, quantity INT);

INSERT INTO orders VALUES (1, 100), (2, 100);
INSERT INTO order_items VALUES (1, 'A', 2), (1, 'B', 1), (2, 'C', 3);

-- RESULT: Cartesian product within each order
-- Order 1 has 2 items, creates 2 rows
SELECT o.*, oi.item_id
FROM orders o
INNER JOIN order_items oi ON o.order_id = oi.order_id;
-- 3 rows total (not 2!)
```

#### Scenario 3: Multiple join conditions

```sql
-- Joining on multiple columns
SELECT a.*, b.data
FROM table_a a
INNER JOIN table_b b
    ON a.col1 = b.col1
    AND a.col2 = b.col2;  -- Both conditions must be true

-- Asymmetric join conditions
SELECT a.*, b.data
FROM table_a a
INNER JOIN table_b b
    ON a.id = b.a_id
    AND b.status = 'active'  -- Can include non-join columns in ON clause
WHERE a.department = 'Sales';
```
