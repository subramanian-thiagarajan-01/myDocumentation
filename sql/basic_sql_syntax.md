# 02 Basic SQL Syntax & Operations

### SELECT Statement

The `SELECT` statement retrieves data from tables.

```sql
-- Basic SELECT
SELECT column1, column2 FROM table_name;

-- SELECT all columns
SELECT * FROM table_name;

-- SELECT with aliases
SELECT first_name AS "Employee Name", salary AS "Annual Salary"
FROM employees;

-- SELECT with expressions
SELECT employee_id, first_name, salary * 12 AS annual_salary
FROM employees;

-- SELECT DISTINCT (remove duplicates)
SELECT DISTINCT department_id FROM employees;

-- SELECT with LIMIT and OFFSET (pagination)
SELECT * FROM employees LIMIT 10 OFFSET 20;  -- Skip 20, get next 10

-- SELECT with column expressions
SELECT
    employee_id,
    first_name || ' ' || last_name AS full_name,  -- String concatenation
    COALESCE(middle_name, 'N/A') AS middle_name
FROM employees;
```

### FROM Clause

Specifies the source table(s).

```sql
-- FROM single table
SELECT * FROM employees;

-- FROM with table alias
SELECT e.employee_id, e.first_name
FROM employees e;

-- FROM subquery
SELECT * FROM (
    SELECT employee_id, first_name FROM employees WHERE salary > 50000
) AS high_earners;
```

### WHERE Clause

Filters rows based on conditions.

```sql
-- WHERE with single condition
SELECT * FROM employees WHERE salary > 50000;

-- WHERE with multiple conditions (AND)
SELECT * FROM employees
WHERE salary > 50000 AND department_id = 1;

-- WHERE with OR
SELECT * FROM employees
WHERE department_id = 1 OR department_id = 2;

-- WHERE with NOT
SELECT * FROM employees
WHERE NOT department_id = 1;

-- WHERE with IN
SELECT * FROM employees
WHERE department_id IN (1, 2, 3);

-- WHERE with BETWEEN
SELECT * FROM employees
WHERE salary BETWEEN 40000 AND 80000;

-- WHERE with LIKE (pattern matching)
SELECT * FROM employees
WHERE first_name LIKE 'J%';        -- Starts with J
WHERE first_name LIKE '%n';        -- Ends with n
WHERE first_name LIKE '%oh%';      -- Contains 'oh'
WHERE first_name LIKE '_ohn';      -- Exactly 4 chars, 3rd-4th are 'oh'

-- WHERE with IS NULL / IS NOT NULL
SELECT * FROM employees
WHERE middle_name IS NULL;
WHERE manager_id IS NOT NULL;
```

### ORDER BY Clause

Sorts query results.

```sql
-- ORDER BY ascending (default)
SELECT * FROM employees ORDER BY salary;

-- ORDER BY descending
SELECT * FROM employees ORDER BY salary DESC;

-- ORDER BY multiple columns
SELECT * FROM employees
ORDER BY department_id ASC, salary DESC;

-- ORDER BY column position (not recommended for clarity)
SELECT first_name, salary FROM employees ORDER BY 2 DESC;

-- ORDER BY expression
SELECT first_name, salary
FROM employees
ORDER BY salary * 12 DESC;

-- ORDER BY with NULLS handling
SELECT * FROM employees
ORDER BY commission NULLS LAST;  -- NULLs at end
ORDER BY commission NULLS FIRST; -- NULLs at beginning
```

### GROUP BY Clause

Groups rows by specified columns for aggregation.

```sql
-- Basic GROUP BY
SELECT department_id, COUNT(*) AS num_employees
FROM employees
GROUP BY department_id;

-- GROUP BY multiple columns
SELECT department_id, job_title, COUNT(*)
FROM employees
GROUP BY department_id, job_title;

-- GROUP BY with expressions
SELECT
    DATE_TRUNC('month', hire_date)::DATE AS month,
    COUNT(*) AS hires
FROM employees
GROUP BY DATE_TRUNC('month', hire_date)
ORDER BY month;

-- Important: All non-aggregated columns in SELECT must be in GROUP BY
-- This is INVALID:
-- SELECT department_id, first_name, COUNT(*)
-- FROM employees GROUP BY department_id;  -- first_name not in GROUP BY!
```

### HAVING Clause

Filters groups based on aggregate conditions (applied AFTER grouping).

```sql
-- WHERE vs HAVING (key distinction)
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
WHERE hire_date > '2020-01-01'      -- WHERE filters INDIVIDUAL rows BEFORE grouping
GROUP BY department_id
HAVING AVG(salary) > 60000;         -- HAVING filters GROUPS AFTER grouping

-- More complex HAVING conditions
SELECT
    department_id,
    COUNT(*) AS num_employees,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5 AND AVG(salary) > 55000;

-- HAVING with subquery
SELECT department_id, SUM(salary) AS total_salary
FROM employees
GROUP BY department_id
HAVING SUM(salary) > (SELECT AVG(total_salary) FROM (...));
```
