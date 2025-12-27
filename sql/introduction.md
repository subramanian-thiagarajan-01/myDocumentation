# 01 Introduction & Core Concepts

### What is SQL and PostgreSQL?

**SQL (Structured Query Language)** is the standard language for managing relational databases. **PostgreSQL** is an advanced, open-source relational database management system (RDBMS) that implements SQL standards with extensive features like JSON support, full-text search, and advanced indexing.

**Key characteristics of PostgreSQL:**

- ACID compliant
- MVCC (Multi-Version Concurrency Control) for concurrency
- Extensible type system
- Advanced indexing (B-tree, Hash, GiST, GIN, BRIN, SP-GiST)
- PL/pgSQL procedural language
- Foreign Data Wrappers for external data access

### Relational Database Model

The relational model organizes data into **tables (relations)** with **rows (tuples)** and **columns (attributes)**. Each table has:

- **Primary Key**: Uniquely identifies each row
- **Foreign Key**: References a primary key in another table
- **Constraints**: Rules ensuring data integrity

### DBMS (Database Management System)

A DBMS is software that manages database creation, modification, and access. PostgreSQL provides:

- **Data storage & retrieval**
- **Concurrency control** via MVCC
- **Transaction management** with ACID properties
- **Query optimization** via query planner
- **Security mechanisms** (roles, permissions)

---

## SQL Language Categories

### 1. DQL (Data Query Language)

Retrieves data from the database without modification.

**Commands:** `SELECT`

```sql
-- Example: Basic SELECT query
SELECT employee_id, first_name, salary
FROM employees
WHERE salary > 50000;
```

### 2. DML (Data Manipulation Language)

Modifies data within existing tables.

**Commands:** `INSERT`, `UPDATE`, `DELETE`

```sql
-- INSERT: Add new records
INSERT INTO employees (employee_id, first_name, salary)
VALUES (101, 'John Doe', 55000);

-- UPDATE: Modify existing records
UPDATE employees
SET salary = 60000
WHERE employee_id = 101;

-- DELETE: Remove records
DELETE FROM employees
WHERE employee_id = 101;
```

### 3. DDL (Data Definition Language)

Defines the structure of the database and its objects.

**Commands:** `CREATE`, `ALTER`, `DROP`, `TRUNCATE`

```sql
-- CREATE: Define new table
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    salary NUMERIC(10, 2)
);

-- ALTER: Modify table structure
ALTER TABLE employees ADD COLUMN department_id INT;

-- DROP: Delete table
DROP TABLE employees;

-- TRUNCATE: Remove all rows (faster than DELETE)
TRUNCATE TABLE employees;
```

### 4. DCL (Data Control Language)

Manages database access and permissions.

**Commands:** `GRANT`, `REVOKE`

```sql
-- GRANT: Give permissions
GRANT SELECT, INSERT ON employees TO user_name;
GRANT ALL PRIVILEGES ON DATABASE mydb TO user_name;

-- REVOKE: Remove permissions
REVOKE INSERT ON employees FROM user_name;
```

### 5. TCL (Transaction Control Language)

Manages transactions and ensures data consistency.

**Commands:** `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`

```sql
-- Transaction management
BEGIN;
UPDATE employees SET salary = 60000 WHERE employee_id = 101;
SAVEPOINT sp1;
UPDATE employees SET salary = 65000 WHERE employee_id = 102;
ROLLBACK TO SAVEPOINT sp1; -- Rollback to sp1, keeping first update
COMMIT; -- Finalize the transaction
```

---

## Data Types in PostgreSQL

### Numeric Types

| Type                              | Range                                                   | Usage                                  |
| --------------------------------- | ------------------------------------------------------- | -------------------------------------- |
| `SMALLINT`                        | -32,768 to 32,767                                       | Small integers                         |
| `INTEGER` / `INT`                 | -2,147,483,648 to 2,147,483,647                         | Standard integers                      |
| `BIGINT`                          | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 | Large integers                         |
| `DECIMAL(p, s)` / `NUMERIC(p, s)` | Arbitrary precision                                     | Precise decimal calculations (finance) |
| `REAL`                            | Single-precision floating-point                         | Approximate decimals                   |
| `DOUBLE PRECISION`                | Double-precision floating-point                         | Approximate decimals (more precision)  |

```sql
-- Examples
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    price NUMERIC(10, 2),        -- Max 10 digits, 2 decimal places
    stock_quantity INTEGER,
    rating REAL
);
```

### Character Types

| Type                 | Characteristics                            |
| -------------------- | ------------------------------------------ |
| `CHAR(n)`            | Fixed-length string (padded with spaces)   |
| `VARCHAR(n)`         | Variable-length string with max length     |
| `VARCHAR` (no limit) | Variable-length string, no limit           |
| `TEXT`               | Variable-length string, no practical limit |

```sql
-- Best practices
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,    -- Limited, indexed typically
    bio TEXT,                         -- Unlimited text
    phone_number CHAR(10)            -- Fixed 10 digits
);
```

### Date & Time Types

| Type                                        | Range                       | Precision             |
| ------------------------------------------- | --------------------------- | --------------------- |
| `DATE`                                      | 1900-01-01 to 9999-12-31    | Day                   |
| `TIME`                                      | 00:00:00 to 23:59:59.999999 | Microsecond           |
| `TIMESTAMP` / `TIMESTAMP WITHOUT TIME ZONE` | Date + Time                 | Microsecond (no TZ)   |
| `TIMESTAMPTZ` / `TIMESTAMP WITH TIME ZONE`  | Date + Time + TZ            | Microsecond (with TZ) |
| `INTERVAL`                                  | Time duration               | Microsecond           |

```sql
CREATE TABLE events (
    event_id SERIAL PRIMARY KEY,
    event_date DATE NOT NULL,
    start_time TIME NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    duration INTERVAL
);

-- Examples
INSERT INTO events (event_date, start_time, duration)
VALUES ('2025-11-09', '14:30:00', '2 hours 30 minutes');
```

### Boolean Type

```sql
CREATE TABLE features (
    feature_id SERIAL PRIMARY KEY,
    is_active BOOLEAN DEFAULT TRUE,
    is_premium BOOLEAN
);

-- TRUE values: true, 't', 'yes', 'y', 'on', '1'
-- FALSE values: false, 'f', 'no', 'n', 'off', '0'
```

### JSON Types

```sql
-- JSON vs JSONB
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    metadata JSON,              -- Stored as text, slower
    config JSONB                -- Stored in binary, faster
);

-- Examples
INSERT INTO products (metadata, config)
VALUES (
    '{"color": "red", "size": "large"}',
    '{"enabled": true, "version": 2}'
);
```

### Binary Type

```sql
-- BYTEA: Binary data
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    file_content BYTEA
);
```

### UUID Type

```sql
-- Universally Unique Identifier
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(100) UNIQUE
);
```

### Array Types

```sql
-- PostgreSQL supports arrays
CREATE TABLE user_preferences (
    user_id SERIAL PRIMARY KEY,
    tags TEXT[],               -- Array of text
    scores INTEGER[]           -- Array of integers
);

-- Examples
INSERT INTO user_preferences (tags, scores)
VALUES ('{sql, postgres, database}', '{95, 87, 92}');

SELECT * FROM user_preferences WHERE 'sql' = ANY(tags);
```
