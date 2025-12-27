# 17 Security & Permissions

### User and Role Management

```sql
-- Create user (role without login)
CREATE ROLE data_analyst;

-- Create user (role with login)
CREATE USER app_user WITH PASSWORD 'secure_password';

-- Create user with additional options
CREATE USER report_user WITH
    PASSWORD 'temp_password'
    VALID UNTIL '2026-12-31'
    CREATEDB;  -- Can create databases

-- Change password
ALTER USER app_user WITH PASSWORD 'new_password';

-- Grant login privilege
ALTER ROLE data_analyst WITH LOGIN;

-- Drop user
DROP USER app_user;

-- List users
SELECT * FROM pg_user;
\du  -- In psql
```

### GRANT Permissions

```sql
-- Grant SELECT on table
GRANT SELECT ON employees TO app_user;

-- Grant SELECT and UPDATE
GRANT SELECT, UPDATE ON employees TO app_user;

-- Grant all privileges
GRANT ALL PRIVILEGES ON employees TO app_user;

-- Grant on schema
GRANT USAGE ON SCHEMA public TO app_user;

-- Grant on all tables in schema
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_user;

-- Grant default privileges (for future objects)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO app_user;

-- Grant execution on functions
GRANT EXECUTE ON FUNCTION calculate_bonus(INT, NUMERIC) TO app_user;

-- Grant create privilege
GRANT CREATE ON DATABASE mydb TO app_user;
```

### REVOKE Permissions

```sql
-- Revoke specific privilege
REVOKE UPDATE ON employees FROM app_user;

-- Revoke all privileges
REVOKE ALL PRIVILEGES ON employees FROM app_user;

-- Revoke with cascade (revokes dependent grants)
REVOKE SELECT ON employees FROM app_user CASCADE;
```

### Row-Level Security (PostgreSQL specific)

```sql
-- Create table
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    salary NUMERIC,
    department_id INT
);

-- Create policy: employees can only see their own records
CREATE POLICY emp_select_policy ON employees
    FOR SELECT
    USING (employee_id = CURRENT_USER_ID);  -- Requires user tracking

-- Enable RLS on table
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

-- Create policy for management (can see all)
CREATE ROLE manager;
CREATE POLICY manager_policy ON employees
    FOR SELECT
    TO manager
    USING (TRUE);  -- Managers see all rows
```

### Auditing and Logging

```sql
-- Enable statement logging
SET log_statement = 'all';  -- or 'mod' for modifications only

-- Log slow queries
SET log_min_duration_statement = 1000;  -- Log queries > 1 second

-- Log connection attempts
SET log_connections = on;
SET log_disconnections = on;

-- View logs (typically in data directory/log/)
-- Or use pg_log_*() functions

-- Application-level auditing (via triggers)
CREATE TABLE audit_log (
    audit_id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    operation VARCHAR(10),
    user_name VARCHAR(50),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    old_values JSONB,
    new_values JSONB
);

CREATE OR REPLACE FUNCTION audit_function()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, user_name, old_values, new_values)
    VALUES (TG_TABLE_NAME, TG_OP, CURRENT_USER, row_to_json(OLD), row_to_json(NEW));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```
