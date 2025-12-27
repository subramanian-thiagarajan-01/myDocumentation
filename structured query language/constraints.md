# 09 Data Definition & Constraints

### CREATE TABLE

Defines table structure and constraints.

```sql
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone_number VARCHAR(15),
    hire_date DATE NOT NULL,
    salary NUMERIC(10, 2) CHECK (salary > 0),
    department_id INT NOT NULL REFERENCES departments(department_id),
    manager_id INT REFERENCES employees(employee_id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);

-- CREATE TABLE AS SELECT (CTAS)
CREATE TABLE high_earners AS
SELECT * FROM employees WHERE salary > 100000;

-- CREATE TABLE IF NOT EXISTS
CREATE TABLE IF NOT EXISTS backup_employees AS
SELECT * FROM employees;

-- CREATE TEMPORARY TABLE (session-specific)
CREATE TEMPORARY TABLE temp_calculations AS
SELECT department_id, AVG(salary) AS avg_sal
FROM employees
GROUP BY department_id;
```

### Constraints

#### Primary Key Constraint

```sql
-- Column-level primary key
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(100)
);

-- Table-level primary key
CREATE TABLE users (
    user_id INT,
    email VARCHAR(100),
    PRIMARY KEY (user_id)
);

-- Composite primary key
CREATE TABLE order_items (
    order_id INT,
    item_id INT,
    quantity INT,
    PRIMARY KEY (order_id, item_id)
);
```

#### Foreign Key Constraint

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    department_id INT REFERENCES departments(department_id),
    -- or table-level:
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
        ON DELETE CASCADE        -- Delete employee if dept deleted
        ON UPDATE RESTRICT       -- Don't allow dept_id updates
);

-- Referential integrity options
ON DELETE CASCADE;         -- Delete child rows when parent deleted
ON DELETE SET NULL;        -- Set foreign key to NULL when parent deleted
ON DELETE RESTRICT;        -- Prevent deletion if children exist
ON UPDATE CASCADE;         -- Update child keys when parent key updated
```

#### Unique Constraint

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE,  -- Column-level
    username VARCHAR(50) UNIQUE NOT NULL
);

-- Composite unique constraint
CREATE TABLE subscriptions (
    subscription_id INT PRIMARY KEY,
    user_id INT,
    service_id INT,
    UNIQUE (user_id, service_id)  -- Can't subscribe to same service twice
);

-- Allowing multiple NULLs with partial unique index
CREATE UNIQUE INDEX idx_email_not_deleted
ON users(email)
WHERE is_deleted = FALSE;  -- Multiple NULL values allowed
```

#### Check Constraint

```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    price NUMERIC(10, 2) CHECK (price > 0),
    quantity INT CHECK (quantity >= 0),
    email VARCHAR(100) CHECK (email LIKE '%@%.%')
);

-- Composite check constraint
CREATE TABLE salary_history (
    employee_id INT,
    salary NUMERIC(10, 2),
    start_date DATE,
    end_date DATE,
    CHECK (end_date > start_date)
);
```

#### Default Values

```sql
CREATE TABLE audit_log (
    log_id SERIAL PRIMARY KEY,
    action VARCHAR(50) NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_processed BOOLEAN DEFAULT FALSE,
    ip_address INET DEFAULT INET_CLIENT_ADDR()
);

-- Computed default (PostgreSQL specific)
CREATE TABLE events (
    event_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### ALTER TABLE

Modifies existing table structure.

```sql
-- Add column
ALTER TABLE employees ADD COLUMN bonus NUMERIC(10, 2);

-- Add column with default
ALTER TABLE employees ADD COLUMN last_review_date DATE DEFAULT CURRENT_DATE;

-- Add constraint
ALTER TABLE employees ADD CONSTRAINT fk_dept
FOREIGN KEY (department_id) REFERENCES departments(department_id);

-- Modify column (change type, constraints)
ALTER TABLE employees
ALTER COLUMN salary TYPE NUMERIC(12, 2);

-- Rename column
ALTER TABLE employees RENAME COLUMN phone_number TO phone;

-- Rename table
ALTER TABLE employees RENAME TO staff;

-- Drop column (careful!)
ALTER TABLE employees DROP COLUMN bonus;

-- Drop constraint
ALTER TABLE employees DROP CONSTRAINT fk_dept;

-- Set column default
ALTER TABLE employees ALTER COLUMN is_active SET DEFAULT TRUE;

-- Drop column default
ALTER TABLE employees ALTER COLUMN hire_date DROP DEFAULT;

-- Change column nullability
ALTER TABLE employees ALTER COLUMN email SET NOT NULL;
ALTER TABLE employees ALTER COLUMN phone DROP NOT NULL;
```

### DROP TABLE

```sql
-- Drop single table
DROP TABLE employees;

-- Drop multiple tables
DROP TABLE employees, departments, jobs;

-- Drop only if exists (no error if doesn't exist)
DROP TABLE IF EXISTS temporary_table;

-- Drop with CASCADE (delete dependent objects)
DROP TABLE departments CASCADE;  -- Drops dependent views, indexes, etc.

-- Drop with RESTRICT (default, fails if dependencies exist)
DROP TABLE departments RESTRICT;
```

### TRUNCATE vs DELETE

```sql
-- TRUNCATE: Fast removal of all rows, resets identity
TRUNCATE TABLE employees;                    -- All rows deleted
TRUNCATE TABLE employees RESTART IDENTITY;   -- Reset auto-increment
TRUNCATE TABLE employees CASCADE;            -- Also truncate dependent tables

-- DELETE: Slower, supports WHERE clause, logs each row
DELETE FROM employees;                       -- All rows
DELETE FROM employees WHERE department_id = 5;
DELETE FROM employees WHERE hire_date < '2020-01-01';

-- Key differences
-- TRUNCATE: DDL command, faster, no WHERE clause, resets identity
-- DELETE: DML command, slower, WHERE support, triggers fire, can ROLLBACK
```
