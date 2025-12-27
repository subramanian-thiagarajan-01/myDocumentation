# 04 Data Aggregation Functions

### Standard Aggregate Functions

| Function     | Purpose            | Syntax                        | Null Handling                    |
| ------------ | ------------------ | ----------------------------- | -------------------------------- |
| `COUNT()`    | Count rows         | `COUNT(*)` or `COUNT(column)` | Ignores NULLs (except COUNT(\*)) |
| `SUM()`      | Sum values         | `SUM(column)`                 | Ignores NULLs                    |
| `AVG()`      | Average values     | `AVG(column)`                 | Ignores NULLs                    |
| `MIN()`      | Minimum value      | `MIN(column)`                 | Ignores NULLs                    |
| `MAX()`      | Maximum value      | `MAX(column)`                 | Ignores NULLs                    |
| `STDDEV()`   | Standard deviation | `STDDEV(column)`              | Ignores NULLs                    |
| `VARIANCE()` | Variance           | `VARIANCE(column)`            | Ignores NULLs                    |

```sql
-- COUNT examples
SELECT COUNT(*) AS total_rows FROM employees;           -- All rows
SELECT COUNT(commission) AS commissioned FROM employees; -- Non-null values
SELECT COUNT(DISTINCT department_id) AS num_departments FROM employees;

-- SUM
SELECT SUM(salary) AS total_payroll FROM employees;

-- AVG (important: ignores NULLs)
SELECT AVG(commission) FROM employees;  -- Average of non-null commissions
-- To include NULLs as 0:
SELECT AVG(COALESCE(commission, 0)) FROM employees;

-- MIN/MAX
SELECT MIN(salary) AS lowest_salary, MAX(salary) AS highest_salary FROM employees;

-- Multiple aggregates
SELECT
    COUNT(*) AS total,
    AVG(salary) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary,
    SUM(salary) AS total_payroll
FROM employees;

-- Aggregate with GROUP BY
SELECT
    department_id,
    COUNT(*) AS num_employees,
    AVG(salary) AS avg_salary,
    SUM(salary) AS total_payroll
FROM employees
GROUP BY department_id
ORDER BY total_payroll DESC;

-- Filtering aggregates (use CASE for conditional aggregation)
SELECT
    department_id,
    COUNT(*) AS total_employees,
    COUNT(CASE WHEN salary > 60000 THEN 1 END) AS high_earners,
    SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) AS active_count
FROM employees
GROUP BY department_id;
```

### String Aggregation

```sql
-- STRING_AGG: Concatenate strings (PostgreSQL specific)
SELECT
    department_id,
    STRING_AGG(first_name, ', ') AS employee_names
FROM employees
GROUP BY department_id;

-- ORDER within STRING_AGG
SELECT
    department_id,
    STRING_AGG(first_name ORDER BY first_name, ', ') AS sorted_names
FROM employees
GROUP BY department_id;

-- ARRAY_AGG: Aggregate into array
SELECT
    department_id,
    ARRAY_AGG(employee_id) AS employee_ids,
    ARRAY_AGG(salary ORDER BY salary DESC) AS salaries_desc
FROM employees
GROUP BY department_id;
```
