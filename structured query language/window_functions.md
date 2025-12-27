# 07 Window Functions

Window functions perform calculations over a set of rows (a "window") without collapsing rows like GROUP BY.

### Window Function Syntax

```sql
SELECT
    column1,
    aggregate_function() OVER (
        [PARTITION BY column_list]
        [ORDER BY column_list [ASC|DESC]]
        [ROWS|RANGE frame_specification]
    ) AS window_result
FROM table_name;
```

### 1. Ranking Window Functions

```sql
-- ROW_NUMBER: Unique ranking (1, 2, 3, 4, ...)
SELECT
    employee_id,
    first_name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rank
FROM employees;

-- RANK: Ties get same rank, next rank skipped (1, 2, 2, 4, ...)
SELECT
    employee_id,
    first_name,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;

-- Top earner per department
SELECT * FROM (
    SELECT
        employee_id,
        first_name,
        salary,
        department_id,
        RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
    FROM employees
) ranked
WHERE dept_rank = 1;

-- NTILE: Distribute rows into N buckets
SELECT
    employee_id,
    salary,
    NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;
```

### 2. Aggregate Window Functions

```sql
-- Running SUM (cumulative sum)
SELECT
    order_date,
    order_amount,
    SUM(order_amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_total
FROM orders
ORDER BY order_date;

-- Moving average (last 3 rows)
SELECT
    date,
    close_price,
    AVG(close_price) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3day
FROM stock_prices;

-- Partition with aggregate
SELECT
    employee_id,
    first_name,
    department_id,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary,
    SUM(salary) OVER (PARTITION BY department_id) AS dept_total_salary
FROM employees;

-- Comparing to aggregate (useful pattern)
SELECT
    employee_id,
    first_name,
    salary,
    AVG(salary) OVER () AS company_avg,
    salary - AVG(salary) OVER () AS diff_from_avg
FROM employees;
```

### 3. Value Window Functions

```sql
-- LEAD/LAG: Access next/previous rows
SELECT
    date,
    close_price,
    LAG(close_price) OVER (ORDER BY date) AS prev_price,
    LEAD(close_price) OVER (ORDER BY date) AS next_price,
    close_price - LAG(close_price) OVER (ORDER BY date) AS price_change
FROM stock_prices;

-- FIRST_VALUE / LAST_VALUE
SELECT
    employee_id,
    hire_date,
    salary,
    FIRST_VALUE(salary) OVER (ORDER BY hire_date) AS first_salary,
    LAST_VALUE(salary) OVER (
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_salary
FROM employees;

-- NTH_VALUE: Get Nth row value
SELECT
    employee_id,
    salary,
    NTH_VALUE(salary, 2) OVER (ORDER BY salary DESC) AS second_highest
FROM employees;
```

### Window Frame Specifications

```sql
-- ROWS BETWEEN frame
SELECT
    date,
    value,
    AVG(value) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING  -- 5-row window
    ) AS avg_5row
FROM data;

-- RANGE (with expressions)
SELECT
    date,
    value,
    SUM(value) OVER (
        ORDER BY date
        RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
    ) AS sum_7day
FROM data;

-- Frame specifications
ROWS/RANGE BETWEEN:
    UNBOUNDED PRECEDING      -- From first row
    N PRECEDING              -- N rows before
    CURRENT ROW              -- Current row
    N FOLLOWING              -- N rows after
    UNBOUNDED FOLLOWING      -- To last row
```

### Interview Question: Window Functions vs GROUP BY

```sql
-- GROUP BY (collapses rows)
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;
-- Result: 10 rows (one per department)

-- Window function (keeps all rows)
SELECT
    employee_id,
    first_name,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg
FROM employees;
-- Result: 100 rows (all employees with their dept avg)
```
