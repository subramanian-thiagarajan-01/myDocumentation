# 11 Stored Procedures & User-Defined Functions

### User-Defined Functions (UDFs)

Functions return a value and can be used in queries.

```sql
-- CREATE FUNCTION basic syntax
CREATE OR REPLACE FUNCTION function_name(param_name param_type, ...)
RETURNS return_type AS $$
DECLARE
    -- Variable declarations
    var_name TYPE;
BEGIN
    -- Function logic
    RETURN result;
END;
$$ LANGUAGE plpgsql;

-- Simple scalar function
CREATE OR REPLACE FUNCTION get_employee_salary(emp_id INT)
RETURNS NUMERIC AS $$
DECLARE
    emp_salary NUMERIC;
BEGIN
    SELECT salary INTO emp_salary
    FROM employees
    WHERE employee_id = emp_id;

    RETURN COALESCE(emp_salary, 0);
END;
$$ LANGUAGE plpgsql;

-- Use in query
SELECT first_name, get_employee_salary(employee_id)
FROM employees;

-- Function with multiple parameters
CREATE OR REPLACE FUNCTION calculate_bonus(
    emp_id INT,
    bonus_rate NUMERIC DEFAULT 0.1
)
RETURNS NUMERIC AS $$
DECLARE
    emp_salary NUMERIC;
BEGIN
    SELECT salary * bonus_rate INTO emp_salary
    FROM employees
    WHERE employee_id = emp_id;

    RETURN COALESCE(emp_salary, 0);
END;
$$ LANGUAGE plpgsql;

-- Function returning table
CREATE OR REPLACE FUNCTION get_dept_employees(dept_id INT)
RETURNS TABLE (emp_id INT, emp_name VARCHAR, emp_salary NUMERIC) AS $$
BEGIN
    RETURN QUERY
    SELECT employee_id, first_name, salary
    FROM employees
    WHERE department_id = dept_id;
END;
$$ LANGUAGE plpgsql;

-- Use returning function
SELECT * FROM get_dept_employees(2);
```

### Stored Procedures

Procedures execute operations without returning values (though they can have OUT parameters).

```sql
-- Basic stored procedure
CREATE OR REPLACE PROCEDURE transfer_funds(
    from_account INT,
    to_account INT,
    amount NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
    COMMIT;
EXCEPTION WHEN OTHERS THEN
    ROLLBACK;
    RAISE EXCEPTION 'Transfer failed: %', SQLERRM;
END;
$$;

-- Call procedure
CALL transfer_funds(1, 2, 100);

-- Procedure with OUT parameters
CREATE OR REPLACE PROCEDURE get_employee_info(
    emp_id INT,
    OUT emp_name VARCHAR,
    OUT emp_salary NUMERIC,
    OUT dept_name VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT e.first_name, e.salary, d.department_name
    INTO emp_name, emp_salary, dept_name
    FROM employees e
    LEFT JOIN departments d ON e.department_id = d.department_id
    WHERE e.employee_id = emp_id;
END;
$$;

-- Call and retrieve OUT parameters
CALL get_employee_info(5, emp_name => ?, emp_salary => ?, dept_name => ?);

-- Procedure with INOUT parameters
CREATE OR REPLACE PROCEDURE process_employees(
    INOUT emp_salary NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    emp_salary := emp_salary * 1.1;  -- Increase by 10%
END;
$$;

-- Procedure with transaction control
CREATE OR REPLACE PROCEDURE bulk_update_salaries(
    dept_id INT,
    increment NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    updated_count INT;
BEGIN
    UPDATE employees
    SET salary = salary + increment
    WHERE department_id = dept_id;

    GET DIAGNOSTICS updated_count = ROW_COUNT;
    RAISE NOTICE 'Updated % employees', updated_count;

    COMMIT;
EXCEPTION WHEN OTHERS THEN
    ROLLBACK;
    RAISE EXCEPTION 'Update failed: %', SQLERRM;
END;
$$;
```

### Control Structures in PL/pgSQL

```sql
-- IF-THEN-ELSE
CREATE OR REPLACE FUNCTION classify_salary(salary NUMERIC)
RETURNS VARCHAR AS $$
BEGIN
    IF salary < 30000 THEN
        RETURN 'Entry Level';
    ELSIF salary < 60000 THEN
        RETURN 'Mid Level';
    ELSIF salary < 100000 THEN
        RETURN 'Senior Level';
    ELSE
        RETURN 'Executive';
    END IF;
END;
$$ LANGUAGE plpgsql;

-- CASE statement
CREATE OR REPLACE FUNCTION get_rating(score INT)
RETURNS VARCHAR AS $$
BEGIN
    CASE
        WHEN score >= 90 THEN RETURN 'A';
        WHEN score >= 80 THEN RETURN 'B';
        WHEN score >= 70 THEN RETURN 'C';
        ELSE RETURN 'F';
    END CASE;
END;
$$ LANGUAGE plpgsql;

-- LOOP
CREATE OR REPLACE FUNCTION generate_series_demo()
RETURNS TABLE (num INT) AS $$
DECLARE
    i INT := 1;
BEGIN
    LOOP
        RETURN NEXT i;
        i := i + 1;
        EXIT WHEN i > 10;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- WHILE loop
CREATE OR REPLACE FUNCTION calculate_factorial(n INT)
RETURNS INT AS $$
DECLARE
    result INT := 1;
    counter INT := 2;
BEGIN
    WHILE counter <= n LOOP
        result := result * counter;
        counter := counter + 1;
    END LOOP;
    RETURN result;
END;
$$ LANGUAGE plpgsql;

-- FOR loop
CREATE OR REPLACE PROCEDURE populate_test_data(row_count INT)
LANGUAGE plpgsql
AS $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..row_count LOOP
        INSERT INTO test_table (value) VALUES (i * 10);
    END LOOP;
    COMMIT;
END;
$$;
```

### Error Handling

```sql
CREATE OR REPLACE FUNCTION divide_numbers(numerator INT, denominator INT)
RETURNS FLOAT AS $$
BEGIN
    IF denominator = 0 THEN
        RAISE EXCEPTION 'Division by zero not allowed';
    END IF;

    RETURN numerator::FLOAT / denominator;
EXCEPTION
    WHEN division_by_zero THEN
        RAISE EXCEPTION 'Caught: %', SQLERRM;
        RETURN 0;
    WHEN OTHERS THEN
        RAISE NOTICE 'Error: %', SQLERRM;
        RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```
