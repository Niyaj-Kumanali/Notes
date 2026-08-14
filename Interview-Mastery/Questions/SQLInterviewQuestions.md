# SQL Interview Questions

## Questions

1. What is SQL?
2. What is SQL Server?
3. What is a relational database?
4. What is a database?
5. What is a schema?
6. What is a table?
7. What is a view?
8. What is a stored procedure?
9. What is a function?
10. What is a trigger?
11. What is a constraint?
12. What is a JOIN?
13. What is a subquery?
14. What is a correlated subquery?
15. What is a non-correlated subquery?
16. What is the difference between a JOIN and a subquery?
17. What is a CTE?
18. Why do we use CTEs?
19. What is the difference between CTE and temporary table?
20. Can a CTE be referenced multiple times?
21. Can a CTE be indexed?
22. Does a CTE physically store data?
23. What is a temporary table?
24. What is the difference between #TempTable and ##GlobalTempTable?
25. What is a trigger?
26. What is SQL Server Agent?
27. What is a SQL Server Agent Job?
28. What is Database Mail?
29. How do you configure Database Mail?
30. What is an SMTP server?
31. What is a database backup?
32. Why do we take database backups?
33. What is a full backup?
34. What is a differential backup?
35. What is database security?
36. What is a login?
37. What is a database user?
38. What is a role?
39. Difference between login and user?
40. What are server-level permissions?
41. What are database-level permissions?
42. What is a SQL Server instance?
43. What is a SQL Server database?
44. What is the difference between a server and database?
45. What are the different types of constraints in SQL Server? Explain Primary Key, Foreign Key, Unique, NOT NULL, CHECK, and DEFAULT with practical examples.
46. What is the difference between DELETE, TRUNCATE, and DROP? Explain their impact on transactions, identity values, indexes, and rollback.
47. Explain INNER JOIN, LEFT JOIN, RIGHT JOIN, FULL JOIN, CROSS JOIN, and SELF JOIN. When would you use each one?
48. What is the difference between a CTE, Temporary Table, Table Variable, Derived Table, and Subquery? When would you choose one over another?
49. Explain normalization and denormalization. What is the difference between an OLTP and OLAP database, and how would their designs differ?
50. What is an Execution Plan? What is the difference between an Estimated Execution Plan and an Actual Execution Plan? How do you use an execution plan to troubleshoot a slow query?
51. Explain Table Scan, Clustered Index Scan, Index Scan, Clustered Index Seek, and Nonclustered Index Seek. When is a Scan acceptable, and when is it a performance problem?
52. Suppose a query has an index on the filtering column, but SQL Server is still doing a Table Scan. What could be the reasons, and how would you investigate it?
53. A query that normally takes 5 seconds suddenly takes 10 minutes in production. Walk me through your complete troubleshooting approach.
54. What is a SARGable predicate? Give examples of SARGable and non-SARGable queries and explain how functions, conversions, LIKE, and calculations can affect index usage.
55. What is a covering index? Explain key columns vs included columns. How would you design an index for a query containing WHERE, JOIN, and ORDER BY conditions?
56. What is index fragmentation? Explain Index Rebuild vs Index Reorganize. How would you decide which one to use?
57. Can having too many indexes reduce performance? Explain how indexes affect INSERT, UPDATE, and DELETE operations. How would you identify unnecessary indexes?
58. What are statistics in SQL Server? Why are they important for query optimization? What happens when statistics are outdated?
59. In an execution plan, the estimated rows are 10 but the actual rows are 10 million. What could cause this difference, and what impact can it have on query performance?
60. What are Nested Loops, Hash Match, and Merge Join? How does SQL Server decide which join algorithm to use, and how can a bad choice affect performance?
61. What is a Key Lookup? When does it occur, why can it become expensive, and how can you eliminate or reduce it?
62. What is an implicit conversion? How can datatype mismatches between JOIN or WHERE columns cause performance problems? How would you identify it in an execution plan?
63. What is parameter sniffing? Give a production scenario where a stored procedure performs well for one parameter but poorly for another. How would you troubleshoot and fix it?
64. A query contains multiple JOINs, subqueries, CTEs, and millions of rows. How would you systematically optimize it without randomly adding indexes or rewriting everything?
65. Explain ROW_NUMBER(), RANK(), and DENSE_RANK(). How would you use them to find the latest record per customer, top 3 records per department, or remove duplicates?
66. What is the difference between a Stored Procedure, View, Scalar Function, Inline Table-Valued Function, and Multi-Statement Table-Valued Function? Which can have performance issues and why?
67. What is dynamic SQL? What is the difference between EXEC() and sp_executesql? How do you prevent SQL Injection and maintain good execution-plan behavior?
68. What are transactions and ACID properties? Explain isolation levels, blocking, locking, and deadlocks. How would you troubleshoot a blocking or deadlock issue in production?
69. What is Query Store? How can you use it to identify slow queries, plan changes, and query-performance regressions in production?
70. A SQL Server Agent Job that normally completes in 30 minutes has been running for 4 hours. How would you investigate whether the problem is blocking, query performance, resource pressure, or data volume?
71. Explain Full, Differential, and Transaction Log backups. If a production database fails at 3:45 PM and you need to restore it to 3:40 PM, explain the complete recovery approach.
72. What is a Linked Server? What is the difference between a four-part query and OPENQUERY? If a query joining a local table with a remote table takes 30 minutes, how would you optimize it?
73. A production report suddenly returns different numbers from the application. How would you investigate the data mismatch? Explain how you would validate source data, joins, filters, duplicates, NULLs, and aggregation logic.
74. Tell me about the most complex SQL performance problem you have personally solved. What was the original query and execution time, what did you identify from the execution plan, what changes did you make, and what was the final performance improvement?
75. What is a Nested Loops Join? How does it work internally?
76. If Table A has 1 million rows and Table B has 10 million rows, when might SQL Server choose a Nested Loops Join?
77. What happens if the join columns have different data types?

---

## Answers

### 1. What is SQL?
SQL (Structured Query Language) is the standard language used to communicate with relational databases. It lets you create, read, update, and delete data, and manage database objects like tables and views. It is a *declarative* language — you specify **what** you want, and the database engine decides **how** to get it.

```sql
-- Query: get all engineering employees
SELECT Name, Salary
FROM Employees
WHERE Department = 'Engineering';
```

- SQL has sub-languages: **DDL** (CREATE/ALTER/DROP), **DML** (SELECT/INSERT/UPDATE/DELETE), **DCL** (GRANT/REVOKE), and **TCL** (COMMIT/ROLLBACK).
- SQL Server's dialect is called **T-SQL** (Transact-SQL).

### 2. What is SQL Server?
SQL Server is Microsoft's relational database management system (RDBMS). It stores data in tables, uses the T-SQL dialect, and provides enterprise features like indexing, transactions, high availability (Always On), security, and Business Intelligence tools (SSIS, SSRS, SSAS).

```sql
-- Creating a database and a table in SQL Server
CREATE DATABASE CompanyDB;
GO
USE CompanyDB;
GO
CREATE TABLE Employees (
    EmployeeID INT IDENTITY(1,1) PRIMARY KEY,
    Name NVARCHAR(100) NOT NULL,
    Salary DECIMAL(10,2)
);
```

- It supports ACID transactions, so data stays consistent even on failure.
- It runs as a Windows service (the **SQL Server Database Engine**), and you manage it with SSMS, Azure Data Studio, or `sqlcmd`.

### 3. What is a relational database?
A relational database stores data in **tables** (relations) made of rows and columns, and links tables together using **keys**. It enforces integrity with constraints and supports powerful set-based queries via JOINs.

```sql
-- Two related tables linked by a foreign key
CREATE TABLE Customers (CustomerID INT PRIMARY KEY, Name NVARCHAR(100));
CREATE TABLE Orders (
    OrderID INT PRIMARY KEY,
    CustomerID INT NOT NULL,
    Amount DECIMAL(10,2),
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
);
```

- **Primary key** uniquely identifies a row inside one table.
- **Foreign key** links a row in one table to a row in another, preventing orphan records.

### 4. What is a database?
A database is an organized, persistent collection of data plus the objects that structure it (tables, views, procedures, indexes). In SQL Server, a database is a logical container with its own data files (`.mdf`) and log files (`.ldf`).

```sql
CREATE DATABASE SalesDB;
GO
-- A database holds many related tables
USE SalesDB;
CREATE TABLE Products (ProductID INT PRIMARY KEY, Name NVARCHAR(100));
```

- A database is different from a *DBMS*: the DBMS is the software (SQL Server), the database is the data + structure managed by it.

### 5. What is a schema?
The word *schema* has two related meanings in SQL Server:
1. **Structural schema** — the overall design of the tables, columns, keys, and relationships.
2. **Namespace schema** — a logical container inside a database that groups objects (tables, views, procedures) and can be used for permission management.

```sql
-- Namespace usage: a Sales schema holding related objects
CREATE SCHEMA Sales;
GO
CREATE TABLE Sales.Orders (OrderID INT PRIMARY KEY);
-- Default schema is usually 'dbo': dbo.Orders is fully qualified
SELECT * FROM Sales.Orders;
```

- Using schemas keeps large databases organized (e.g., `Sales.Orders`, `HR.Employees`).
- You can grant permissions on a whole schema at once.

### 6. What is a table?
A table is the basic storage unit of a relational database. It has a fixed set of **columns** (attributes, each with a data type and rules) and a variable number of **rows** (records).

```sql
CREATE TABLE Employees (
    EmployeeID INT IDENTITY(1,1) NOT NULL,   -- auto-increment
    Name       NVARCHAR(100) NOT NULL,
    Department NVARCHAR(50),
    Salary     DECIMAL(10,2),
    CONSTRAINT PK_Employees PRIMARY KEY (EmployeeID)
);

INSERT INTO Employees (Name, Department, Salary)
VALUES ('Niyaj', 'Engineering', 100000.00);
```

- Tables store data physically; unlike views or CTEs, a table has its own storage.
- Rules on columns (constraints) keep data clean at the source.

### 7. What is a view?
A view is a **virtual table** — a saved `SELECT` query that you can query like a table. It does not store data itself (unless it is an indexed/materialized view); it just presents a predefined result set.

```sql
CREATE VIEW vw_EngineeringSalaries AS
SELECT Name, Salary
FROM Employees
WHERE Department = 'Engineering';

-- Query the view
SELECT * FROM vw_EngineeringSalaries;
```

- **Benefits:** simplifies complex queries, adds a security layer (show only some columns), and enforces consistent logic.
- **Limitations:** no parameters, and you cannot always `INSERT/UPDATE/DELETE` through a view (must be a single base table, no aggregation, etc.).

### 8. What is a stored procedure?
A stored procedure is a **precompiled, reusable block of T-SQL** stored in the database. It accepts input/output parameters, can contain transactions, error handling, and multiple statements, and can return result sets.

```sql
CREATE PROCEDURE GetEmployeesByDept
    @Department NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;
    SELECT Name, Salary
    FROM Employees
    WHERE Department = @Department;
END;

-- Execute it
EXEC GetEmployeesByDept @Department = 'Engineering';
```

- **Benefits:** reuse, security (grant EXECUTE instead of direct table access), encapsulation of business logic, and better plan reuse (when parameterized properly).
- Procedures are the recommended place for multi-step business logic.

### 9. What is a function?
A function is a T-SQL routine that **must return a value** (scalar or table) and is deterministic or near-deterministic. Unlike a stored procedure, a function cannot change data or run DDL, and cannot use side-effecting operations like `INSERT/UPDATE/DELETE` on permanent tables.

```sql
-- Scalar function: returns a single value
CREATE FUNCTION dbo.AnnualSalary (@Monthly DECIMAL(10,2))
RETURNS DECIMAL(12,2)
AS
BEGIN
    RETURN @Monthly * 12;
END;

SELECT dbo.AnnualSalary(100000.00) AS AnnualSalary;

-- Table-valued function: returns a result set
CREATE FUNCTION dbo.Engineers ()
RETURNS TABLE
AS
RETURN (SELECT * FROM Employees WHERE Department = 'Engineering');
```

- **Scalar functions** can be embedded in `SELECT` and `WHERE` clauses.
- Scalar UDFs are often slow because SQL Server runs them **row by row** — avoid them in hot queries.

### 10. What is a trigger?
A trigger is a special stored-procedure-like object that **runs automatically** in response to an event: DML (`INSERT/UPDATE/DELETE`) or DDL (`CREATE/ALTER/DROP`), or logon events.

```sql
CREATE TRIGGER trg_AuditSalaryChanges
ON Employees
AFTER UPDATE
AS
BEGIN
    INSERT INTO SalaryAudit (EmployeeID, OldSalary, NewSalary, ChangedBy, ChangedAt)
    SELECT i.EmployeeID, d.Salary, i.Salary, SUSER_SNAME(), GETDATE()
    FROM inserted i
    JOIN deleted d ON i.EmployeeID = d.EmployeeID
    WHERE d.Salary <> i.Salary;
END;
```

- **inserted** and **deleted** pseudo-tables hold the affected rows inside the trigger.
- **Caution:** triggers run implicitly and add overhead to every DML; hidden logic and recursion can cause hard-to-debug issues. Use them for auditing or complex cascades that cannot be done with constraints.

### 11. What is a constraint?
A constraint is a rule enforced at the database level that guarantees data integrity — the data that enters the table must satisfy the rule, otherwise the statement fails.

```sql
CREATE TABLE Orders (
    OrderID    INT PRIMARY KEY,                  -- uniqueness + identity
    CustomerID INT NOT NULL,                     -- no NULLs
    Status     NVARCHAR(20) CHECK (Status IN ('Pending','Shipped','Delivered')),
    Amount     DECIMAL(10,2) DEFAULT 0.00,       -- default value
    CONSTRAINT FK_Orders_Customers FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
);
```

- **Types:** PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, CHECK, DEFAULT (detailed in Q45).
- Constraints are the last line of defense — application code can be buggy, constraints protect the data no matter what.

### 12. What is a JOIN?
A JOIN combines rows from two or more tables based on a related condition (usually matching key values), producing a single result set.

```sql
SELECT c.Name, o.OrderID, o.Amount
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID;
```

- **INNER JOIN** returns only matching rows; **OUTER JOINs** (LEFT/RIGHT/FULL) preserve unmatched rows from one or both sides.
- The `ON` clause defines the relationship; without it you get a CROSS JOIN (Cartesian product).

### 13. What is a subquery?
A subquery is a `SELECT` query nested inside another query — inside `WHERE`, `HAVING`, `FROM`, or `SELECT`. It returns a value, a single row, or a set of values.

```sql
-- Subquery returning a set of values used with IN
SELECT Name FROM Employees
WHERE DepartmentID IN (SELECT DepartmentID FROM Departments WHERE Active = 1);

-- Scalar subquery returning a single value
SELECT Name,
       (SELECT MAX(Salary) FROM Employees) AS MaxSalary
FROM Employees;
```

- Subqueries can often be rewritten as JOINs; the optimizer usually treats them the same, but reading them clearly matters for maintainability.

### 14. What is a correlated subquery?
A correlated subquery **references a column from the outer query**, so it must be evaluated once for each outer row. It depends on the outer query — it cannot run by itself.

```sql
-- Employees earning more than the average salary of their own department
SELECT e.Name, e.Department, e.Salary
FROM Employees e
WHERE e.Salary > (
    SELECT AVG(Salary)
    FROM Employees
    WHERE Department = e.Department    -- correlated reference
);
```

- Because it runs per row, it can be slow on large tables — but for small outer result sets it is perfectly fine and reads naturally.
- The same logic can usually be rewritten with a window function or a JOIN on a grouped subquery.

### 15. What is a non-correlated subquery?
A non-correlated subquery is **independent** of the outer query — it can run on its own and is typically evaluated once.

```sql
-- Runs once: finds the employee with the highest salary overall
SELECT Name, Salary
FROM Employees
WHERE Salary = (SELECT MAX(Salary) FROM Employees);

-- Set-based: departments that currently have no employees
SELECT * FROM Departments
WHERE DepartmentID NOT IN (SELECT DepartmentID FROM Employees);
```

- Since it is evaluated once, it is usually cheaper than a correlated one.
- The result is either a scalar, a single row, or a set of values consumed by the outer query.

### 16. What is the difference between a JOIN and a subquery?
Conceptually they can return the same results, but they differ in structure and behavior:

| Aspect | JOIN | Subquery |
|---|---|---|
| Result | Columns from **both** tables are available in the output | Usually only the outer query's columns are returned |
| Duplication | Can duplicate rows if the join multiplies matches | `IN/EXISTS` won't duplicate outer rows |
| Readability | Better when you need columns from both sides | Better for "find rows where X exists/matches" checks |

```sql
-- Same result, two ways:
-- JOIN
SELECT DISTINCT c.Name
FROM Customers c
INNER JOIN Orders o ON c.CustomerID = o.CustomerID
WHERE o.Amount > 500;

-- Subquery
SELECT Name FROM Customers
WHERE CustomerID IN (SELECT CustomerID FROM Orders WHERE Amount > 500);
```

- Use JOIN when you need data from the joined table; use `EXISTS` (subquery) for existence checks — the optimizer often converts `IN` to the same plan anyway.

### 17. What is a CTE?
A CTE (Common Table Expression) is a **named, temporary result set** defined with the `WITH` clause, scoped to a single statement. It makes complex queries easier to read by breaking them into logical steps.

```sql
WITH HighEarners AS (
    SELECT Name, Department, Salary
    FROM Employees
    WHERE Salary > 90000
)
SELECT Department, COUNT(*) AS Count
FROM HighEarners
GROUP BY Department;
```

- The CTE is not stored; it behaves like an inline view or derived table.
- CTEs also support **recursion** (recursive CTEs), which joins themselves repeatedly — useful for hierarchies.

### 18. Why do we use CTEs?
CTEs improve **readability, maintainability, and logical structuring** of SQL:

- Break a monster query into named, self-documenting steps.
- Reference the same logical block multiple times in one statement (see Q20).
- Enable **recursive queries** (org charts, bill-of-materials, tree structures).
- Make code review easier and reduce deeply nested subqueries.

```sql
WITH Sales AS (
    SELECT CustomerID, SUM(Amount) AS Total
    FROM Orders GROUP BY CustomerID
)
SELECT TOP 5 *
FROM Sales
ORDER BY Total DESC;               -- Top 5 customers by spend
```

- Note: CTEs do not add performance by themselves — the optimizer treats them like inline subqueries. For materialization, use a temporary table.

### 19. What is the difference between CTE and temporary table?

| Feature | CTE | Temporary Table |
|---|---|---|
| Storage | No physical storage (inlined by optimizer) | Physically stored in tempdb |
| Scope | Single statement only | Session (local) or all sessions (global) |
| Indexing | Cannot be indexed | Can have indexes |
| Statistics | Inherits base-table statistics | Can update statistics explicitly |
| Reuse | Per statement (or multiple refs in same statement) | Across many statements in a session |
| When to use | Readability, simple one-shot logic, recursion | Multi-statement flows, large intermediate results, repeated use |

```sql
-- CTE: scoped to this one statement
WITH c AS (SELECT * FROM Employees WHERE Active = 1)
SELECT * FROM c;

-- Temp table: usable in later statements
SELECT * INTO #ActiveEmployees FROM Employees WHERE Active = 1;
SELECT COUNT(*) FROM #ActiveEmployees;
```

- For a **large** intermediate result that is used many times, a temp table (with an index) is usually faster because it avoids re-evaluating the CTE repeatedly.

### 20. Can a CTE be referenced multiple times?
Yes — **within the same statement**. You can reference the same CTE name several times in the one query that follows its definition. You cannot use it across separate statements (each statement needs its own `WITH`).

```sql
WITH EmployeeStats AS (
    SELECT Department, AVG(Salary) AS AvgSal, MAX(Salary) AS MaxSal
    FROM Employees GROUP BY Department
)
SELECT e.Name, e.Department, e.Salary
FROM Employees e
JOIN EmployeeStats s ON e.Department = s.Department
WHERE e.Salary > s.AvgSal;        -- referenced once

-- same CTE, referenced twice (self-join pattern)
WITH Pairs AS (SELECT * FROM EmployeeStats)
SELECT a.Department, a.AvgSal, b.MaxSal FROM Pairs a JOIN Pairs b ON 1=1;
```

- Repeated references do **not** cache data — the CTE may be evaluated multiple times (materialize with a temp table if that matters).

### 21. Can a CTE be indexed?
**No.** A CTE is a logical construct, not a stored object, so you cannot create an index on it directly.

```sql
-- This is NOT allowed:
-- WITH x AS (...) 
-- CREATE INDEX ix ON x(Column)   -- error

-- Workaround: put the data in a temp table, then index it
SELECT * INTO #tmp FROM Employees WHERE Active = 1;
CREATE INDEX ix_tmp ON #tmp(Department);
```

- If you need an index on an intermediate result, materialize it into a **temporary table** and create the index there.

### 22. Does a CTE physically store data?
**No** — a CTE does not store data on its own. It is basically an inline view; SQL Server merges (inlines) it into the outer query. The optimizer *may* decide to spool parts to tempdb internally, but you have no control over it.

```sql
WITH Big AS (SELECT * FROM Orders)   -- no storage created here
SELECT * FROM Big WHERE Amount > 1000;
```

- The practical implication: referencing a complex CTE multiple times can recompute it each time. If you truly need materialization (store once), use a temp table.

### 23. What is a temporary table?
A temporary table is a table created in **tempdb** that exists for the duration of a session or batch. Local temp tables use a single `#`, global ones use `##`.

```sql
-- Local temp table: visible only in my session
CREATE TABLE #ActiveCustomers (
    CustomerID INT PRIMARY KEY,
    Name NVARCHAR(100)
);

INSERT INTO #ActiveCustomers
SELECT CustomerID, Name FROM Customers WHERE Active = 1;

SELECT * FROM #ActiveCustomers;
```

- Temp tables can have indexes, statistics, and constraints.
- They are ideal for multi-step processing: store a working set once, then query it several times.

### 24. What is the difference between #TempTable and ##GlobalTempTable?

| Feature | #LocalTemp | ##GlobalTemp |
|---|---|---|
| Visibility | Only the creating session | All sessions |
| Lifetime | Ends when session ends (or batch, if created in stored proc) | Ends when the creating session ends AND all others finish using it |
| Name visibility | Collides with other sessions' `#x` (SQL Server appends a suffix) | Single shared name |
| Permissions | Private | Any session can read/update |

```sql
CREATE TABLE #Local (ID INT);      -- private to this session
CREATE TABLE ##Global (ID INT);    -- visible to all sessions
```

- Use local temp tables almost always; global temp tables are rare and shared — easy to create cross-session interference.

### 25. What is a trigger?
See Q10 — a trigger is a database object that automatically executes code when a DML (INSERT/UPDATE/DELETE), DDL, or logon event occurs. A quick reminder of the pattern:

```sql
CREATE TRIGGER trg_PreventDelete ON Orders
INSTEAD OF DELETE
AS
BEGIN
    RAISERROR('Orders cannot be deleted.', 16, 1);
    ROLLBACK;
END;
```

- **AFTER** triggers run after the operation; **INSTEAD OF** triggers replace the operation.
- Keep triggers simple; hidden side effects and overhead are the most common complaints.

### 26. What is SQL Server Agent?
SQL Server Agent is a **scheduler service** that runs automated tasks on a SQL Server instance: jobs, schedules, alerts, and operators. It is used for backups, index maintenance, data imports, and any recurring T-SQL or OS-level task.

```sql
-- Agent is a Windows service; typical jobs you schedule:
-- 1. Nightly full backup
-- 2. Weekly index rebuild + stats update
-- 3. Data purge / archiving
-- 4. ETL refresh for reporting
```

- Each job is made of **steps**, runs on a **schedule**, can notify **operators**, and can send emails via **Database Mail**.
- Agent needs to be running and properly configured (SQL Server Agent service account) for jobs to fire.

### 27. What is a SQL Server Agent Job?
A SQL Server Agent Job is a defined unit of automated work made of one or more **steps**, an optional **schedule**, **alerts**, and **notification** settings.

```sql
-- Conceptually a job looks like this:
-- Job: NightlyBackup
--   Step 1: BACKUP DATABASE SalesDB TO DISK = 'D:\Backups\SalesDB.bak'
--   Step 2: if Step 1 succeeds -> send success email; else retry or fail
```

- Steps can run T-SQL, PowerShell, CmdExec, SSIS packages, etc.
- Jobs are stored in **msdb** and managed via SSMS → SQL Server Agent, or `msdb.dbo.sp_add_job`, `sp_add_jobstep`, `sp_add_schedule`.

### 28. What is Database Mail?
Database Mail is a SQL Server feature that lets the server **send email** (via SMTP) from T-SQL — commonly used to notify DBAs about job failures, backup results, alerts, or to send report output.

```sql
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'DBA Mail',
    @recipients   = 'dba@company.com',
    @subject      = 'Backup completed',
    @body         = 'Nightly backup of SalesDB finished successfully.';
```

- It uses an **SMTP server**, a **profile**, and optionally an **account** with authentication.
- Email goes to **msdb**'s mail queue (`sysmail_*` tables) and is sent asynchronously.

### 29. How do you configure Database Mail?
Configuration has these steps:

1. **Enable Database Mail** (if disabled) via Surface Area Configuration or `sp_configure`.
2. Create a **Database Mail account** — SMTP server address, port (usually 587/25), and credentials.
3. Create a **profile** and add the account to it.
4. Grant the profile to users (`msdb` role `DatabaseMailUserRole`).
5. **Test** with `sp_send_dbmail`.

```sql
EXEC msdb.dbo.sysmail_add_account_sp
    @account_name = 'DBA Account',
    @mailserver_name = 'smtp.company.com',
    @port = 587;

EXEC msdb.dbo.sysmail_add_profile_sp @profile_name = 'DBA Mail';
EXEC msdb.dbo.sysmail_add_profileaccount_sp
    @profile_name = 'DBA Mail', @account_name = 'DBA Account', @sequence_number = 1;
```

- Verify with the **Database Mail Log** and `sysmail_unsentitems` / `sysmail_sentitems` if emails don't arrive.

### 30. What is an SMTP server?
SMTP (Simple Mail Transfer Protocol) is the standard protocol for **sending email**. An SMTP server accepts outgoing messages, queues them, and routes them to the recipient's mail server.

- In the SQL Server world, Database Mail connects to an SMTP server (e.g., Exchange, Office 365, a relay) to deliver emails.
- Authentication (user/password) and **TLS/SSL** are common requirements for modern SMTP services.
- It only *sends* mail — receiving mail uses a different protocol (POP3/IMAP).

### 31. What is a database backup?
A database backup is a **copy of the database data** (and sometimes log) written to a backup file/device, used to restore the database after failure, corruption, or user error.

```sql
-- Full backup of SalesDB to disk
BACKUP DATABASE SalesDB
TO DISK = 'D:\Backups\SalesDB_Full.bak'
WITH INIT, COMPRESSION;
```

- Backups can be **full**, **differential**, or **transaction log** (see Q33/Q34/Q71).
- Without a valid, tested backup, the database effectively has no recovery capability — a backup that is never restored is a backup that is never trusted.

### 32. Why do we take database backups?
Backups exist to make data **recoverable**:

- **Disaster recovery** — hardware failure, datacenter outage.
- **Human error** — someone deletes rows, drops a table, or runs a bad UPDATE.
- **Corruption** — page or logical corruption from software/hardware issues.
- **Compliance** — regulatory retention requirements.
- Backups define your **RPO** (Recovery Point Objective) and **RTO** (Recovery Time Objective): how much data you can lose and how fast you must be back up.

```sql
-- Best practice pattern: full weekly + differential nightly + log backups every 15-30 min
-- Then a restore can reach near-point-in-time (see Q71).
```

- Always **test restores** on a staging server — untested backups give false confidence.

### 33. What is a full backup?
A full backup copies **all data** (every used data page) plus enough log to make the database consistent. It is the foundation of the recovery model — differential and log backups are applied on top of the latest full backup.

```sql
BACKUP DATABASE SalesDB TO DISK = 'D:\Backups\SalesDB_Full.bak' WITH COMPRESSION;
```

- Full backups take time and space proportional to the database size.
- Typical strategy: full backup nightly or weekly, depending on size and SLA.

### 34. What is a differential backup?
A differential backup captures **all changes since the last full backup** (it does not reset the base). Restoring requires the last full backup + the last differential — and log backups after that, if the recovery model allows.

```sql
BACKUP DATABASE SalesDB TO DISK = 'D:\Backups\SalesDB_Diff.bak' WITH DIFFERENTIAL;
```

- Smaller and faster than a full backup, so it's common nightly.
- Once a new full backup is taken, the old differential chain is replaced.

### 35. What is database security?
Database security protects the data through **authentication** (who are you), **authorization** (what can you do), **data protection** (encryption, masking), and **auditing** (tracking activity).

```sql
-- Layers of security in SQL Server:
-- 1. Authentication: LOGIN at server level
CREATE LOGIN app_user WITH PASSWORD = 'StrongP@ssw0rd!';

-- 2. Authorization: USER mapped in a database
USE SalesDB;
CREATE USER app_user FOR LOGIN app_user;
GRANT SELECT, INSERT, UPDATE ON Sales.Orders TO app_user;

-- 3. Data protection: encrypt columns, TDE, dynamic data masking
-- 4. Auditing: SQL Server Audit to log who did what
```

- Follow **least privilege**: give only the permissions needed.
- Separate credentials for app vs DBA vs reporting.

### 36. What is a login?
A **login** is a server-level identity that lets someone (or an application) **connect** to a SQL Server instance. It authenticates against the instance; by itself it grants no access to any database.

```sql
-- Server-level: can connect to the instance
CREATE LOGIN niyaj WITH PASSWORD = 'StrongP@ss!';
-- Windows login (domain account)
CREATE LOGIN [DOMAIN\niyaj] FROM WINDOWS;
```

- A login becomes useful only when it's mapped to a **database user** with permissions.

### 37. What is a database user?
A **database user** is a database-level identity inside a specific database, mapped to a login, that holds permissions **within that database**.

```sql
USE SalesDB;
CREATE USER niyaj FOR LOGIN niyaj;
GRANT SELECT ON dbo.Orders TO niyaj;
```

- The same login can have different users in different databases, each with different rights.
- Special built-in users: `dbo` (database owner), `guest`, `INFORMATION_SCHEMA`.

### 38. What is a role?
A role is a **group of permissions** that you assign to users, making permission management manageable. SQL Server has **fixed server roles**, **fixed database roles**, and **user-defined roles**.

```sql
USE SalesDB;
CREATE ROLE SalesReader;
GRANT SELECT ON dbo.Orders TO SalesReader;
GRANT SELECT ON dbo.Customers TO SalesReader;

EXEC sp_addrolemember 'SalesReader', 'niyaj';   -- niyaj inherits those grants
```

- Examples of fixed roles: `sysadmin` (server), `db_owner`, `db_datareader`, `db_datawriter` (database).
- Use roles instead of granting to each user individually — it scales and makes audits easier.

### 39. Difference between login and user?

| | Login | User |
|---|---|---|
| Level | Server/instance | Single database |
| Purpose | Connect to the instance | Access objects inside a database |
| Created by | `CREATE LOGIN` | `CREATE USER ... FOR LOGIN ...` |
| Dropping | `DROP LOGIN` | `DROP USER` |
| Analogy | A building pass | A room key inside the building |

```sql
CREATE LOGIN app_login WITH PASSWORD = 'x';       -- server-level
USE SalesDB;
CREATE USER app_user FOR LOGIN app_login;          -- database-level
```

- You must have a login **and** a user (in each database you need) to do anything.

### 40. What are server-level permissions?
Server-level permissions control what a **login** can do across the whole instance: connect, view server state, create databases, manage logins, run extended stored procedures, etc.

```sql
-- Examples
GRANT VIEW SERVER STATE TO app_monitor;    -- read perf/wait stats
GRANT ALTER ANY LOGIN TO sec_admin;        -- manage logins
-- or assign a fixed server role
EXEC sp_addsrvrolemember 'app_monitor', 'processadmin';
```

- Fixed server roles: `sysadmin`, `securityadmin`, `serveradmin`, `processadmin`, `dbcreator`, `bulkadmin`, `diskadmin`.
- Start with the least powerful role needed.

### 41. What are database-level permissions?
Database-level permissions control what a **user** can do inside a database: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `EXECUTE`, `ALTER`, `CREATE TABLE`, etc., scoped to objects or schemas.

```sql
USE SalesDB;
GRANT SELECT ON Sales.Orders TO SalesReader;
GRANT EXECUTE ON dbo.GetEmployeesByDept TO SalesReader;
DENY DELETE ON Sales.Orders TO SalesReader;
```

- Fixed database roles: `db_owner`, `db_securityadmin`, `db_datareader`, `db_datawriter`, `db_ddladmin`.
- Permission precedence: `DENY` overrides `GRANT`. Users can inherit permissions through roles and schema membership.

### 42. What is a SQL Server instance?
An **instance** is a separate, independent installation of the SQL Server database engine on a machine. Each instance has its own services, port, databases, logins, and configuration. One machine can run several instances (default + named instances).

```sql
-- Connecting by instance:
-- Default instance:  SERVERNAME
-- Named instance:    SERVERNAME\INSTANCENAME   (e.g., PRODDB\BI)
SELECT @@SERVERNAME AS ServerName, @@VERSION AS Version;
```

- Instances share machine resources (CPU/RAM) but are otherwise isolated.
- Useful for separating environments or workloads on one box, but each instance costs memory and management overhead.

### 43. What is a SQL Server database?
A SQL Server database is the **container for data and database objects** within an instance — tables, views, procedures, indexes, users, schemas. Physically it maps to at least two files: a primary data file (`.mdf`) and a transaction log (`.ldf`).

```sql
CREATE DATABASE SalesDB ON
  (NAME = SalesDB, FILENAME = 'D:\Data\SalesDB.mdf'),
  (NAME = SalesDB_Log, FILENAME = 'D:\Log\SalesDB_log.ldf');
```

- Each database has its own **recovery model** (SIMPLE/FULL/BULK_LOGGED) that decides log backup behavior.
- System databases: `master`, `model`, `msdb`, `tempdb` — and user databases like `SalesDB`.

### 44. What is the difference between a server and database?

| | Server (Instance) | Database |
|---|---|---|
| Scope | Hosts many databases | One logical container |
| Objects | Logins, server config, services | Tables, views, users, procedures |
| Examples | `PRODDB\MAIN` | `SalesDB`, `InventoryDB` |
| Analogy | The apartment building | An apartment unit inside it |

```sql
SELECT DB_NAME() AS CurrentDB;      -- which database I'm in
SELECT name FROM sys.databases;     -- databases on this instance
```

- A connection goes: **instance** (login) → **database** (user) → **objects**.

### 45. What are the different types of constraints in SQL Server? Explain Primary Key, Foreign Key, Unique, NOT NULL, CHECK, and DEFAULT with practical examples.
Constraints enforce data integrity at the database level. The six core types:

```sql
CREATE TABLE Customers (
    CustomerID   INT IDENTITY(1,1) PRIMARY KEY,             -- 1. Primary Key
    Email        NVARCHAR(100) UNIQUE,                      -- 3. Unique
    Name         NVARCHAR(100) NOT NULL,                    -- 4. NOT NULL
    Country      NVARCHAR(50) DEFAULT 'IN',                 -- 6. Default
    Age          INT CHECK (Age BETWEEN 18 AND 65),         -- 5. Check
    CustomerCode NVARCHAR(20)
);

CREATE TABLE Orders (
    OrderID    INT PRIMARY KEY,
    CustomerID INT NOT NULL,
    Amount     DECIMAL(10,2) CHECK (Amount > 0),
    CONSTRAINT FK_Orders_Customers                          -- 2. Foreign Key
        FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
        ON DELETE CASCADE
);
```

| Constraint | Purpose | Example behavior |
|---|---|---|
| **PRIMARY KEY** | Uniquely identifies each row; one per table; NOT NULL implied; creates a clustered index by default | `INSERT` with a duplicate `CustomerID` fails |
| **FOREIGN KEY** | Links to a parent table; prevents orphan rows | `INSERT` with a non-existent `CustomerID` fails |
| **UNIQUE** | Ensures distinct values in a column/combination; multiple allowed | Duplicate `Email` rejected |
| **NOT NULL** | Column cannot hold NULL | Inserting `NULL` into `Name` fails |
| **CHECK** | Validates a boolean condition per row | `Age = 70` or `Amount = -5` fails |
| **DEFAULT** | Supplies a value when none is provided | `Country` becomes `'IN'` automatically |

```sql
-- Practical failure demo
INSERT INTO Customers (Email, Name, Age) VALUES ('a@b.com', 'Sam', 30);
INSERT INTO Customers (Email, Name, Age) VALUES ('a@b.com', 'Jo', 25);   -- UNIQUE fails
INSERT INTO Customers (Email, Name, Age) VALUES ('b@c.com', NULL, 25);   -- NOT NULL fails
INSERT INTO Customers (Email, Name, Age) VALUES ('c@d.com', 'Ann', 80);  -- CHECK fails
SELECT * FROM Customers WHERE Name = 'Sam';  -- Country shows 'IN' (default applied)
```

- Constraints are checked **per row** by default (`WITH NOCHECK` skips validation — use carefully for large backfills).

### 46. What is the difference between DELETE, TRUNCATE, and DROP? Explain their impact on transactions, identity values, indexes, and rollback.

```sql
DELETE FROM Orders WHERE OrderID = 10;     -- delete specific rows
TRUNCATE TABLE Orders;                     -- remove all rows fast
DROP TABLE Orders;                         -- remove the table itself
```

| Aspect | DELETE | TRUNCATE | DROP |
|---|---|---|---|
| What it does | Removes matching rows | Removes **all** rows | Removes the **table** (structure + data) |
| WHERE clause | Yes | No | No |
| Logged | Fully logged (row by row) | Logged as page deallocations | Logged as schema change |
| Transactional / rollback | Can be rolled back | Can be rolled back | Can be rolled back (in a transaction) |
| Identity counter | Not reset | **Reset** to seed value | Gone (object removed) |
| Triggers | Fire | Do **not** fire | Do not fire |
| Indexes | Kept, updated | Kept, emptied | Removed with the table |
| Speed | Slow on big tables | Very fast | Fast |
| Permission | DELETE on table | ALTER on table | ALTER/DROP on schema |

```sql
BEGIN TRAN;
DELETE FROM Orders;          -- or TRUNCATE TABLE Orders;
SELECT COUNT(*) FROM Orders; -- 0
ROLLBACK;                    -- rows come back (both DELETE and TRUNCATE can be rolled back)

-- Identity behavior
CREATE TABLE T (ID INT IDENTITY(1,1) PRIMARY KEY, V INT);
INSERT INTO T (V) VALUES (1),(2),(3);
TRUNCATE TABLE T;
INSERT INTO T (V) VALUES (99);  -- ID restarts at 1
```

- **DELETE** is slowest on huge tables but gives row-level control and fires triggers.
- **TRUNCATE** is the fastest way to clear all rows, but it needs `ALTER` permission and won't work on tables referenced by a foreign key.
- **DROP** removes the object entirely — there is nothing left to recover except via backup.

### 47. Explain INNER JOIN, LEFT JOIN, RIGHT JOIN, FULL JOIN, CROSS JOIN, and SELF JOIN. When would you use each one?

```sql
-- Setup
CREATE TABLE Customers (CustomerID INT PRIMARY KEY, Name NVARCHAR(50));
CREATE TABLE Orders (OrderID INT PRIMARY KEY, CustomerID INT, Amount DECIMAL(10,2));
INSERT INTO Customers VALUES (1,'Ann'),(2,'Bob'),(3,'Cat');
INSERT INTO Orders VALUES (10,1,100),(11,1,150),(12,2,200);   -- Cat has no orders
```

| Join | Rows returned | Use when |
|---|---|---|
| **INNER** | Only matching on both sides | "Customers who have ordered" |
| **LEFT** | All left rows + matches from right | "All customers + their orders, show NULL if none" |
| **RIGHT** | All right rows + matches from left | "All orders + customer, keep orphans on right" |
| **FULL** | All rows from both sides, matched where possible | "Complete picture, including unmatched on either side" |
| **CROSS** | Cartesian product (every combination) | Generating combinations (size × price tables) |
| **SELF** | A table joined with itself | Hierarchies: manager–employee, friend pairs |

```sql
-- INNER: only Ann and Bob (Cat has no orders)
SELECT c.Name, o.Amount
FROM Customers c INNER JOIN Orders o ON c.CustomerID = o.CustomerID;

-- LEFT: all customers; Cat appears with NULL amount
SELECT c.Name, o.Amount
FROM Customers c LEFT JOIN Orders o ON c.CustomerID = o.CustomerID;

-- RIGHT: all orders with their customer
SELECT c.Name, o.Amount
FROM Customers c RIGHT JOIN Orders o ON c.CustomerID = o.CustomerID;

-- FULL: unmatched on either side
SELECT c.Name, o.OrderID
FROM Customers c FULL JOIN Orders o ON c.CustomerID = o.CustomerID;

-- CROSS: every customer x every order
SELECT c.Name, o.OrderID
FROM Customers c CROSS JOIN Orders o;

-- SELF: employees and their manager
SELECT e.Name AS Employee, m.Name AS Manager
FROM Employees e LEFT JOIN Employees m ON e.ManagerID = m.EmployeeID;
```

- Practice rule: use `INNER` unless you need to preserve non-matching rows; then choose the side(s) to preserve with LEFT/RIGHT/FULL.

### 48. What is the difference between a CTE, Temporary Table, Table Variable, Derived Table, and Subquery? When would you choose one over another?

| Construct | Scope | Storage | Indexing | Best for |
|---|---|---|---|---|
| **Subquery** | Single expression | None (inlined) | — | Simple filter/set logic |
| **Derived table** | Single statement (in FROM) | None (inlined) | — | One-shot set in a query |
| **CTE** | Single statement | None (inlined), can be recursive | No | Readability + recursion |
| **Table variable** (`@t`) | Batch/session | tempdb (memory-ish, spills) | Only PK/unique constraints | Small sets (<1000 rows) |
| **Temp table** (`#t`) | Session | tempdb, real table | Any index | Larger/multi-use sets |

```sql
-- Derived table (subquery in FROM)
SELECT d.Dept, COUNT(*) FROM (SELECT Department AS Dept FROM Employees) d GROUP BY d.Dept;

-- CTE
WITH d AS (SELECT Department AS Dept FROM Employees)
SELECT Dept, COUNT(*) FROM d GROUP BY d.Dept;

-- Table variable
DECLARE @t TABLE (ID INT PRIMARY KEY);
INSERT INTO @t SELECT EmployeeID FROM Employees WHERE Active = 1;
SELECT * FROM @t;

-- Temp table
SELECT EmployeeID INTO #t FROM Employees WHERE Active = 1;
CREATE INDEX ix_t ON #t(EmployeeID);
SELECT * FROM #t;
```

**Decision guide:**
- Small set, used once → **subquery / derived table / CTE** (clarity decides).
- Needs recursion → **CTE**.
- Very small set used in a loop → **table variable** (cheap, but no stats — optimizer may underestimate).
- Large set, used multiple times, or needs indexes/stats → **temp table**.
- Remember: table variables have **no statistics**, so on large datasets the optimizer guesses badly — that's their #1 performance trap.

### 49. Explain normalization and denormalization. What is the difference between an OLTP and OLAP database, and how would their designs differ?
**Normalization** splits data into separate tables to remove redundancy and ensure each fact is stored once (1NF → 2NF → 3NF...). **Denormalization** intentionally merges/duplicates data to reduce JOINs and speed up reads.

```sql
-- NORMALIZED (3NF): customer info stored once
CREATE TABLE Customers (CustomerID INT PRIMARY KEY, Name NVARCHAR(100), Address NVARCHAR(200));
CREATE TABLE Orders (OrderID INT PRIMARY KEY, CustomerID INT, Amount DECIMAL(10,2));

-- DENORMALIZED (for reporting): customer name repeated on every order row
CREATE TABLE OrderFacts (
    OrderID INT, Amount DECIMAL(10,2),
    CustomerName NVARCHAR(100), CustomerAddress NVARCHAR(200)   -- duplicated
);
```

| | OLTP (transactional) | OLAP (analytical/reporting) |
|---|---|---|
| Goal | Fast, correct transactions | Fast aggregations/analytics |
| Workload | Many small INSERT/UPDATE | Large scans, GROUP BY, few huge reads |
| Design | Normalized (3NF) | Denormalized (star/snowflake, fact + dimension tables) |
| Normalization choice | High | Low (intentional redundancy) |
| Indexes | Narrow, seek-friendly | Wide, columnstore often |
| Concurrency | Heavy locking | Read-mostly |
| Example | Booking system, bank transfer | Monthly sales dashboard |

```sql
-- Star schema: fact table + dimension tables (denormalized for analytics)
CREATE TABLE DimCustomer (CustomerKey INT PRIMARY KEY, Name NVARCHAR(100));
CREATE TABLE FactSales (SaleID INT PRIMARY KEY, CustomerKey INT, Amount DECIMAL(10,2), SaleDate DATE);
```

- Rule of thumb: **normalize for write-heavy OLTP correctness, denormalize for read-heavy reporting** — and use columnstore indexes in SQL Server for large analytic tables.

### 50. What is an Execution Plan? What is the difference between an Estimated Execution Plan and an Actual Execution Plan? How do you use an execution plan to troubleshoot a slow query?
An **execution plan** is the step-by-step strategy SQL Server's optimizer builds to run a query — which indexes it seeks/scans, how it joins tables (nested loop/hash/merge), sorts, and where cost concentrates.

- **Estimated plan:** generated without running the query, based purely on statistics; shows what the optimizer *expects* to happen.
- **Actual plan:** produced by executing the query; includes **actual rows**, **actual executions**, and actual I/O/CPU time.

```sql
-- In SSMS: Ctrl+M (include actual plan), or
SET STATISTICS IO, TIME ON;
SELECT * FROM Orders o JOIN Customers c ON o.CustomerID = c.CustomerID WHERE c.Country = 'US';
```

**How to troubleshoot with a plan:**
1. Look for the most expensive operator (highest `% of cost`).
2. Compare **estimated rows vs actual rows** — a big gap means bad statistics.
3. Hunt for Table Scans, Key Lookups, and Hash Joins on huge inputs.
4. Check the **missing index** suggestion (`MissingIndexDetails`/tooltip).
5. Look for implicit conversions (CONVERT on a column) — see Q62.
6. Fix the biggest lever first (missing index, non-sargable predicate, stale statistics), re-run, and compare plans.

```sql
-- Query the plan's missing-index suggestions
SELECT * FROM sys.dm_db_missing_index_details;
```

- Always validate a change with the **actual** plan and `SET STATISTICS IO` — don't rely on the estimated plan alone.

### 51. Explain Table Scan, Clustered Index Scan, Index Scan, Clustered Index Seek, and Nonclustered Index Seek. When is a Scan acceptable, and when is it a performance problem?

| Operation | What it reads | When it appears |
|---|---|---|
| **Table Scan** (heap) | Every row page of a heap, in no order | No useful index; large portions of a small table |
| **Clustered Index Scan** | All rows via the clustered index (effectively the whole table) | Predicate doesn't filter enough |
| **Index Scan** (nonclustered) | All entries of a nonclustered index | Narrower than the table but still most rows |
| **Clustered Index Seek** | Only the matching range via clustered B-tree | Precise point/range lookup on PK/clustered key |
| **Nonclustered Index Seek** | Only matching entries via a nonclustered B-tree | Filter on an indexed column |

```sql
-- Seek: uses the index on CustomerID
SELECT * FROM Orders WHERE CustomerID = 5;

-- Scan: filtering on an unindexed column (e.g., Status without index)
SELECT * FROM Orders WHERE Status = 'Pending';
```

**When is a scan acceptable?**
- The table is tiny (few pages) — seeking would not pay off.
- The query returns a **large percentage** of rows (e.g., >20–30%) — scanning is cheaper than many lookups.
- The workload is analytic (full table aggregation with columnstore).

**When is it a problem?**
- A large table is scanned for a highly selective predicate — that's a missing-index smell.
- **Key Lookups** hide under an Index Seek and multiply random I/O (see Q61).

### 52. Suppose a query has an index on the filtering column, but SQL Server is still doing a Table Scan. What could be the reasons, and how would you investigate it?
Possible reasons the optimizer ignores the index:

1. **Low selectivity** — the predicate matches most rows (e.g., `Status = 'Active'` on a 95% active table). Seeking would be more expensive than scanning.
2. **Non-SARGable predicate** — a function/wrapper on the column: `WHERE YEAR(OrderDate) = 2025` or `WHERE LOWER(Name) = 'x'` prevents an index seek.
3. **Implicit conversion** — comparing a string column to a number, or `varchar = nvarchar`, blocks index use.
4. **Stale statistics** — the optimizer's row estimate is wrong; refresh with `UPDATE STATISTICS`.
5. **The index doesn't actually serve this query** — it's on a different column/order, or the query selects many columns and the index isn't covering (lookup cost > scan).
6. **The index is disabled or filtered** — a filtered index may not cover the predicate.
7. **Hint/plan forcing** — `WITH (INDEX(0))` or a forced plan from Query Store.
8. **Small table** — optimizer estimates scanning beats seeking.

```sql
-- Investigation:
-- 1. Look at the actual execution plan: what does the predicate look like?
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2025;   -- non-SARGable

-- 2. Check stats freshness
SELECT name, last_updated, rows_sampled, modification_counter
FROM sys.stats s CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id)
WHERE s.name = 'IX_Orders_OrderDate';

-- 3. Verify the index exists and is enabled
SELECT * FROM sys.indexes WHERE name = 'IX_Orders_OrderDate';

-- 4. Test with a hint to confirm index usefulness (diagnostic only)
SELECT * FROM Orders WITH (INDEX(IX_Orders_OrderDate)) WHERE OrderDate >= '2025-01-01';
```

- Fix by rewriting the predicate to be SARGable, creating a proper covering index, or refreshing statistics — depending on which cause you found.

### 53. A query that normally takes 5 seconds suddenly takes 10 minutes in production. Walk me through your complete troubleshooting approach.
A structured, hypothesis-driven approach:

**1. Reproduce and capture the current state**
```sql
-- Get the actual query text (from app/SP/session)
-- Check current blocking and who is running what
SELECT session_id, blocking_session_id, wait_type, wait_time, cpu_time, reads
FROM sys.dm_exec_requests
WHERE session_id > 50;

-- Capture wait stats since restart
SELECT wait_type, wait_time_ms, waiting_tasks_count
FROM sys.dm_os_wait_stats ORDER BY wait_time_ms DESC;
```

**2. Classify the wait** — where is the time going?
- **Blocking locks** (`LCK_M_*`) → another session holds locks (find the blocker).
- **I/O** (`PAGEIOLATCH_*`, `WRITELOG`) → disk slow, log growing.
- **CPU** (`SOS_SCHEDULER_YIELD`, high `cpu_time`) → plan is CPU-heavy (scans, bad join).
- **Memory** (`RESOURCE_SEMAPHORE`) → temp spills to disk.

**3. Get the plan and compare**
```sql
SELECT plan_handle FROM sys.dm_exec_query_stats WHERE sql_handle = ...; 
-- or Query Store: sys.query_store_plan, sys.query_store_runtime_stats
```
- Compare **before/after** plans (Query Store makes this easy — Q69). Look for a scan replacing a seek, a different join operator, or a **stale-statistics** estimate gap.

**4. Check statistics and data growth**
```sql
-- Was a huge amount of data added? Rows/statistics changed?
EXEC sp_spaceused 'Orders';
```

**5. Check parameter sniffing** (Q63) — same query, different parameter, different plan.

**6. Check system health** — CPU/memory/disk of the host, other heavy queries running concurrently.

**7. Fix incrementally:** refresh stats → add/fix index → rewrite predicate → fix plan (WITH RECOMPILE or plan guide only as last resort) → verify with `SET STATISTICS IO, TIME ON`.

- The golden rule: **measure first** (waits, plan, stats), then change one thing at a time and re-measure.

### 54. What is a SARGable predicate? Give examples of SARGable and non-SARGable queries and explain how functions, conversions, LIKE, and calculations can affect index usage.
**SARGable** (Search ARGument-able) means the predicate can be evaluated directly against the index B-tree so the optimizer can **seek**. A predicate is non-SARGable when you wrap the column in a function, conversion, or calculation, forcing an index/table **scan**.

```sql
-- NON-SARGable: function on the column -> scan
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2025;
SELECT * FROM Orders WHERE DATEADD(day, -30, GETDATE()) > OrderDate;  -- actually this one is fine
SELECT * FROM Orders WHERE LEN(Name) = 10;
SELECT * FROM Orders WHERE ISNULL(Status, '') = 'Pending';

-- SARGable rewrite: range predicate -> seek
SELECT * FROM Orders WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01';
```

More examples:

| Non-SARGable | SARGable alternative |
|---|---|
| `WHERE YEAR(OrderDate) = 2025` | `WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01'` |
| `WHERE CONVERT(date, CreatedAt) = '2025-01-01'` | range on `CreatedAt` |
| `WHERE Name + ' ' = 'John'` | `WHERE Name = 'John'` (don't compute on column) |
| `WHERE LEFT(Code, 3) = 'ABC'` | `WHERE Code >= 'ABC' AND Code < 'ABD'` |
| `WHERE Salary * 1.1 > 100` | `WHERE Salary > 100 / 1.1` |
| `WHERE CAST(varcharCol AS INT) = 5` | change datatype / fix comparison types |

**LIKE rules:**
```sql
WHERE Name LIKE 'Ann%'    -- SARGable (leading value known) -> seek
WHERE Name LIKE '%Ann%'   -- NON-SARGable (wildcard at front) -> scan
WHERE Name LIKE '%nn'     -- NON-SARGable
```

- **Implicit conversions** (e.g., comparing `int` column to `varchar`, or `nvarchar` column to `varchar` literal) also block seeks (see Q62).
- Keep the **column alone** on one side of the operator; put all functions/constants on the other side.

### 55. What is a covering index? Explain key columns vs included columns. How would you design an index for a query containing WHERE, JOIN, and ORDER BY conditions?
A **covering index** contains **all** columns a query needs, so SQL Server never has to go back to the base table (no Key Lookup). It covers the query.

- **Key columns** (in the index key) are used for seek/range and to keep the index sorted — put WHERE/JOIN/ORDER BY columns here in the right order.
- **Included columns** (`INCLUDE`) are stored only at the leaf level — they satisfy the SELECT list without being part of the sortable key (you can't seek on them).

```sql
-- Query: WHERE + JOIN + ORDER BY
SELECT o.OrderID, o.OrderDate, o.Amount, c.Name
FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
WHERE o.OrderDate >= '2025-01-01'
ORDER BY o.Amount DESC;

-- Design: key = filter + join, include = select list
CREATE INDEX IX_Orders_OrderDate_CustomerID
ON Orders(OrderDate, CustomerID)
INCLUDE (OrderID, Amount);
```

**Design steps for WHERE + JOIN + ORDER BY:**
1. **WHERE equality** columns first (best selectivity → most to least selective).
2. **WHERE range** column next (only one range column can be used by a seek).
3. **JOIN column** (satisfies the join, or the join column is usually in the key or covered).
4. **ORDER BY** column — if it follows the key columns in the same direction, the index is already sorted and the SORT operator disappears.
5. **INCLUDE** the remaining SELECT columns.

```sql
-- ORDER BY Amount while key is (OrderDate, CustomerID): SORT still needed
-- If you reorder key as (OrderDate, Amount, CustomerID), the index delivers sorted data.
```

- **Trade-off:** covering indexes are wider and slower to maintain on writes — cover only the hot, frequent queries.

### 56. What is index fragmentation? Explain Index Rebuild vs Index Reorganize. How would you decide which one to use?
**Fragmentation** happens when index pages become scattered (page splits, out-of-order pages, empty pages), causing more I/O and less efficient scans. It's measured by **avg_fragmentation_in_percent** (logical fragmentation) via `sys.dm_db_index_physical_stats`.

```sql
SELECT OBJECT_NAME(ps.object_id) AS TableName, i.name AS IndexName,
       ps.avg_fragmentation_in_percent, ps.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'SAMPLED') ps
JOIN sys.indexes i ON ps.object_id = i.object_id AND ps.index_id = i.index_id
WHERE ps.page_count > 1000;
```

| Maintenance | What it does | Cost | When to use |
|---|---|---|---|
| **REORGANIZE** | Defragments leaf pages, logically reorders, low-lock (online-ish), lighter | Low, can run during hours | Fragmentation 5–30% |
| **REBUILD** | Recreates the entire index from scratch | High, blocks (unless `ONLINE = ON`) | Fragmentation > 30%, or just to compact/update stats |

```sql
-- Decide by fragmentation %
ALTER INDEX IX_Orders_OrderDate ON Orders REORGANIZE;          -- 5-30%
ALTER INDEX IX_Orders_OrderDate ON Orders REBUILD WITH (ONLINE = ON);  -- > 30%
```

**Additional decision factors:**
- **Page count:** an index with <1000 pages, even at 60% fragmentation, has almost no impact — rebuild it only if cheap.
- **Workload:** scans suffer more from fragmentation than seeks.
- **Downtime:** rebuild with `ONLINE = ON` on Enterprise/Standard (2022) avoids blocking.
- REBUILD also **refreshes statistics**; REORGANIZE does not (unless you update stats separately).

### 57. Can having too many indexes reduce performance? Explain how indexes affect INSERT, UPDATE, and DELETE operations. How would you identify unnecessary indexes?
Yes. Every index must be **maintained on every write**:

- **INSERT**: each new row is added to every index (B-tree insert, possible page splits).
- **UPDATE**: if an indexed column changes, the index entry must move.
- **DELETE**: every index entry must be removed.
- **Index maintenance** (stats, rebuilds) and **memory** for buffer pool also cost resources.

So write-heavy tables suffer more from excess indexes than read-heavy ones.

```sql
-- Identify rarely used / duplicate indexes:
-- 1. Missing stats on usage
SELECT OBJECT_NAME(s.object_id) AS tbl, i.name AS idx,
       u.user_seeks, u.user_scans, u.user_lookups, u.user_updates,
       (u.user_seeks + u.user_scans + u.user_lookups) AS reads,
       u.user_updates AS writes
FROM sys.dm_db_index_usage_stats u
JOIN sys.indexes i ON u.object_id = i.object_id AND u.index_id = i.index_id
WHERE u.user_updates > 0 AND (u.user_seeks + u.user_scans + u.user_lookups) = 0;

-- 2. Duplicate indexes (same key columns)
SELECT a.object_id, a.name AS IndexA, b.name AS IndexB
FROM sys.indexes a JOIN sys.indexes b
  ON a.object_id = b.object_id AND a.index_id < b.index_id
WHERE a.name IS NOT NULL AND b.name IS NOT NULL;
```

**How to identify unnecessary indexes:**
1. `sys.dm_db_index_usage_stats` — indexes with lots of `user_updates` but zero/near-zero reads.
2. Look for **duplicate/redundant** indexes (same leading columns).
3. Compare read benefit vs write cost; drop unneeded indexes, keep those feeding the top slow queries.
4. Remove **before** a big ETL/bulk load and re-add after if needed.

### 58. What are statistics in SQL Server? Why are they important for query optimization? What happens when statistics are outdated?
**Statistics** are histograms (distribution of values) plus density info that the optimizer uses to **estimate how many rows** a predicate returns. The estimated row count drives every decision: index seek vs scan, join order, join type, memory grants.

```sql
-- View stats info
EXEC sp_helpstats 'Orders', 'ALL';
DBCC SHOW_STATISTICS ('Orders', 'IX_Orders_OrderDate');
```

**Why they matter:** a wrong estimate (stale stats) → wrong plan → scans instead of seeks, hash joins instead of nested loops, huge memory spills, parameter-sniffing-like badness.

**What happens when outdated:**
- Estimated rows diverge wildly from actual rows (e.g., estimate 10, actual 10M — Q59).
- The optimizer picks the wrong index, join order, or join algorithm.
- Symptoms: a query that was fast suddenly becomes slow after data grows.

**Fixing / refreshing:**
```sql
UPDATE STATISTICS Orders IX_Orders_OrderDate;           -- one index
UPDATE STATISTICS Orders;                               -- all on a table
EXEC sp_updatestats;                                    -- all in the database
-- Or let AUTO_UPDATE_STATISTICS / auto-created stats do it (sample-based)
```

- Automatic stats update uses a **sampling threshold** (e.g., ~20% of rows change) — on huge volatile tables, manual/scheduled updates are wise.
- **Index REBUILD** also updates statistics with a full scan.

### 59. In an execution plan, the estimated rows are 10 but the actual rows are 10 million. What could cause this difference, and what impact can it have on query performance?
The optimizer guessed **10 rows**, reality returned **10 million**. Causes:

1. **Stale statistics** — data grew/changed a lot since the last stats update; histogram is old.
2. **Sampled stats** — auto-update sampled too little data; histogram misses new ranges.
3. **Parameter sniffing / local variables** — the optimizer used an estimate based on an unknown value or sniffed parameter, so it guessed a low-density value.
4. **Multi-column correlation** — columns are correlated but stats treat them as independent (e.g., `WHERE City='NY' AND Zip=...`).
5. **String/prefix searches** — `LIKE '%...%'` has no useful stats.
6. **Table variables** — no statistics at all, optimizer assumes 1 row.

```sql
-- Impact example: estimated 10 rows -> optimizer picks Nested Loops + Key Lookup
-- Actual 10M rows -> the same plan does 10M lookups = minutes instead of milliseconds

-- Diagnosis:
SET STATISTICS IO ON;   -- notice high logical reads for small estimates
DBCC SHOW_STATISTICS ('Orders', IX_Orders_OrderDate);  -- check histogram vs reality

-- Fixes:
UPDATE STATISTICS Orders WITH FULLSCAN;    -- accurate stats
OPTION (RECOMPILE);                        -- get fresh estimates for that value
-- consider covering index to remove lookups
```

- **Impact:** bad estimates → wrong index, wrong join type, wrong memory grant (spills to tempdb), and a query that runs orders of magnitude slower than it should.

### 60. What are Nested Loops, Hash Match, and Merge Join? How does SQL Server decide which join algorithm to use, and how can a bad choice affect performance?
The **physical join operators**:

| Join | How it works | Good when | Bad when |
|---|---|---|---|
| **Nested Loops** | For each outer row, probe the inner (indexed) side | Small outer input + indexed inner | Large outer × large inner (O(n×m)) |
| **Merge Join** | Both inputs sorted on join key, walk them together | Large sorted inputs, equality join | Inputs not sorted → expensive sort |
| **Hash Join** | Build a hash table on smaller input, probe with larger | Large unsorted inputs, equality join | Non-equality joins (no hash), memory pressure spills |

```sql
-- You don't write operators; the optimizer chooses based on estimates:
SELECT * FROM Orders o JOIN Customers c ON o.CustomerID = c.CustomerID;
-- small Customers -> Nested Loops with a seek on Orders
-- large both -> Hash or Merge
```

**How the optimizer decides:**
1. Estimate rows per input (using statistics).
2. Estimate cost of each operator (indexed inner for loops, sorted inputs for merge, hash build/probe cost).
3. Pick the cheapest plan overall (cost-based optimization).

**Bad choice impact:**
- **Nested Loops on two 10M-row tables** → billions of probes → minutes/hours.
- **Hash join with a memory grant too small** → spills to tempdb → huge I/O.
- **Merge join forcing a SORT** on big unsorted inputs → expensive sort instead of a hash.

```sql
-- Diagnose & influence: check operator icons in the plan; look for the 
-- "estimated vs actual" mismatch; add/fix indexes, update stats,
-- or in the last resort use a join hint to test: INNER LOOP JOIN / INNER HASH JOIN / INNER MERGE JOIN
```

- Never add join hints in production without testing — they freeze the plan against future data changes.

### 61. What is a Key Lookup? When does it occur, why can it become expensive, and how can you eliminate or reduce it?
A **Key Lookup** (RID Lookup on heaps) happens when a **nonclustered index covers only part** of a query's needs: the index seek finds the matching rows, but the query needs extra columns that live in the clustered index (or heap), so it fetches each row by its clustered key.

```sql
-- Index only on OrderDate, but the query wants Amount too
CREATE INDEX IX_Orders_OrderDate ON Orders(OrderDate);
SELECT OrderID, OrderDate, Amount FROM Orders WHERE OrderDate >= '2025-01-01';
-- Plan: Index Seek on IX_Orders_OrderDate + Key Lookup to get Amount
```

**Why expensive:** a Key Lookup is a **random, single-row read per matched row**. If the seek returns 100K rows, that's up to 100K separate clustered-index lookups — thousands of random I/Os.

**How to eliminate / reduce:**
1. **Add INCLUDE columns** so the nonclustered index covers the query (best fix):
```sql
CREATE INDEX IX_Orders_OrderDate ON Orders(OrderDate) INCLUDE (OrderID, Amount);
```
2. **Covering index** that includes all needed columns (Q55).
3. **Clustered index** on the most-queried range column.
4. Sometimes **shrinking the seek result** (more selective predicate) reduces the lookups proportionally.

- A tiny number of key lookups on a small result is fine; hundreds of thousands on a hot query is a classic tuning target.

### 62. What is an implicit conversion? How can datatype mismatches between JOIN or WHERE columns cause performance problems? How would you identify it in an execution plan?
**Implicit conversion** happens when SQL Server silently converts one datatype to another to compare values. It is visible in the plan as a `CONVERT_IMPLICIT(...)` **on the column side**, which makes the predicate **non-SARGable** → index scans.

```sql
-- SalesAmount is DECIMAL; comparing to a VARCHAR literal -> CONVERT on the column
SELECT * FROM Sales WHERE SalesAmount = '100.50';      -- scan

-- join with mismatched types
CREATE TABLE A (CustomerID INT);   -- int
CREATE TABLE B (CustomerID NVARCHAR(20));  -- nvarchar
SELECT * FROM A JOIN B ON A.CustomerID = B.CustomerID;  -- CONVERT_IMPLICIT on one side
```

**Why it hurts:**
- `CONVERT_IMPLICIT` on a **column** prevents an index seek on that column (the index can't be searched by the converted value).
- Order of precedence: the lower-precedence type converts to the higher one. **nvarchar vs varchar** → the varchar column gets converted → its indexes are useless; **int vs string** → the string side converts.

**How to identify:**
- In SSMS, hover the operator's tooltip and look for `CONVERT_IMPLICIT`.
- Search the actual plan XML for `Convert` / `CONVERT_IMPLICIT`.
- Compare column datatypes across the tables in the join.

```sql
-- Fix by aligning datatypes
ALTER TABLE B ALTER COLUMN CustomerID INT;
-- or use a proper cast on the *constant* side only
SELECT * FROM Sales WHERE SalesAmount = CAST('100.50' AS DECIMAL(10,2));
```

- Rule: **never let the engine convert the column**; convert the literal/parameter side instead.

### 63. What is parameter sniffing? Give a production scenario where a stored procedure performs well for one parameter but poorly for another. How would you troubleshoot and fix it?
**Parameter sniffing** is the optimizer using the **actual parameter value from the first call** to build and cache a plan. That plan is tuned for the sniffed value — it may be terrible for other values that follow.

**Production scenario:**
```sql
CREATE PROCEDURE GetOrders @Country NVARCHAR(50) AS
SELECT * FROM Orders WHERE Country = @Country;

-- 1st call: @Country = 'US' (returns 100 rows) -> plan uses an index seek on Country
EXEC GetOrders 'US';
-- 2nd call: @Country = 'GLOBAL' (returns 9,000,000 rows) -> reuses the 'US' seek plan
EXEC GetOrders 'GLOBAL';   -- 9M key lookups -> 10 minutes when it should hash/scan in 5s
```

**Troubleshoot:**
1. Compare the **plan cache** entry vs a fresh plan (`OPTION (RECOMPILE)`) — the cached plan differs.
2. Check **estimated vs actual rows** for the sniffed value.
3. Use **Query Store** to see plan history and plan choice per value.

**Fixes (in rough order of preference):**
```sql
-- 1. OPTION (RECOMPILE): rebuild plan per call, fresh estimates (good for skew + few calls)
SELECT * FROM Orders WHERE Country = @Country OPTION (RECOMPILE);

-- 2. Parameterize with hints on the statement
EXEC sp_executesql N'SELECT * FROM Orders WHERE Country = @Country OPTION (RECOMPILE)',
                  N'@Country NVARCHAR(50)', @Country = 'GLOBAL';

-- 3. Copy parameter into a local variable (forces generic estimate) -- caution: can backfire
DECLARE @c NVARCHAR(50) = @Country;
SELECT * FROM Orders WHERE Country = @c;

-- 4. Query hints to pick the right operator/plan shape:
SELECT * FROM Orders WHERE Country = @Country OPTION (OPTIMIZE FOR (@Country = 'GLOBAL'));
```

- Choose based on the workload: `RECOMPILE` for skewed data and low call rates; `OPTIMIZE FOR` for a known "typical" value; or fix statistics/indexes so the generic plan is good.

### 64. A query contains multiple JOINs, subqueries, CTEs, and millions of rows. How would you systematically optimize it without randomly adding indexes or rewriting everything?
A methodical, step-by-step approach:

**1. Capture the actual plan and baseline**
```sql
SET STATISTICS IO, TIME ON;
-- run the query, note elapsed time, logical reads, and the actual plan
```

**2. Identify the biggest operator** — sort plan operators by cost; the expensive ones are scans, sorts, hash joins, and key lookups.

**3. Rewrite for clarity first** — inline CTEs into the main query (CTEs are just views), push filters down (apply WHERE as early as possible), convert `SELECT *` to needed columns.

**4. Compare estimated vs actual rows** per operator — find the wrong estimates (stale stats, table variables, correlated columns).

**5. Fix the levers in order:**
- **Statistics** first: `UPDATE STATISTICS ... WITH FULLSCAN` for the mis-estimated tables.
- **Indexes** second: add/fix indexes for the scanned columns, join columns, and ORDER BY (covering indexes for the hot operators).
- **Join strategy** third: after stats/indexes, the optimizer should pick better joins automatically; test with hints only diagnostically.

```sql
-- Example: add an index supporting the most expensive operator
CREATE INDEX IX_Orders_CustomerID_Amount
ON Orders(CustomerID, Amount);
```

**6. Reduce data early** — filter in subqueries, aggregate before joining (fewer rows into joins):
```sql
-- Instead of joining huge detail rows, aggregate first
SELECT d.Dept, s.Total
FROM Departments d
JOIN (SELECT DeptID, SUM(Salary) AS Total FROM Employees GROUP BY DeptID) s
     ON d.DeptID = s.DeptID;
```

**7. Remove duplicates** — if a join multiplies rows, use `DISTINCT` only when truly needed or fix the join.

**8. Verify each change** against the baseline (time + reads + plan diff). Change one thing at a time.

- Rule: **measure → fix estimates → fix indexes → simplify** — never "randomly add indexes or rewrite everything."

### 65. Explain ROW_NUMBER(), RANK(), and DENSE_RANK(). How would you use them to find the latest record per customer, top 3 records per department, or remove duplicates?
Window ranking functions assign numbers to rows inside a partition.

```sql
SELECT EmployeeID, Department, Salary,
       ROW_NUMBER() OVER (PARTITION BY Department ORDER BY Salary DESC) AS RN,
       RANK()       OVER (PARTITION BY Department ORDER BY Salary DESC) AS RK,
       DENSE_RANK() OVER (PARTITION BY Department ORDER BY Salary DESC) AS DRK
FROM Employees;
```

| Function | Behavior on ties | Example (Salaries 100, 90, 90, 80 in a dept) |
|---|---|---|
| **ROW_NUMBER()** | Always unique, arbitrary order among ties | 1,2,3,4 |
| **RANK()** | Ties get same rank, **gaps** follow | 1,2,2,4 |
| **DENSE_RANK()** | Ties get same rank, **no gaps** | 1,2,2,3 |

**Latest record per customer:**
```sql
WITH c AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY OrderDate DESC) AS rn
    FROM Orders
)
SELECT * FROM c WHERE rn = 1;
```

**Top 3 per department:**
```sql
SELECT EmployeeID, Department, Salary FROM (
    SELECT *, RANK() OVER (PARTITION BY Department ORDER BY Salary DESC) AS rk
    FROM Employees
) t WHERE rk <= 3;     -- use RANK to include ties in the top 3
```

**Remove duplicates (keep the newest row):**
```sql
WITH c AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY CustomerID ORDER BY CreatedAt DESC) AS rn
    FROM Customers
)
DELETE FROM c WHERE rn > 1;
```

- **ROW_NUMBER** for deterministic numbering (pagination, dedupe); **RANK** when ties should all count but you accept gaps; **DENSE_RANK** for "top N distinct" like medals (1,2,3 for top three distinct scores).

### 66. What is the difference between a Stored Procedure, View, Scalar Function, Inline Table-Valued Function, and Multi-Statement Table-Valued Function? Which can have performance issues and why?

| Object | Returns | Parameters | Data changes | Typical perf concerns |
|---|---|---|---|---|
| **Stored Procedure** | Result set(s), output params | Yes | Yes (INSERT/UPDATE/DELETE) | Parameter sniffing (Q63) |
| **View** | Virtual table | **No** | Only through a simple view | Indexed views require Enterprise; non-indexed views are just inlined text |
| **Scalar Function** | Single value | Yes | No | Runs **per row** — deadly on large result sets |
| **Inline Table-Valued Function** | Table (single SELECT) | Yes | No | Great — gets inlined into the outer query |
| **Multi-Statement Table-Valued Function** | Table, built in multiple statements | Yes | No | Prevents inlining; optimizer has **no accurate stats**, assumes few rows → bad joins |

```sql
-- Scalar function (row-by-row = slow)
CREATE FUNCTION dbo.Tax(@amount DECIMAL(10,2)) RETURNS DECIMAL(10,2)
AS BEGIN RETURN @amount * 0.18; END;
SELECT OrderID, dbo.Tax(Amount) FROM Orders WHERE OrderID < 100000;   -- 100K calls

-- Inline TVF (inlined, fast)
CREATE FUNCTION dbo.BigOrders(@min DECIMAL(10,2)) RETURNS TABLE
AS RETURN (SELECT * FROM Orders WHERE Amount >= @min);
SELECT * FROM dbo.BigOrders(5000);

-- Multi-statement TVF (not inlined, poor estimates)
CREATE FUNCTION dbo.BigOrdersMS(@min DECIMAL(10,2)) RETURNS @t TABLE (OrderID INT, Amount DECIMAL(10,2))
AS BEGIN INSERT INTO @t SELECT OrderID, Amount FROM Orders WHERE Amount >= @min; RETURN; END;
SELECT * FROM dbo.BigOrdersMS(5000);   -- optimizer assumes tiny @t -> bad joins
```

**Which cause performance issues and why:**
- **Scalar functions** — invoked once per row; the engine can't inline them.
- **Multi-statement TVFs** — not inlined, no stats on the returned table → row-count guesses → wrong joins.
- **Views** — fine unless heavily nested (the optimizer usually flattens them); never index unless necessary.
- **Stored procedures** — main risk is **parameter sniffing**; otherwise they're the preferred performant choice.

### 67. What is dynamic SQL? What is the difference between EXEC() and sp_executesql? How do you prevent SQL Injection and maintain good execution-plan behavior?
**Dynamic SQL** is T-SQL built and executed at runtime — typically because table/column names, filters, or a whole statement is only known at runtime.

```sql
-- EXEC(): string built by concatenation
DECLARE @tbl NVARCHAR(50) = 'Orders';
DECLARE @sql NVARCHAR(MAX) = 'SELECT * FROM ' + @tbl;
EXEC(@sql);

-- sp_executesql: parameterized
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM Orders WHERE Country = @c';
EXEC sp_executesql @sql, N'@c NVARCHAR(50)', @c = 'US';
```

| | `EXEC()` | `sp_executesql` |
|---|---|---|
| Parameters | None — concatenate values | Proper `@params`, `@params...` list |
| Injection risk | High if you concatenate user input | Low if you parameterize |
| Plan reuse | New plan per unique string | Plans reused for the same template + different values |
| Type-safe | No | Yes (typed parameters) |

**Prevent SQL Injection:**
```sql
-- BAD: user input concatenated
DECLARE @u NVARCHAR(50) = 'x''; DROP TABLE Orders;--';   -- attacker input
EXEC('SELECT * FROM Users WHERE Name = ''' + @u + '''');   -- injection!

-- GOOD: parameterize
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM Users WHERE Name = @n';
EXEC sp_executesql @sql, N'@n NVARCHAR(50)', @n = @u;
```

- **Whitelist** dynamic identifiers (table/column names) — never concatenate them raw; validate against `sys.tables`/`sys.columns`.
- **Escaping** for `LIKE` wildcards when needed; validate/sanitize all inputs.
- **Plan behavior:** parameterized dynamic SQL (sp_executesql) reuses cached plans and avoids both injection and plan-cache bloat from thousands of unique strings.

### 68. What are transactions and ACID properties? Explain isolation levels, blocking, locking, and deadlocks. How would you troubleshoot a blocking or deadlock issue in production?
A **transaction** is a unit of work that must fully succeed or fully fail. ACID guarantees:

- **Atomicity** — all or nothing (ROLLBACK undoes partial work).
- **Consistency** — data is valid before and after (constraints hold).
- **Isolation** — concurrent transactions don't see each other's uncommitted mess.
- **Durability** — committed data survives crashes (via the transaction log).

```sql
BEGIN TRAN;
UPDATE Accounts SET Balance = Balance - 100 WHERE AccountID = 1;
UPDATE Accounts SET Balance = Balance + 100 WHERE AccountID = 2;
IF @@ERROR <> 0 BEGIN ROLLBACK; END
ELSE COMMIT;
```

**Isolation levels** (default READ COMMITTED) control what concurrent reads see:

| Level | Dirty reads | Non-repeatable reads | Phantom reads |
|---|---|---|---|
| READ UNCOMMITTED | Yes | Yes | Yes |
| READ COMMITTED | No | Yes | Yes |
| REPEATABLE READ | No | No | Yes |
| SNAPSHOT (row versioning) | No | No | No |
| SERIALIZABLE | No | No | No |

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**Locking & blocking:** transactions take locks (shared/update/exclusive, row→table). When one holds an exclusive lock and another needs it, the second **blocks**. Blocking is normal and short; long blocking is a problem.

**Deadlock:** two sessions hold locks the other needs → SQL Server picks a **victim** (based on cost) and kills it with error 1205; the killed transaction is rolled back and must be retried.

**Troubleshoot blocking:**
```sql
SELECT blocking_session_id, wait_type, wait_time, command
FROM sys.dm_exec_requests WHERE blocking_session_id <> 0;

-- the classic query for the whole blocking chain
SELECT r.session_id, r.blocking_session_id, r.wait_type, r.wait_time,
       t.text
FROM sys.dm_exec_requests r
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id <> 0;
```

**Troubleshoot deadlocks:**
- Enable deadlock capture: `DBCC TRACEON(1222, -1)` and read the SQL Server error log — it logs both sessions, their statements, and the locks involved.
- Fixes: consistent **lock order** across code paths, **short transactions**, indexes to reduce scanned/locked rows, `READ COMMITTED SNAPSHOT` (RCSI) to reduce read blocking, and retry logic in the app.

### 69. What is Query Store? How can you use it to identify slow queries, plan changes, and query-performance regressions in production?
**Query Store** records query text, execution plans, and runtime stats over time, letting you compare plan performance and **force** good plans. It is the DBA's memory of what happened to queries.

```sql
ALTER DATABASE SalesDB SET QUERY_STORE = ON;
ALTER DATABASE SalesDB SET QUERY_STORE (OPERATION_MODE = READ_WRITE, INTERVAL_LENGTH_MINUTES = 5);
```

**Usage patterns:**
1. **Top resource consumers** — find the queries using the most CPU/duration/reads in a time window.
2. **Regressions** — the "Tracked Queries"/regressed queries view shows queries whose duration jumped (a plan changed → new plan is slow).
3. **Plan comparison** — each query's "Plan Summary" lists all plans with stats; compare the fast plan vs the slow one.
4. **Force a plan** — pin the historically good plan with `sp_query_store_force_plan` so the optimizer uses it until the root cause is fixed.
5. **Plan-change history** — see when a plan was created and how its metrics evolved.

```sql
-- Find slow queries
SELECT q.query_id, p.plan_id, rs.avg_duration, rs.avg_logical_io_reads,
       qt.query_sql_text
FROM sys.query_store_query q
JOIN sys.query_store_plan p ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
WHERE rs.last_execution_time > DATEADD(day, -1, GETDATE())
ORDER BY rs.avg_duration DESC;
```

- Query Store makes **parameter sniffing** and plan regressions visible and reversible — a key production tool (Q53, Q63 rely on it).

### 70. A SQL Server Agent Job that normally completes in 30 minutes has been running for 4 hours. How would you investigate whether the problem is blocking, query performance, resource pressure, or data volume?
A systematic triage:

**1. Confirm what it's doing right now**
```sql
SELECT session_id, blocking_session_id, wait_type, wait_time, cpu_time, reads, writes, command, status
FROM sys.dm_exec_requests WHERE session_id > 50;
-- or find the job's session via msdb.dbo.sysjobs / sysjobactivity
```

**2. Classify by wait type:**
- **`LCK_M_*`** → **blocking**: another session holds locks; find the blocker with the chain query (Q68).
- **`PAGEIOLATCH_*` / `WRITELOG` / `ASYNC_IO_COMPLETION`** → **resource pressure**: disk I/O, log growth, or storage slowness.
- **High `cpu_time` with low wait** → **query performance**: plan problem, scans, bad joins, stale stats.
- **`RESOURCE_SEMAPHORE` / tempdb spills** → memory grant issues.
- **Rows processed keep growing** → **data volume**: the job re-scans a table that grew massively, or a loop now iterates far more.

```sql
SELECT wait_type, wait_time_ms, waiting_tasks_count
FROM sys.dm_os_wait_stats ORDER BY wait_time_ms DESC;   -- instance-wide since last restart

-- Compare with job's historical duration
EXEC msdb.dbo.sp_help_jobhistory @job_name = 'NightlyETL';
```

**3. Compare vs baseline** — the job history tells you when it started slipping; check what changed that day (schema, indexes, stats, data volume, hardware).

**4. Dig into the query:**
```sql
-- find the running query text of the job session
SELECT r.session_id, r.wait_type, r.blocking_session_id, t.text
FROM sys.dm_exec_requests r
OUTER APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.session_id = <job_session_id>;

-- run the same query manually with SET STATISTICS IO, TIME ON to compare reads/time
```

**5. Rule in/out data volume** — `EXEC sp_spaceused 'Orders';` and compare row counts vs when it took 30 minutes.

**Conclusion path:** if blocked → resolve the blocker; if CPU-heavy plan → stats/index tuning; if I/O → storage; if data grew → partition/archive/improve the query. Always fix the **primary** bottleneck and re-run to verify.

### 71. Explain Full, Differential, and Transaction Log backups. If a production database fails at 3:45 PM and you need to restore it to 3:40 PM, explain the complete recovery approach.
- **Full backup:** complete copy of the database (foundation of the chain).
- **Differential backup:** all changes since the last **full** backup (faster, smaller).
- **Transaction log backup:** all log records since the last **log** backup — enables point-in-time recovery. Requires the database recovery model to be **FULL**.

```sql
-- Typical cycle: nightly full, hourly differential, log every 15 min
BACKUP DATABASE SalesDB TO DISK = 'D:\Backups\SalesDB_Full.bak';
BACKUP DATABASE SalesDB TO DISK = 'D:\Backups\SalesDB_Diff.bak' WITH DIFFERENTIAL;
BACKUP LOG SalesDB TO DISK = 'D:\Backups\SalesDB_Log.trn';
```

**Restore to 3:40 PM (failure at 3:45):**
1. Restore the **latest full backup** with `NORECOVERY`.
2. Restore the **latest differential** with `NORECOVERY` (it covers changes up to its point in time, based on the full).
3. Restore **all log backups** in order, each with `NORECOVERY` — up to the log containing 3:40.
4. Restore the **final log** with `STOPAT = '2025-01-15T15:40:00'` and **RECOVERY** — replaying to exactly 3:40, dropping later changes.

```sql
RESTORE DATABASE SalesDB FROM DISK = 'D:\Backups\SalesDB_Full.bak' WITH NORECOVERY;
RESTORE DATABASE SalesDB FROM DISK = 'D:\Backups\SalesDB_Diff.bak' WITH NORECOVERY;
RESTORE DATABASE SalesDB FROM DISK = 'D:\Backups\SalesDB_Log1.trn' WITH NORECOVERY;
RESTORE LOG SalesDB FROM DISK = 'D:\Backups\SalesDB_Log2.trn'
   WITH STOPAT = '2025-01-15T15:40:00', RECOVERY;
```

- **Key points:** you can only hit 3:40 if you have log backups covering that time (**FULL recovery model**); log backups taken at 3:15 and 3:30 exist, so STOPAT gets there. If the database were in SIMPLE mode, you'd lose everything since the last backup.

### 72. What is a Linked Server? What is the difference between a four-part query and OPENQUERY? If a query joining a local table with a remote table takes 30 minutes, how would you optimize it?
A **Linked Server** connects one SQL Server to another data source (SQL Server, Oracle, files, etc.) so you can query remote objects with a 4-part name.

```sql
EXEC sp_addlinkedserver @server = 'REMOTESQL', @srvproduct = 'SQL Server';
EXEC sp_addlinkedsrvlogin 'REMOTESQL', 'false', NULL, 'remote_user', 'p@ss';
```

**Four-part query vs OPENQUERY:**
```sql
-- 4-part: SQL Server optimizes locally, may pull the whole remote table
SELECT * FROM LocalTable l
JOIN REMOTESQL.RemoteDB.dbo.RemoteTable r ON l.ID = r.ID;

-- OPENQUERY: runs the query ON the remote server (distributed query passed through)
SELECT * FROM OPENQUERY(REMOTESQL,
  'SELECT * FROM RemoteDB.dbo.RemoteTable WHERE ID IN (SELECT ID FROM ...)');
```

| | 4-part join | OPENQUERY |
|---|---|---|
| Where work happens | Often local (data shipped over) | Remote (query executes there) |
| Optimizer visibility | Local optimizer sees only row-count estimates | Remote executes; local just gets results |
| Best for | Small remote tables | Large remote tables / push logic to source |

**Optimizing a 30-minute local↔remote join:**
1. **Reduce rows remotely first** — filter/aggregate at the source with OPENQUERY so only the needed rows cross the wire:
```sql
SELECT * FROM OPENQUERY(REMOTESQL, '
  SELECT ID, Amount FROM RemoteDB.dbo.RemoteTable
  WHERE OrderDate >= ''2025-01-01'' AND OrderDate < ''2025-02-01''');
```
2. **Create a temp table locally** from the remote result, index it, then join locally:
```sql
SELECT * INTO #remote FROM OPENQUERY(REMOTESQL, 'SELECT ... filtered ...');
CREATE INDEX ix_remote ON #remote(ID);
SELECT * FROM LocalTable l JOIN #remote r ON l.ID = r.ID;
```
3. **Check the plan** — ensure the remote part is the driving/filtered side, not a full remote table pull.
4. Consider **ETL/refresh** of the remote table locally if it's queried often.

- The #1 cause of slowness: shipping the **entire remote table** over the network then joining locally — fix by pushing filters/aggregation to the source.

### 73. A production report suddenly returns different numbers from the application. How would you investigate the data mismatch? Explain how you would validate source data, joins, filters, duplicates, NULLs, and aggregation logic.
A methodical reconciliation:

**1. Reproduce and scope it** — which numbers differ, since when, is it one row or the total.

**2. Validate source data first** — is the *input* the same?
```sql
SELECT COUNT(*) FROM Orders;                    -- row count sanity
SELECT MAX(CreatedAt), MIN(CreatedAt) FROM Orders;  -- timeframe changed?
SELECT * FROM Orders WHERE OrderID IN (SELECT TOP 5 OrderID ...); -- spot-check values
```

**3. Check joins** — a changed join can duplicate or drop rows:
```sql
-- find join multiplies: duplicate key on one side
SELECT o.CustomerID, COUNT(*) FROM Orders o
JOIN Customers c ON o.CustomerID = c.CustomerID
GROUP BY o.CustomerID HAVING COUNT(*) <> 1;    -- customers with duplicate rows?
```

**4. Check filters** — `WHERE` clauses, status values, date ranges may have changed (e.g., new status value not in the report filter).

**5. Check duplicates** — duplicate rows inflate counts:
```sql
SELECT OrderID, COUNT(*) FROM Orders GROUP BY OrderID HAVING COUNT(*) > 1;
```

**6. Check NULLs** — aggregations silently skip NULLs:
```sql
SELECT SUM(Amount)        AS TotalWithNulls,
       SUM(ISNULL(Amount,0)) AS TotalIgnoringNull,
       COUNT(Amount)      AS NonNullCount,
       COUNT(*)           AS AllRows
FROM Orders;
```
- `COUNT(*)` vs `COUNT(col)` behave differently; `SUM`/`AVG` ignore NULLs.

**7. Validate aggregation logic** — same data, different aggregation:
- Are you `SUM`-ing detail rows that were previously aggregated?
- Did an `AVG` vs weighted average change?
- Re-run the report with the same date range as the app and compare row-by-row:
```sql
-- compare app's total vs a fresh aggregation
SELECT CustomerID, SUM(Amount) AS Total FROM Orders
WHERE OrderDate >= '2025-01-01' GROUP BY CustomerID;
```

**8. Compare against a known-good baseline** — last week's correct numbers, or reconcile via an independent query. When fixed, document the root cause (join duplication, NULL handling, filter drift).

### 74. Tell me about the most complex SQL performance problem you have personally solved. What was the original query and execution time, what did you identify from the execution plan, what changes did you make, and what was the final performance improvement?
**Problem:** In my current company, the Lenovo CDMS channel-partner reporting stored procedure was taking **~5 hours** to run. The business needed reports daily; the nightly batch kept failing, so partners' incentive data was delayed.

**Original query:** a large stored procedure with multiple CTEs and subqueries joining a **5-million-row transaction table**, with heavy filtering on `PartnerID` and a date range.

**What I found from the execution plan:**
- A **Table Scan** on the 5M-row table — no useful index on the join/filter column.
- A **non-SARGable predicate** — a `YEAR(ReportDate)`-style function on the filtered column, blocking index seeks.
- A **Key Lookup** pattern on a secondary index that was fetching extra columns for the SELECT list.
- **Stale statistics** — estimated rows (thousands) vs actual rows (millions) mismatched, so the optimizer picked poor join orders.
- A **missing join predicate** in one subquery causing a Cartesian product (massive nested loop).

**What I changed:**
1. Created **composite indexes** on the transaction table — leading with the highly selective `PartnerID`, then the date range column, and `INCLUDE`d the SELECT-list columns to make them **covering**.
2. Rewrote the **non-SARGable predicates** to range comparisons.
3. Refreshed **statistics** with `FULLSCAN`.
4. Added a **missing join predicate** that eliminated the Cartesian product.
5. Validated **output correctness** after every change by comparing old vs new results row-by-row.

**Final result:** execution dropped from **~5 hours to under 12 minutes** (roughly 96% faster), the nightly job ran reliably, and partners got their incentives on time. I kept a change log and verified each optimization against the actual execution plan and `SET STATISTICS IO`.

### 75. What is a Nested Loops Join? How does it work internally?
A **Nested Loops Join** processes an **outer input** row by row; for each outer row it probes the **inner input** (usually via an index seek) to find matching rows.

```sql
-- Optimizer chooses a nested loops plan when the outer is small and the inner is indexed
SELECT * FROM Customers c
JOIN Orders o ON o.CustomerID = c.CustomerID
WHERE c.Country = 'US';
-- Plan: Index Seek (Customers) -> Nested Loops -> Index Seek (Orders)
```

**Internals:**
1. Read **outer row 1** → probe inner index for matches → output each match.
2. Read **outer row 2** → probe again → ...
3. Repeat until the outer input is exhausted.

```
for each outer_row in OuterInput:
    for each inner_row in InnerInput matching predicate:
        output (outer_row, inner_row)
```

- **Cost:** roughly **O(outer × inner)** but with an indexed inner it's really **O(outer × log(inner))** per probe.
- **Best when:** the outer is small and the inner side can **seek** (equality on an indexed column).
- **Worst when:** both inputs are large and the inner side is **scanned** per outer row — that's millions × millions.

### 76. If Table A has 1 million rows and Table B has 10 million rows, when might SQL Server choose a Nested Loops Join?
Even though both tables are large, Nested Loops can still win when the **effective outer input is small**:

1. **A strong filter shrinks the outer** — e.g., `WHERE A.Status = 'Active'` leaves only 1,000 rows of A; then 1,000 probes into indexed B ≈ cheap.
2. **The inner side is indexed** on the join column, so each probe is a **seek**, not a scan.
3. **The optimizer's estimate says few rows from A** (statistics-driven) — it trusts the estimate and picks loops.
4. **The join is non-equality** (e.g., `<`, `>`), which hash/merge can't handle — loops may be the only option.
5. **Plan hints / plan forcing** direct it (even if suboptimal).

```sql
SELECT * FROM A JOIN B ON B.AID = A.ID WHERE A.Active = 1;   -- small outer -> loops
SELECT * FROM A JOIN B ON B.AID = A.ID;                       -- both large -> likely hash
```

- **Danger:** if the estimate is wrong (stale stats — Q59) and A actually yields **1M rows**, the "cheap" loops plan becomes **1M probes → minutes/hours**. That's exactly when a plan regression appears. Fix by refreshing statistics so the optimizer picks a **hash or merge** join instead.

### 77. What happens if the join columns have different data types?
A type mismatch triggers an **implicit conversion** on one side of the join (Q62). SQL Server converts the **lower-precedence** type to the higher-precedence one. The problem: the conversion often lands **on the column** side, making it non-SARGable → the index on that column can't be used for seeks → **scans**, **key lookups**, poor performance.

```sql
-- A.CustomerID is INT, B.CustomerID is VARCHAR
SELECT * FROM A JOIN B ON A.CustomerID = B.CustomerID;
-- plan shows CONVERT_IMPLICIT(INT, [B].[CustomerID]) -> B's index unusable -> scan

-- nvarchar vs varchar: the varchar column is converted to nvarchar, its indexes ignored
SELECT * FROM T1 JOIN T2 ON T1.Code = T2.Code;  -- if Code is varchar on T2, nvarchar on T1
```

**Correctness risk too:** conversions can change matching rules — e.g., comparing `INT` to `VARCHAR` may drop rows that should match, or with weird collations compare differently.

**How to fix:**
```sql
-- 1. Align datatypes on both sides (best)
ALTER TABLE B ALTER COLUMN CustomerID INT;

-- 2. Or convert explicitly on the constant/parameter side only
SELECT * FROM A JOIN B ON A.CustomerID = CAST(B.CustomerID AS INT);  -- still converts B (column) -> scan
-- Better: convert B's value to the indexed column's type:
SELECT * FROM A JOIN B ON A.CustomerID = TRY_CONVERT(INT, B.CustomerID);  -- same problem; convert at source
```

**The real fix:** make the datatypes match at the table level (or in a temp-table staging step) so no column-side conversion happens. **Check the actual plan** for `CONVERT_IMPLICIT` after the change — the scan should become a seek.
