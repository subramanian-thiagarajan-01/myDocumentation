# 14 Normalization & Database Design

### 1NF - First Normal Form

Ensures atomicity: all column values are atomic (indivisible).

```sql
-- VIOLATION OF 1NF (composite value in column)
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    phone_numbers VARCHAR(100)  -- "555-1234, 555-5678" - composite!
);

-- CORRECTED (1NF compliant)
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE phone_numbers (
    phone_id SERIAL PRIMARY KEY,
    employee_id INT REFERENCES employees(employee_id),
    phone_number VARCHAR(15)
);

-- Or using PostgreSQL array type (still 1NF if properly used)
CREATE TABLE employees_arrays (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    phone_numbers VARCHAR(15)[]
);
```

### 2NF - Second Normal Form

Eliminates partial dependencies: non-key attributes must depend on the entire primary key (not just part of it).

```sql
-- VIOLATION OF 2NF (student_name depends on student_id, not entire composite key)
CREATE TABLE course_enrollments_bad (
    student_id INT,
    course_id INT,
    student_name VARCHAR(100),  -- Depends only on student_id, not (student_id, course_id)
    grade CHAR(1),
    PRIMARY KEY (student_id, course_id)
);

-- CORRECTED (2NF compliant)
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100)
);

CREATE TABLE course_enrollments (
    student_id INT REFERENCES students(student_id),
    course_id INT REFERENCES courses(course_id),
    grade CHAR(1),
    PRIMARY KEY (student_id, course_id)
);
```

### 3NF - Third Normal Form

Eliminates transitive dependencies: non-key attributes must depend directly on the primary key, not through other non-key attributes.

```sql
-- VIOLATION OF 3NF (salary_grade depends on department through salary_range)
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    salary NUMERIC,
    salary_grade VARCHAR(20),  -- Derived from salary_range, not primary key
    salary_range VARCHAR(20)   -- Derived from salary
);

-- CORRECTED (3NF compliant)
CREATE TABLE salary_grades (
    grade_id SERIAL PRIMARY KEY,
    grade_name VARCHAR(20),
    min_salary NUMERIC,
    max_salary NUMERIC
);

CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    salary NUMERIC,
    salary_grade_id INT REFERENCES salary_grades(grade_id)
);
```

### BCNF - Boyce-Codd Normal Form

Stricter than 3NF: every determinant must be a candidate key.

```sql
-- VIOLATION OF BCNF
CREATE TABLE professor_courses_bad (
    professor_id INT,
    course_id INT,
    subject_id INT,
    PRIMARY KEY (professor_id, course_id),
    -- Constraint: subject_id is determined by course_id
    -- But professor_id also determines subject_id (one professor teaches one subject)
    -- So subject_id -> course_id is a problematic dependency
    UNIQUE (professor_id, subject_id)
);

-- CORRECTED (BCNF compliant)
CREATE TABLE professors (
    professor_id INT PRIMARY KEY,
    subject_id INT
);

CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    subject_id INT
);

CREATE TABLE professor_courses (
    professor_id INT REFERENCES professors(professor_id),
    course_id INT REFERENCES courses(course_id),
    PRIMARY KEY (professor_id, course_id)
);
```

### Denormalization Trade-offs

```sql
-- Normalized (better for updates, smaller storage)
CREATE TABLE orders_normalized (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    order_date DATE,
    total_amount NUMERIC  -- Would need to calculate via SUM query
);

-- Denormalized (better for reads, redundant data)
CREATE TABLE orders_denormalized (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id),
    customer_name VARCHAR(100),  -- Denormalized (redundant)
    order_date DATE,
    total_amount NUMERIC,        -- Denormalized (calculated, not stored)
    num_items INT                -- Denormalized
);

-- When to denormalize:
-- 1. Complex aggregations needed frequently
-- 2. Read-heavy workload with minimal writes
-- 3. Performance-critical dashboards
-- 4. Maintain triggers to keep denormalized data consistent
```
