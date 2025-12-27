# 08 Common Table Expressions (CTEs)

CTEs (WITH clauses) create named temporary result sets for better readability and recursion.

### Non-Recursive CTEs

```sql
-- Single CTE
WITH high_earners AS (
    SELECT employee_id, first_name, salary
    FROM employees
    WHERE salary > 80000
)
SELECT * FROM high_earners;

-- Multiple CTEs
WITH
dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
),
high_earners AS (
    SELECT e.*
    FROM employees e
    JOIN dept_avg da ON e.department_id = da.department_id
    WHERE e.salary > da.avg_salary * 1.2
)
SELECT * FROM high_earners;

-- CTE reused multiple times
WITH salary_stats AS (
    SELECT
        AVG(salary) AS avg_sal,
        MIN(salary) AS min_sal,
        MAX(salary) AS max_sal,
        STDDEV(salary) AS std_sal
    FROM employees
)
SELECT
    first_name,
    salary,
    salary - (SELECT avg_sal FROM salary_stats) AS diff_from_avg
FROM employees
WHERE salary > (SELECT max_sal FROM salary_stats) - 5000;
```

### Recursive CTEs

For hierarchical data (organization structure, bill of materials, etc.).

```sql
-- Basic recursive CTE structure
WITH RECURSIVE cte_name AS (
    -- Anchor member (base case)
    SELECT base_columns FROM table WHERE condition
    UNION ALL
    -- Recursive member
    SELECT recursive_columns FROM table
    WHERE join_condition_to_cte
)
SELECT * FROM cte_name;

-- Finding all subordinates of an employee
WITH RECURSIVE employee_hierarchy AS (
    -- Anchor: select manager (top of hierarchy)
    SELECT employee_id, first_name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: select subordinates
    SELECT e.employee_id, e.first_name, e.manager_id, eh.level + 1
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy;

-- Finding reporting chain (managers up the tree)
WITH RECURSIVE manager_chain AS (
    -- Anchor: select specific employee
    SELECT employee_id, first_name, manager_id, 0 AS level
    FROM employees
    WHERE employee_id = 5

    UNION ALL

    -- Recursive: join to get managers
    SELECT e.employee_id, e.first_name, e.manager_id, mc.level + 1
    FROM employees e
    INNER JOIN manager_chain mc ON e.employee_id = mc.manager_id
)
SELECT * FROM manager_chain;

-- Detecting cycles (important for preventing infinite recursion)
WITH RECURSIVE search_graph AS (
    SELECT id, link, data, 0 AS depth, false AS is_cycle, ARRAY[id] AS path
    FROM graph
    WHERE id = 1

    UNION ALL

    SELECT g.id, g.link, g.data, sg.depth + 1,
           g.id = ANY(path) AS is_cycle,  -- Cycle detection
           path || g.id
    FROM graph g
    JOIN search_graph sg ON g.id = sg.link
    WHERE NOT is_cycle AND sg.depth < 100  -- Depth limit safety
)
SELECT * FROM search_graph;

-- Generating sequences
WITH RECURSIVE date_series AS (
    SELECT '2025-01-01'::DATE AS date
    UNION ALL
    SELECT date + INTERVAL '1 day'
    FROM date_series
    WHERE date < '2025-12-31'
)
SELECT * FROM date_series;
```
