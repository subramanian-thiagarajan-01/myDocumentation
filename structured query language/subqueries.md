# 06 Subqueries & Correlated Subqueries

### Subqueries (Non-Correlated)

Independent queries executed once, returning result to outer query.

```sql
-- Subquery in WHERE clause (filtering)
SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Subquery with IN
SELECT * FROM employees
WHERE department_id IN (
    SELECT department_id FROM departments
    WHERE location = 'New York'
);

-- Subquery with NOT IN (watch out for NULL!)
SELECT * FROM employees
WHERE department_id NOT IN (
    SELECT department_id FROM departments
    WHERE status = 'inactive'
);
-- Better approach:
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM departments d
    WHERE d.department_id = e.department_id
    AND d.status = 'inactive'
);

-- Subquery returning multiple columns (used with compound condition)
SELECT * FROM employees
WHERE (department_id, salary) IN (
    SELECT department_id, MAX(salary)
    FROM employees
    GROUP BY department_id
);

-- Subquery in FROM clause (derived table)
SELECT * FROM (
    SELECT employee_id, first_name, salary * 12 AS annual_salary
    FROM employees
    WHERE status = 'active'
) AS active_employees
WHERE annual_salary > 600000;

-- Subquery in SELECT clause (scalar subquery)
SELECT
    employee_id,
    first_name,
    salary,
    (SELECT AVG(salary) FROM employees) AS company_avg
FROM employees;

-- EXISTS vs IN (EXISTS is often faster)
SELECT * FROM employees e
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = e.employee_id
);
```

### Correlated Subqueries

Subqueries that reference columns from outer query; executed once per outer row.

```sql
-- Finding employees earning more than their department average
SELECT * FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id = e.department_id
);

-- Finding top earner per department
SELECT * FROM employees e1
WHERE salary = (
    SELECT MAX(salary)
    FROM employees e2
    WHERE e1.department_id = e2.department_id
);

-- Checking if employee has any orders (EXISTS pattern)
SELECT e.* FROM employees e
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = e.employee_id
);

-- NOT EXISTS (more efficient than NOT IN with NULL handling)
SELECT e.* FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM inactive_employees i
    WHERE i.employee_id = e.employee_id
);

-- Correlated UPDATE
UPDATE employees e
SET salary = (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id = e.department_id
)
WHERE department_id = 2;

-- Correlated DELETE
DELETE FROM employees e
WHERE salary < (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id = e.department_id
) AND department_id = 3;
```

### Subquery Performance Considerations

```sql
-- INEFFICIENT: Correlated subquery (many executions)
SELECT e.* FROM employees e
WHERE salary > (
    SELECT AVG(salary) FROM employees
    WHERE department_id = e.department_id
);

-- EFFICIENT: CTE or JOIN
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT e.* FROM employees e
JOIN dept_avg da ON e.department_id = da.department_id
WHERE e.salary > da.avg_salary;
```
