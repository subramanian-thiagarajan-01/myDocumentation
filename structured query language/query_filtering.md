# 03 Query Filtering & Sorting

### Logical Operators (AND, OR, NOT)

```sql
-- AND (all conditions must be true)
SELECT * FROM employees
WHERE salary > 50000 AND department_id = 2 AND hire_date > '2020-01-01';

-- OR (at least one condition is true)
SELECT * FROM employees
WHERE department_id = 1 OR department_id = 2 OR department_id = 3;

-- NOT
SELECT * FROM employees WHERE NOT status = 'inactive';

-- Combining AND/OR (pay attention to operator precedence)
SELECT * FROM employees
WHERE (department_id = 1 OR department_id = 2)
  AND salary > 50000;
```

### IN, BETWEEN, LIKE Operators

```sql
-- IN operator
SELECT * FROM employees WHERE employee_id IN (101, 102, 103);

-- NOT IN (be careful with NULL!)
-- If subquery returns NULL, result is always UNKNOWN
SELECT * FROM employees
WHERE employee_id NOT IN (
    SELECT employee_id FROM inactive_employees
);
-- Better approach with NOT EXISTS to avoid NULL issues:
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM inactive_employees i
    WHERE i.employee_id = e.employee_id
);

-- BETWEEN (inclusive on both ends)
SELECT * FROM employees WHERE salary BETWEEN 40000 AND 80000;

-- LIKE with wildcards
SELECT * FROM employees
WHERE email LIKE '%@company.com';

-- ILIKE (case-insensitive, PostgreSQL specific)
SELECT * FROM employees
WHERE first_name ILIKE 'john%';
```

### Advanced Filtering

```sql
-- CASE statements for conditional filtering
SELECT employee_id, first_name,
    CASE
        WHEN salary < 40000 THEN 'Entry Level'
        WHEN salary BETWEEN 40000 AND 70000 THEN 'Mid Level'
        WHEN salary > 70000 THEN 'Senior Level'
        ELSE 'Unknown'
    END AS salary_level
FROM employees;

-- Filtering with window functions
SELECT * FROM (
    SELECT
        employee_id,
        first_name,
        salary,
        RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank
    FROM employees
) ranked_employees
WHERE rank <= 3;  -- Top 3 per department
```
