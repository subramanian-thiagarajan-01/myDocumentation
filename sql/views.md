# 13 Views

Views are virtual tables created from queries; they don't store data.

### Create Views

```sql
-- Basic view
CREATE VIEW employee_summary AS
SELECT
    employee_id,
    first_name || ' ' || last_name AS full_name,
    salary,
    department_id
FROM employees
WHERE is_active = TRUE;

-- Use view
SELECT * FROM employee_summary WHERE salary > 50000;

-- View with joins
CREATE VIEW emp_dept_view AS
SELECT
    e.employee_id,
    e.first_name,
    e.salary,
    d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;

-- Updatable view (with simple structure)
CREATE VIEW active_employees AS
SELECT employee_id, first_name, salary
FROM employees
WHERE is_active = TRUE;

-- Update through view
UPDATE active_employees SET salary = 60000 WHERE employee_id = 1;  -- Updates source table

-- WITH CHECK OPTION (ensures updated rows still match view criteria)
CREATE VIEW sales_employees AS
SELECT employee_id, first_name, department_id
FROM employees
WHERE department_id = 3
WITH CHECK OPTION;  -- Can't update to move employee to different dept

-- View with aggregation (not updatable)
CREATE VIEW department_summary AS
SELECT
    department_id,
    COUNT(*) AS num_employees,
    AVG(salary) AS avg_salary,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department_id;

-- Materialized view (stores results for performance)
CREATE MATERIALIZED VIEW mv_sales_by_month AS
SELECT
    DATE_TRUNC('month', order_date)::DATE AS month,
    SUM(order_amount) AS total_sales,
    COUNT(*) AS num_orders
FROM orders
GROUP BY DATE_TRUNC('month', order_date);

-- Refresh materialized view
REFRESH MATERIALIZED VIEW mv_sales_by_month;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_by_month;  -- Non-blocking
```

### Modify & Drop Views

```sql
-- Modify view (replace)
CREATE OR REPLACE VIEW employee_summary AS
SELECT
    employee_id,
    first_name || ' ' || last_name AS full_name,
    salary,
    department_id,
    hire_date  -- New column
FROM employees
WHERE is_active = TRUE;

-- ALTER VIEW (modify properties)
ALTER VIEW employee_summary RENAME TO emp_summary;

-- Drop view
DROP VIEW employee_summary;

-- Drop with CASCADE (drops dependent objects)
DROP VIEW employee_summary CASCADE;

-- Drop view if exists
DROP VIEW IF EXISTS employee_summary;
```
