# 10 Transactions & ACID Properties

### ACID Properties

**Atomicity:** Transaction is "all or nothing" - either all changes commit or all rollback.
**Consistency:** Database moves from one valid state to another.
**Isolation:** Concurrent transactions don't interfere with each other.
**Durability:** Committed data survives system failures.

### Transaction Control

```sql
-- BEGIN transaction
BEGIN;  -- or START TRANSACTION;

-- Execute statements
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

-- Commit (make permanent)
COMMIT;

-- Or rollback (undo changes)
ROLLBACK;

-- Transaction in a single statement (auto-commit)
UPDATE employees SET salary = 60000 WHERE employee_id = 5;
```

### Savepoints

Nested rollback points within a transaction.

```sql
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    SAVEPOINT sp1;

    UPDATE accounts SET balance = balance + 100 WHERE id = 2;

    -- Oops, realize mistake
    ROLLBACK TO SAVEPOINT sp1;  -- Undo only the second update

    -- Now correct update
    UPDATE accounts SET balance = balance + 100 WHERE id = 3;

COMMIT;  -- First and third updates committed
```

### Isolation Levels

PostgreSQL implements MVCC (Multi-Version Concurrency Control) with 4 isolation levels:

#### 1. Read Uncommitted (Not explicitly supported in PostgreSQL)

Allows reading uncommitted changes from other transactions (dirty reads). PostgreSQL treats this as Read Committed.

#### 2. Read Committed (Default)

Each statement sees committed data as of statement start. Prevents dirty reads but allows non-repeatable reads.

```sql
-- Connection 1
BEGIN;
SELECT salary FROM employees WHERE employee_id = 1;  -- 50000

-- Connection 2 (meanwhile)
BEGIN;
UPDATE employees SET salary = 60000 WHERE employee_id = 1;
COMMIT;

-- Connection 1
SELECT salary FROM employees WHERE employee_id = 1;  -- 60000 (different!)
COMMIT;
```

#### 3. Repeatable Read

Transaction sees consistent snapshot from transaction start. Prevents non-repeatable reads but allows phantom reads (new rows appearing).

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM employees WHERE department_id = 1;  -- 10

-- Meanwhile, another transaction inserts employee
SELECT COUNT(*) FROM employees WHERE department_id = 1;  -- Still 10
-- Later query with more specific filter finds the new row
SELECT * FROM employees
WHERE department_id = 1 AND hire_date > '2025-01-01';  -- Sees new row

COMMIT;
```

#### 4. Serializable

Highest isolation level. Transactions behave as if executed sequentially (no concurrency anomalies).

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Most restrictive, potential for serialization conflicts
```

### Setting Isolation Level

```sql
-- Set for current transaction
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... queries ...
COMMIT;

-- Set default for session
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Set connection-wide default
SET DEFAULT_TRANSACTION_ISOLATION = REPEATABLE READ;
```

### Deadlock Handling

When two transactions wait for each other's locks.

```sql
-- Connection 1
BEGIN;
UPDATE employees SET salary = 60000 WHERE employee_id = 1;
SAVEPOINT sp1;

-- Connection 2 (meanwhile)
BEGIN;
UPDATE employees SET salary = 65000 WHERE employee_id = 2;
UPDATE employees SET salary = 70000 WHERE employee_id = 1;  -- WAIT...
-- -- Connection 1
UPDATE employees SET salary = 70000 WHERE employee_id = 2;  -- WAIT...
-- DEADLOCK! PostgreSQL kills one transaction with error:
-- ERROR: deadlock detected

-- Prevention strategies:
-- 1. Always access rows in same order
-- 2. Keep transactions short
-- 3. Use appropriate isolation level
-- 4. Implement retry logic in application
```
