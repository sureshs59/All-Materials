# MySQL Commands Complete Guide

> A comprehensive reference for MySQL commands across all categories — Database, Table, CRUD, Query, Joins, Index, User, Transaction, and Admin.

---

## Table of Contents

1. [Database Commands](#1-database-commands)
2. [Table Commands](#2-table-commands)
3. [CRUD Commands](#3-crud-commands)
4. [Query Commands](#4-query-commands)
5. [Joins](#5-joins)
6. [Index Commands](#6-index-commands)
7. [User Management](#7-user-management)
8. [Transactions](#8-transactions)
9. [Admin Commands](#9-admin-commands)

---

## 1. Database Commands

### CREATE DATABASE
Create a new database.

```sql
CREATE DATABASE bank_db;
CREATE DATABASE bank_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

> Always specify `utf8mb4` for full Unicode support including emojis.

---

### USE
Switch to a specific database.

```sql
USE bank_db;
```

> Run this before any table operations so MySQL knows which database to work in.

---

### SHOW DATABASES
List all available databases.

```sql
SHOW DATABASES;
SHOW DATABASES LIKE 'bank%';
```

---

### DROP DATABASE
Delete a database permanently.

```sql
DROP DATABASE bank_db;
DROP DATABASE IF EXISTS bank_db;
```

> Warning: This deletes everything. Always backup first. `IF EXISTS` prevents error if DB doesn't exist.

---

### SELECT DATABASE()
Show the currently active database.

```sql
SELECT DATABASE();
```

---

## 2. Table Commands

### CREATE TABLE
Create a new table with columns and constraints.

```sql
CREATE TABLE accounts (
  id         BIGINT AUTO_INCREMENT PRIMARY KEY,
  account_no VARCHAR(20)    NOT NULL UNIQUE,
  name       VARCHAR(100)   NOT NULL,
  balance    DECIMAL(15,2)  DEFAULT 0.00,
  status     ENUM('ACTIVE','INACTIVE','CLOSED') DEFAULT 'ACTIVE',
  created_at DATETIME       DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME       DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

> Use `DECIMAL` for money — never `FLOAT` or `DOUBLE`. `AUTO_INCREMENT` gives surrogate primary keys.

---

### ALTER TABLE
Modify an existing table structure.

```sql
ALTER TABLE accounts ADD COLUMN email VARCHAR(100);
ALTER TABLE accounts MODIFY COLUMN name VARCHAR(200) NOT NULL;
ALTER TABLE accounts DROP COLUMN email;
ALTER TABLE accounts RENAME COLUMN account_no TO acc_number;
ALTER TABLE accounts ADD CONSTRAINT fk_branch FOREIGN KEY (branch_id) REFERENCES branches(id);
```

---

### DROP TABLE / TRUNCATE
Delete a table or clear all its rows.

```sql
DROP TABLE accounts;
DROP TABLE IF EXISTS accounts;
TRUNCATE TABLE accounts;
```

> `DROP` removes the table entirely. `TRUNCATE` removes all rows but keeps the structure. `TRUNCATE` is faster than `DELETE` for clearing all rows.

---

### DESCRIBE / SHOW
View table structure and details.

```sql
DESCRIBE accounts;
SHOW COLUMNS FROM accounts;
SHOW TABLES;
SHOW CREATE TABLE accounts;
SHOW TABLE STATUS LIKE 'accounts';
```

> `SHOW CREATE TABLE` shows the exact SQL used to create the table — useful for replication.

---

### RENAME TABLE
Rename one or more tables.

```sql
RENAME TABLE accounts TO bank_accounts;
RENAME TABLE old_name TO new_name, another TO another_new;
```

---

## 3. CRUD Commands

### INSERT
Add new rows to a table.

```sql
-- Single row
INSERT INTO accounts (account_no, name, balance)
VALUES ('ACC001', 'John Doe', 5000.00);

-- Multiple rows (batch insert)
INSERT INTO accounts (account_no, name, balance) VALUES
  ('ACC002', 'Jane Smith', 10000.00),
  ('ACC003', 'Bob Johnson', 2500.00);

-- Insert from another table
INSERT INTO accounts (account_no, name)
SELECT emp_id, full_name FROM employees WHERE dept = 'Finance';
```

> Batch inserts (multiple VALUES rows) are much faster than individual inserts in loops.

---

### SELECT
Query and retrieve data.

```sql
SELECT * FROM accounts;
SELECT id, name, balance FROM accounts;
SELECT name, balance FROM accounts WHERE status = 'ACTIVE';
SELECT name, balance FROM accounts WHERE balance > 1000 ORDER BY balance DESC;
SELECT name, balance FROM accounts LIMIT 10 OFFSET 20;
SELECT COUNT(*), SUM(balance), AVG(balance) FROM accounts;
```

> Avoid `SELECT *` in production — always name the columns you need for better performance.

---

### UPDATE
Modify existing rows.

```sql
UPDATE accounts SET balance = 6000.00 WHERE id = 1;

UPDATE accounts
SET balance = balance - 500.00, updated_at = NOW()
WHERE account_no = 'ACC001' AND status = 'ACTIVE';

UPDATE accounts SET status = 'INACTIVE'
WHERE last_transaction_date < DATE_SUB(NOW(), INTERVAL 1 YEAR);
```

> Always include a `WHERE` clause — without it you update every row in the table.

---

### DELETE
Remove rows from a table.

```sql
DELETE FROM accounts WHERE id = 1;
DELETE FROM accounts WHERE status = 'CLOSED';
DELETE FROM accounts WHERE created_at < '2020-01-01';

DELETE a FROM accounts a
  JOIN customers c ON a.customer_id = c.id
  WHERE c.status = 'DELETED';
```

> Always use `WHERE`. Run a `SELECT` first with the same `WHERE` clause to preview what will be deleted.

---

### REPLACE / UPSERT
Insert or update in one statement.

```sql
REPLACE INTO accounts (id, account_no, name, balance)
VALUES (1, 'ACC001', 'John Doe Updated', 7000.00);

INSERT INTO accounts (account_no, name, balance)
VALUES ('ACC001', 'John Doe', 5000.00)
ON DUPLICATE KEY UPDATE
  balance = VALUES(balance),
  updated_at = NOW();
```

> `ON DUPLICATE KEY UPDATE` is safer than `REPLACE` — it updates rather than deleting and re-inserting.

---

## 4. Query Commands

### WHERE Clauses
Filter rows with conditions.

```sql
SELECT * FROM accounts WHERE balance BETWEEN 1000 AND 5000;
SELECT * FROM accounts WHERE name LIKE 'John%';
SELECT * FROM accounts WHERE status IN ('ACTIVE', 'INACTIVE');
SELECT * FROM accounts WHERE email IS NULL;
SELECT * FROM accounts WHERE email IS NOT NULL;
SELECT * FROM accounts WHERE balance > 1000 AND status = 'ACTIVE';
SELECT * FROM accounts WHERE name LIKE '%Smith%' OR name LIKE '%Jones%';
```

> `LIKE` with a leading `%` (e.g. `'%Smith'`) cannot use an index — it causes a full table scan.

---

### GROUP BY / HAVING
Aggregate and filter groups.

```sql
SELECT status, COUNT(*) as total, SUM(balance) as total_balance
FROM accounts
GROUP BY status;

SELECT branch_id, AVG(balance) as avg_balance
FROM accounts
GROUP BY branch_id
HAVING AVG(balance) > 5000
ORDER BY avg_balance DESC;
```

> `HAVING` filters after grouping. `WHERE` filters before grouping. Use `WHERE` when possible — it's faster.

---

### ORDER BY / LIMIT
Sort and paginate results.

```sql
SELECT * FROM accounts ORDER BY balance DESC;
SELECT * FROM accounts ORDER BY name ASC, balance DESC;
SELECT * FROM accounts ORDER BY balance DESC LIMIT 10;
SELECT * FROM accounts ORDER BY id LIMIT 10 OFFSET 20;
SELECT * FROM transactions ORDER BY created_at DESC LIMIT 100;
```

> For pagination in large tables use `WHERE id > last_seen_id LIMIT N` instead of `OFFSET` — `OFFSET` gets slow at high page numbers.

---

### Subqueries
Nested queries inside queries.

```sql
SELECT * FROM accounts
WHERE balance > (SELECT AVG(balance) FROM accounts);

SELECT name FROM customers
WHERE id IN (
  SELECT customer_id FROM accounts WHERE status = 'ACTIVE'
);

SELECT a.*, t.last_txn FROM accounts a
JOIN (
  SELECT account_id, MAX(created_at) as last_txn
  FROM transactions GROUP BY account_id
) t ON a.id = t.account_id;
```

> Correlated subqueries run once per row — they can be slow. Consider a `JOIN` instead.

---

### CASE WHEN
Conditional logic inside queries.

```sql
SELECT name, balance,
  CASE
    WHEN balance >= 10000 THEN 'Platinum'
    WHEN balance >= 5000  THEN 'Gold'
    WHEN balance >= 1000  THEN 'Silver'
    ELSE 'Standard'
  END AS account_tier
FROM accounts;

SELECT
  SUM(CASE WHEN status='ACTIVE' THEN 1 ELSE 0 END) as active_count,
  SUM(CASE WHEN status='CLOSED' THEN 1 ELSE 0 END) as closed_count
FROM accounts;
```

---

### Date Functions
Work with dates and times.

```sql
SELECT NOW();                                      -- current datetime
SELECT CURDATE();                                  -- current date only
SELECT DATE_FORMAT(created_at, '%Y-%m-%d') FROM accounts;
SELECT DATEDIFF(NOW(), created_at) as days_old FROM accounts;
SELECT * FROM transactions
  WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY);
SELECT YEAR(created_at), MONTH(created_at), COUNT(*)
  FROM transactions GROUP BY YEAR(created_at), MONTH(created_at);
```

---

### String Functions
Manipulate string values.

```sql
SELECT UPPER(name), LOWER(email) FROM accounts;
SELECT CONCAT(first_name, ' ', last_name) as full_name FROM customers;
SELECT LENGTH(account_no), TRIM(name) FROM accounts;
SELECT SUBSTRING(account_no, 1, 3) as prefix FROM accounts;
SELECT REPLACE(phone, '-', '') as clean_phone FROM customers;
SELECT LPAD(account_no, 10, '0') FROM accounts;
```

---

## 5. Joins

### INNER JOIN
Return rows matching in both tables.

```sql
SELECT a.account_no, a.balance, c.name, c.email
FROM accounts a
INNER JOIN customers c ON a.customer_id = c.id
WHERE a.status = 'ACTIVE'
ORDER BY a.balance DESC;
```

> `INNER JOIN` is the most common. Only returns rows where the join condition matches in both tables.

---

### LEFT JOIN
All rows from the left table, with matching rows from the right.

```sql
SELECT c.name, a.account_no, a.balance
FROM customers c
LEFT JOIN accounts a ON c.id = a.customer_id
ORDER BY c.name;

-- Find customers with NO accounts
SELECT c.name
FROM customers c
LEFT JOIN accounts a ON c.id = a.customer_id
WHERE a.id IS NULL;
```

---

### Multiple JOINs
Join three or more tables together.

```sql
SELECT t.id, t.amount, t.created_at,
       a.account_no, c.name as customer_name,
       b.name as branch_name
FROM transactions t
INNER JOIN accounts  a ON t.account_id = a.id
INNER JOIN customers c ON a.customer_id = c.id
INNER JOIN branches  b ON a.branch_id  = b.id
WHERE t.created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
ORDER BY t.created_at DESC;
```

> Alias your tables (t, a, c, b) to keep multi-join queries readable.

---

### UNION
Combine results from multiple queries.

```sql
SELECT 'Savings' as type, account_no, balance FROM savings_accounts
UNION ALL
SELECT 'Checking', account_no, balance FROM checking_accounts
ORDER BY balance DESC;

SELECT name FROM active_customers
UNION
SELECT name FROM archived_customers;
```

> `UNION` removes duplicates (slower). `UNION ALL` keeps duplicates (faster). Use `UNION ALL` unless you specifically need deduplication.

---

## 6. Index Commands

### CREATE INDEX
Speed up query performance.

```sql
CREATE INDEX idx_account_status ON accounts(status);
CREATE INDEX idx_customer_name ON customers(last_name, first_name);
CREATE UNIQUE INDEX idx_account_no ON accounts(account_no);
CREATE INDEX idx_txn_date ON transactions(created_at);
CREATE INDEX idx_amount_date ON transactions(amount, created_at);
```

> Add indexes on columns used in `WHERE`, `JOIN ON`, and `ORDER BY` clauses. Don't over-index — each index slows down `INSERT`/`UPDATE`.

---

### SHOW / DROP INDEX
View and remove indexes.

```sql
SHOW INDEX FROM accounts;
SHOW INDEX FROM transactions;
DROP INDEX idx_account_status ON accounts;
ALTER TABLE accounts DROP INDEX idx_old_index;
```

---

### EXPLAIN
Analyze query execution plan to find performance issues.

```sql
EXPLAIN SELECT * FROM accounts WHERE status = 'ACTIVE';

EXPLAIN SELECT a.*, c.name
  FROM accounts a JOIN customers c ON a.customer_id = c.id
  WHERE a.balance > 5000;

EXPLAIN FORMAT=JSON SELECT * FROM transactions
  WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);
```

> Look at the `type` column: `ALL` = full table scan (bad), `ref`/`eq_ref` = index used (good). `rows` shows estimated rows scanned.

---

## 7. User Management

### CREATE USER
Create a new MySQL user.

```sql
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'StrongPass123!';
CREATE USER 'appuser'@'%' IDENTIFIED BY 'StrongPass123!';
CREATE USER 'readonly'@'localhost' IDENTIFIED BY 'ReadPass456!';
```

> `'%'` allows connection from any host. `'localhost'` restricts to local only. Always create app-specific users — never use root.

---

### GRANT / REVOKE
Give or remove user permissions.

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON bank_db.* TO 'appuser'@'localhost';
GRANT SELECT ON bank_db.* TO 'readonly'@'localhost';
GRANT ALL PRIVILEGES ON bank_db.* TO 'adminuser'@'localhost';
FLUSH PRIVILEGES;

REVOKE DELETE ON bank_db.* FROM 'appuser'@'localhost';
SHOW GRANTS FOR 'appuser'@'localhost';
```

> In a banking app, give the app user only `SELECT`/`INSERT`/`UPDATE`/`DELETE` — never `GRANT` or `DROP`.

---

### DROP USER / ALTER USER
Delete or modify existing users.

```sql
DROP USER 'olduser'@'localhost';
ALTER USER 'appuser'@'localhost' IDENTIFIED BY 'NewPass789!';
ALTER USER 'appuser'@'localhost' PASSWORD EXPIRE;
SELECT User, Host FROM mysql.user;
```

---

## 8. Transactions

### BEGIN / COMMIT / ROLLBACK
Control transaction boundaries.

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE account_no = 'ACC001';
UPDATE accounts SET balance = balance + 500 WHERE account_no = 'ACC002';

-- Check and commit or rollback
IF (SELECT balance FROM accounts WHERE account_no = 'ACC001') >= 0 THEN
  COMMIT;
ELSE
  ROLLBACK;
END IF;
```

> Always wrap money transfers in a transaction. Either both updates succeed or both are rolled back.

---

### SAVEPOINT
Create checkpoints within a transaction.

```sql
START TRANSACTION;

INSERT INTO transactions (account_id, amount) VALUES (1, -500);
SAVEPOINT after_debit;

INSERT INTO transactions (account_id, amount) VALUES (2, 500);
SAVEPOINT after_credit;

-- Roll back to checkpoint if needed
ROLLBACK TO SAVEPOINT after_debit;

COMMIT;
```

> `SAVEPOINT` lets you roll back to a specific point without undoing the whole transaction.

---

### LOCK TABLES / SELECT FOR UPDATE
Prevent concurrent modification of rows.

```sql
LOCK TABLES accounts WRITE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UNLOCK TABLES;

-- Row-level locking (preferred)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
SELECT * FROM accounts WHERE id = 1 LOCK IN SHARE MODE;
```

> `FOR UPDATE` locks the selected rows until the transaction commits — use for read-then-update patterns in banking.

---

## 9. Admin Commands

### SHOW STATUS / VARIABLES
View server status and configuration.

```sql
SHOW STATUS;
SHOW STATUS LIKE 'Threads%';
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;
```

> `SHOW PROCESSLIST` shows currently running queries — useful to find slow or stuck queries in production.

---

### BACKUP / RESTORE
Export and import database data.

```bash
# Export full database (run in terminal)
mysqldump -u root -p bank_db > bank_db_backup.sql

# Export specific tables
mysqldump -u root -p bank_db accounts transactions > partial.sql

# Import / restore
mysql -u root -p bank_db < bank_db_backup.sql
```

```sql
-- Export as CSV from MySQL
SELECT * FROM accounts
INTO OUTFILE '/tmp/accounts.csv'
FIELDS TERMINATED BY ','
ENCLOSED BY '"'
LINES TERMINATED BY '\n';
```

> Schedule daily `mysqldump` backups. In banking environments, test restores regularly — an untested backup is not a backup.

---

### STORED PROCEDURE
Reusable SQL logic block.

```sql
DELIMITER //
CREATE PROCEDURE TransferFunds(
  IN from_acc VARCHAR(20),
  IN to_acc   VARCHAR(20),
  IN amount   DECIMAL(15,2)
)
BEGIN
  DECLARE from_balance DECIMAL(15,2);
  SELECT balance INTO from_balance FROM accounts WHERE account_no = from_acc FOR UPDATE;

  IF from_balance >= amount THEN
    UPDATE accounts SET balance = balance - amount WHERE account_no = from_acc;
    UPDATE accounts SET balance = balance + amount WHERE account_no = to_acc;
    INSERT INTO audit_log (action, amount) VALUES ('TRANSFER', amount);
    COMMIT;
  ELSE
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Insufficient funds';
  END IF;
END //
DELIMITER ;

-- Call the procedure
CALL TransferFunds('ACC001', 'ACC002', 500.00);
```

> Stored procedures run inside the DB — faster than multiple round trips from your Spring Boot app for complex operations.

---

### TRIGGER
Auto-run logic on table insert, update, or delete events.

```sql
-- Prevent negative balance
CREATE TRIGGER before_account_update
BEFORE UPDATE ON accounts
FOR EACH ROW
BEGIN
  IF NEW.balance < 0 THEN
    SIGNAL SQLSTATE '45000'
    SET MESSAGE_TEXT = 'Balance cannot be negative';
  END IF;
END;

-- Auto audit log on insert
CREATE TRIGGER after_transaction_insert
AFTER INSERT ON transactions
FOR EACH ROW
BEGIN
  INSERT INTO audit_log(table_name, action, record_id, changed_at)
  VALUES ('transactions', 'INSERT', NEW.id, NOW());
END;
```

> Triggers are great for audit logging in banking — automatically record every change without relying on application code.

---

### VIEW
Save a complex query as a virtual table.

```sql
CREATE VIEW active_account_summary AS
SELECT
  c.name as customer_name,
  a.account_no,
  a.balance,
  a.status,
  COUNT(t.id) as transaction_count
FROM accounts a
JOIN customers c ON a.customer_id = c.id
LEFT JOIN transactions t ON a.id = t.account_id
WHERE a.status = 'ACTIVE'
GROUP BY a.id;

-- Query the view like a table
SELECT * FROM active_account_summary WHERE balance > 5000;

-- Remove the view
DROP VIEW active_account_summary;
```

> Views simplify complex queries for reporting. They don't store data — they run the underlying query each time.

---

## Quick Reference Summary

| Category | Key Commands |
|---|---|
| Database | `CREATE DATABASE`, `USE`, `DROP DATABASE`, `SHOW DATABASES` |
| Table | `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, `DESCRIBE` |
| CRUD | `INSERT`, `SELECT`, `UPDATE`, `DELETE`, `ON DUPLICATE KEY UPDATE` |
| Query | `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`, `CASE WHEN` |
| Joins | `INNER JOIN`, `LEFT JOIN`, `UNION`, `UNION ALL` |
| Index | `CREATE INDEX`, `EXPLAIN`, `SHOW INDEX`, `DROP INDEX` |
| User | `CREATE USER`, `GRANT`, `REVOKE`, `DROP USER` |
| Transaction | `START TRANSACTION`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `FOR UPDATE` |
| Admin | `SHOW PROCESSLIST`, `mysqldump`, Stored Procedures, Triggers, Views |

---

*Generated for Java Spring Boot Banking Application Reference — MySQL 8.0+*
