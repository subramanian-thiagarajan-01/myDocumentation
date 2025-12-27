# 12 Triggers

Triggers execute automatically in response to database events (INSERT, UPDATE, DELETE).

### Trigger Syntax

```sql
CREATE OR REPLACE FUNCTION trigger_function_name()
RETURNS TRIGGER AS $$
BEGIN
    -- Trigger logic
    RETURN NEW;  -- For BEFORE/AFTER INSERT/UPDATE
    -- RETURN NULL;  -- For AFTER DELETE
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_name
{BEFORE|AFTER|INSTEAD OF} {INSERT|UPDATE|DELETE}
ON table_name
FOR EACH ROW
[WHEN (condition)]
EXECUTE FUNCTION trigger_function_name();
```

### Practical Trigger Examples

```sql
-- 1. Audit trigger (log all changes)
CREATE TABLE audit_log (
    log_id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    operation VARCHAR(10),
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data)
    VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employees_audit
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
EXECUTE FUNCTION audit_trigger();

-- 2. Timestamp update trigger (automatically update modified_at)
CREATE OR REPLACE FUNCTION update_modified_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at := CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employees_update_timestamp
BEFORE UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION update_modified_timestamp();

-- 3. Validation trigger
CREATE OR REPLACE FUNCTION validate_salary()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.salary <= 0 THEN
        RAISE EXCEPTION 'Salary must be positive';
    END IF;

    IF NEW.salary > 999999.99 THEN
        RAISE WARNING 'Salary exceeds normal range';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER salary_validation
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION validate_salary();

-- 4. Enforcing business rules
CREATE OR REPLACE FUNCTION check_employee_department()
RETURNS TRIGGER AS $$
DECLARE
    dept_exists INT;
BEGIN
    SELECT COUNT(*) INTO dept_exists
    FROM departments
    WHERE department_id = NEW.department_id AND is_active = TRUE;

    IF dept_exists = 0 THEN
        RAISE EXCEPTION 'Department does not exist or is inactive';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_valid_department
BEFORE INSERT OR UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION check_employee_department();

-- 5. Cascade business logic (maintaining denormalized data)
CREATE OR REPLACE FUNCTION update_department_total()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        UPDATE departments
        SET total_salary = total_salary - OLD.salary
        WHERE department_id = OLD.department_id;
    ELSIF TG_OP = 'INSERT' THEN
        UPDATE departments
        SET total_salary = total_salary + NEW.salary
        WHERE department_id = NEW.department_id;
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE departments
        SET total_salary = total_salary - OLD.salary + NEW.salary
        WHERE department_id = NEW.department_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER maintain_dept_salary_total
AFTER INSERT OR UPDATE OR DELETE ON employees
FOR EACH ROW
EXECUTE FUNCTION update_department_total();
```

### Managing Triggers

```sql
-- List triggers
SELECT trigger_name, event_manipulation, event_object_schema, event_object_table
FROM information_schema.triggers;

-- Disable trigger (temporarily)
ALTER TABLE employees DISABLE TRIGGER ALL;

-- Enable trigger
ALTER TABLE employees ENABLE TRIGGER ALL;

-- Drop trigger
DROP TRIGGER trigger_name ON table_name;

-- View trigger definition
SELECT pg_get_triggerdef(trigger_oid);
```
