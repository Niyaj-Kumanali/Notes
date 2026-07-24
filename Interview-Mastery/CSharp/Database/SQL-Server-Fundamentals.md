# SQL Server Fundamentals for C# Developers

---

## Database Fundamentals

### What is a Relational Database?

- A **relational database** stores data in structured **tables** (also called relations) with predefined relationships between them
- Based on Edgar F. Codd's relational model (1970), where data is organized into **rows** (tuples) and **columns** (attributes)
- Tables relate to each other through **keys** — a primary key in one table maps to a foreign key in another
- SQL (Structured Query Language) is the standard language to query and manipulate relational data
- SQL Server is Microsoft's enterprise relational database management system (RDBMS)
- A relational database enforces **data integrity** through constraints, ensuring data remains consistent and valid

### Tables, Rows, and Columns

- A **table** is a collection of related data organized in a grid structure — think of it as a spreadsheet
- A **row** (also called a record or tuple) represents a single, complete unit of data in the table
- A **column** (also called a field or attribute) defines a specific piece of data with a name and data type
- Every column has a **data type** that determines what kind of data it stores: `INT`, `VARCHAR`, `DATETIME`, `DECIMAL`, `BIT`, etc.
- Each table can have a maximum of **1,024 columns** and approximately **900 bytes** per row (key columns limited to 900 bytes)

```sql
CREATE TABLE Employees (
    EmployeeId   INT            PRIMARY KEY,
    FirstName    NVARCHAR(50)   NOT NULL,
    LastName     NVARCHAR(50)   NOT NULL,
    Email        NVARCHAR(100)  UNIQUE NOT NULL,
    HireDate     DATE           NOT NULL,
    Salary       DECIMAL(10,2)  NULL,
    DepartmentId INT            NULL
);
```

### Primary Key

- A **primary key** uniquely identifies each row in a table — no two rows can have the same primary key value
- Primary keys **cannot be NULL** (enforced by a `NOT NULL` constraint)
- A table can have **only one** primary key
- SQL Server automatically creates a **clustered index** on the primary key (by default)
- A primary key can be a single column or a **composite key** (multiple columns combined)

```sql
-- Single-column primary key
CREATE TABLE Orders (
    OrderId   INT IDENTITY(1,1) PRIMARY KEY,
    OrderDate DATETIME NOT NULL,
    Total     DECIMAL(10,2) NOT NULL
);

-- Composite primary key
CREATE TABLE OrderItems (
    OrderId   INT NOT NULL,
    ProductId INT NOT NULL,
    Quantity  INT NOT NULL,
    PRIMARY KEY (OrderId, ProductId)
);
```

### Foreign Key

- A **foreign key** is a column (or set of columns) in one table that references the primary key of another table
- Enforces **referential integrity** — you cannot insert a row with a foreign key value that does not exist in the parent table
- Prevents deleting parent rows that have matching child rows (by default)
- Creates a parent-child relationship between tables
- Foreign keys can reference any **unique key** or **primary key** in the parent table, not just the primary key

```sql
CREATE TABLE Orders (
    OrderId    INT IDENTITY(1,1) PRIMARY KEY,
    CustomerId INT NOT NULL,
    OrderDate  DATETIME NOT NULL,
    CONSTRAINT FK_Orders_Customers
        FOREIGN KEY (CustomerId)
        REFERENCES Customers(CustomerId)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

- **ON DELETE CASCADE** — automatically deletes child rows when the parent row is deleted
- **ON DELETE SET NULL** — sets the foreign key to NULL when the parent row is deleted
- **ON DELETE NO ACTION** — prevents deletion of the parent row (default behavior)
- **ON DELETE RESTRICT** — same as NO ACTION but checked at statement level

### Unique Key

- A **unique key** ensures all values in a column (or set of columns) are distinct across the table
- Unlike a primary key, a unique key **allows one NULL value** (SQL Server allows multiple NULLs in a unique column since NULL != NULL)
- A table can have **multiple** unique keys
- SQL Server automatically creates a **non-clustered index** for each unique key
- Use unique keys for columns that must be unique but are not the main identifier (e.g., email, SSN, username)

```sql
CREATE TABLE Users (
    UserId    INT IDENTITY(1,1) PRIMARY KEY,
    Username  NVARCHAR(50)  NOT NULL UNIQUE,
    Email     NVARCHAR(100) NOT NULL UNIQUE,
    Phone     NVARCHAR(20)  NULL UNIQUE
);
```

### Composite Key

- A **composite key** is a primary key (or unique key) composed of two or more columns
- The combination of columns must be unique, but individual columns can contain duplicate values
- Common in **junction tables** (many-to-many relationships)

```sql
-- Junction table for many-to-many relationship
CREATE TABLE StudentCourses (
    StudentId INT NOT NULL,
    CourseId  INT NOT NULL,
    EnrolledOn DATE NOT NULL,
    PRIMARY KEY (StudentId, CourseId),
    FOREIGN KEY (StudentId) REFERENCES Students(StudentId),
    FOREIGN KEY (CourseId)  REFERENCES Courses(CourseId)
);
```

### Normalization

- **Normalization** is the process of organizing a database to reduce data redundancy and improve data integrity
- It involves decomposing tables into smaller, well-structured tables and defining relationships between them
- The goal is to isolate data so that additions, deletions, and modifications of a field can be made in just one table and propagated through the rest of the database via relationships

#### First Normal Form (1NF)

- Each column must contain **atomic values** — no repeating groups, no arrays, no comma-separated lists
- Each row must be unique (enforced by a primary key)
- Each column must contain values of a single type

```sql
-- VIOLATES 1NF: comma-separated hobbies
-- | UserId | Name  | Hobbies          |
-- | 1      | Alice | reading, coding  |

-- FIX: Create separate table
CREATE TABLE UserHobbies (
    UserId INT NOT NULL,
    Hobby  NVARCHAR(50) NOT NULL,
    PRIMARY KEY (UserId, Hobby),
    FOREIGN KEY (UserId) REFERENCES Users(UserId)
);
```

#### Second Normal Form (2NF)

- Must already be in **1NF**
- Every non-key column must be **fully functionally dependent** on the entire primary key
- Eliminates **partial dependencies** — where a non-key column depends on only part of a composite key

```sql
-- VIOLATES 2NF: DepartmentName depends only on DepartmentId, not on (OrderId, DepartmentId)
-- | OrderId | DepartmentId | DepartmentName |
-- | 1       | 10          | Engineering    |

-- FIX: Split into two tables
CREATE TABLE Orders (
    OrderId      INT PRIMARY KEY,
    DepartmentId INT NOT NULL
);

CREATE TABLE Departments (
    DepartmentId   INT PRIMARY KEY,
    DepartmentName NVARCHAR(100) NOT NULL
);
```

#### Third Normal Form (3NF)

- Must already be in **2NF**
- No non-key column should depend on another non-key column
- Eliminates **transitive dependencies** — where A → B → C, and C should be in its own table

```sql
-- VIOLATES 3NF: City depends on ZipCode, not directly on EmployeeId
-- | EmployeeId | ZipCode | City      |
-- | 1          | 90210   | Beverly Hills |

-- FIX: Move city to a ZipCode lookup table
CREATE TABLE ZipCodes (
    ZipCode VARCHAR(10) PRIMARY KEY,
    City    NVARCHAR(100) NOT NULL,
    State   NVARCHAR(50) NOT NULL
);

CREATE TABLE Employees (
    EmployeeId INT PRIMARY KEY,
    ZipCode    VARCHAR(10) NULL,
    FOREIGN KEY (ZipCode) REFERENCES ZipCodes(ZipCode)
);
```

#### Why Normalization Matters

- **Eliminates data redundancy** — the same data is stored in only one place, reducing storage waste
- **Prevents update anomalies** — updating a value in one place automatically reflects everywhere through relationships
- **Prevents insert anomalies** — you can add data to one table without requiring unrelated data in another
- **Prevents delete anomalies** — deleting a row does not accidentally destroy unrelated data
- **Improves data integrity** — constraints and relationships enforce valid data at the database level
- When performance requires it, **selective denormalization** is acceptable — but always start normalized

---

## Indexes

### What is an Index?

- An **index** is a data structure that provides fast random access to rows in a table based on the values of one or more columns
- Without an index, SQL Server performs a **full table scan** — reading every single row to find matches, which is O(n) complexity
- With an index, SQL Server can find rows using a **B-Tree traversal** in O(log n) time
- Indexes are similar to the index at the back of a book — instead of reading every page, you look up a term and jump directly to the right page
- SQL Server uses **B+ Trees** as the default index structure — leaf nodes are linked for efficient range scans
- The primary trade-off: indexes **speed up reads** but **slow down writes** (INSERT, UPDATE, DELETE) and consume storage

### Clustered Index

- A **clustered index** determines the **physical order** of data on disk — the table data IS the clustered index (called the "clustered leaf level")
- A table can have **only one** clustered index because data can only be physically sorted one way
- If you create a primary key, SQL Server automatically creates the clustered index on it (by default)
- The clustered index leaf nodes contain the **actual data rows** — this is why it's also called the "data page"
- Best candidates for clustered indexes: **auto-incrementing keys**, **unique identifiers**, or columns frequently used in **range queries**

```sql
-- Clustered index is created automatically with PRIMARY KEY
CREATE TABLE Products (
    ProductId   INT IDENTITY(1,1) PRIMARY KEY,  -- clustered index
    ProductName NVARCHAR(200) NOT NULL,
    Price       DECIMAL(10,2) NOT NULL
);

-- Explicitly creating a clustered index on a different column
CREATE CLUSTERED INDEX IX_Products_Price ON Products(Price);

-- To change the clustered index, you must drop the existing one first
-- SQL Server enforces exactly ONE clustered index per table
```

- **Key property**: A table without a clustered index is called a **heap** — data is stored in unordered pages with Row IDs (RIDs) as pointers
- Clustered indexes are especially efficient for **range queries** (`BETWEEN`, `>`, `<`, `ORDER BY`) because rows are physically adjacent

### Non-Clustered Index

- A **non-clustered index** is a separate structure from the data rows — it contains the indexed column values plus a **pointer** back to the data row
- A table can have up to **999 non-clustered indexes** (SQL Server 2008+) or **249** in earlier versions
- Think of it like a book's index at the back — the index entries are separate from the actual chapter content
- The pointer in a non-clustered index is either the **clustered index key** (if the table has one) or the **RID** (Row Identifier, if the table is a heap)
- Non-clustered indexes are best for columns frequently used in **WHERE**, **JOIN**, and **ORDER BY** clauses

```sql
-- Non-clustered index on Email for fast lookups
CREATE NONCLUSTERED INDEX IX_Customers_Email ON Customers(Email);

-- Non-clustered index with multiple columns
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_OrderDate
    ON Orders(CustomerId, OrderDate);

-- Using the index: this query benefits from IX_Customers_Email
SELECT * FROM Customers WHERE Email = 'alice@example.com';
```

- **Key difference from clustered**: The non-clustered index leaf node does NOT contain the full row — it contains the index key + a pointer, which means SQL Server may need a second lookup to retrieve additional columns

### Composite Index

- A **composite index** is an index on **two or more columns** — the columns are combined into a single key
- Follows the **leftmost prefix rule**: an index on `(A, B, C)` can serve queries filtering on `A` alone, `A AND B`, or `A AND B AND C`
- It **cannot** be used for queries filtering on `B` alone or `C` alone — the leading column must be included
- Column order in a composite index matters significantly

```sql
-- Composite index: CustomerId first, then OrderDate
CREATE NONCLUSTERED INDEX IX_Orders_Composite
    ON Orders(CustomerId, OrderDate, Total);

-- This query CAN use the index (starts with CustomerId)
SELECT * FROM Orders WHERE CustomerId = 42;

-- This query CAN use the index (starts with CustomerId AND OrderDate)
SELECT * FROM Orders WHERE CustomerId = 42 AND OrderDate > '2024-01-01';

-- This query CANNOT use the index effectively (skips CustomerId)
SELECT * FROM Orders WHERE OrderDate > '2024-01-01';
```

- **Column order strategy**: Put **equality conditions first**, then **range conditions**, then **sort columns**
- The more columns in a composite index, the more query patterns it can serve — but the more expensive it is to maintain

### Covering Index

- A **covering index** includes all the columns a query needs **within the index itself**, so SQL Server never needs to look up the actual data row
- This eliminates the **Key Lookup** operation (the expensive second step of fetching the actual row)
- Use the `INCLUDE` clause to add non-key columns to the leaf level of the index — these columns don't affect sort order but are stored with the index

```sql
-- Query needs CustomerId, OrderDate, and Total
SELECT CustomerId, OrderDate, Total FROM Orders WHERE CustomerId = 42;

-- Covering index: CustomerId and OrderDate are key columns, Total is included
CREATE NONCLUSTERED INDEX IX_Orders_Covering
    ON Orders(CustomerId, OrderDate)
    INCLUDE (Total);

-- This query now uses only the index (Index Seek) — no Key Lookup needed
```

- **Why covering indexes matter**: They can reduce I/O from two operations (index seek + key lookup) down to one (index seek only)
- A covering index is one of the most impactful optimizations for frequently executed queries

### Filtered Index

- A **filtered index** is a non-clustered index with a `WHERE` clause that includes only a subset of rows
- Significantly smaller than a full index when only a small percentage of rows need to be indexed
- Excellent for queries that consistently filter on a specific value (e.g., active orders, pending tasks)

```sql
-- Only index pending orders (typically 1-2% of all orders)
CREATE NONCLUSTERED INDEX IX_Orders_Pending
    ON Orders(OrderId, OrderDate)
    WHERE Status = 'PENDING';

-- Only index active users
CREATE NONCLUSTERED INDEX IX_Users_Active
    ON Users(Email, FirstName, LastName)
    WHERE IsActive = 1;

-- This query benefits from the filtered index
SELECT OrderId, OrderDate FROM Orders WHERE Status = 'PENDING' AND OrderDate > '2024-01-01';
```

- Filtered indexes have **lower maintenance cost** — fewer rows to update when the table changes
- They can have **different filter predicates** on the same column — multiple filtered indexes on `Status` with different values

### When to Create Indexes

- **Columns in WHERE clauses** — the most common reason; if you filter on a column frequently, index it
- **Columns in JOIN conditions** — foreign keys used in JOINs should always be indexed
- **Columns in ORDER BY** — an index on the sort column avoids an expensive explicit Sort operator
- **High-selectivity columns** — columns with many unique values benefit most from indexing
- **Covering indexes** — when a query always selects the same few columns, create a covering index to avoid lookups
- **Foreign keys** — SQL Server does NOT automatically create indexes on foreign keys; you must do it manually

```sql
-- Always index foreign keys
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId ON Orders(CustomerId);
CREATE NONCLUSTERED INDEX IX_OrderItems_OrderId ON OrderItems(OrderId);
CREATE NONCLUSTERED INDEX IX_OrderItems_ProductId ON OrderItems(ProductId);
```

### Disadvantages of Indexes

- **Slower writes** — every INSERT must update all indexes; every UPDATE on an indexed column requires a delete + insert in the index
- **Storage overhead** — each index consumes disk space, sometimes as much as or more than the table itself
- **Index maintenance** — indexes can become fragmented over time, requiring periodic rebuilds or reorganizations
- **Memory pressure** — more indexes mean more pages to cache in buffer pool; if indexes don't fit in memory, performance degrades
- **Optimizer confusion** — too many indexes can confuse the query optimizer, leading to suboptimal plan selection
- **Plan cache bloat** — more indexes mean more possible access paths, which can bloat the plan cache

### Index Fragmentation and How to Fix It

- **External fragmentation** — the logical order of index pages does not match the physical order on disk, causing additional I/O for range scans
- **Internal fragmentation** — index pages have excessive free space (low page density), wasting I/O and memory
- Fragmentation occurs naturally as rows are inserted, updated, and deleted over time

```sql
-- Check fragmentation levels for all indexes on a table
SELECT
    i.name AS IndexName,
    ips.avg_fragmentation_in_percent,
    ips.avg_page_space_used_in_percent,
    ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), OBJECT_ID('Orders'), NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 5;

-- REORGANIZE: light maintenance, online, defragments at the leaf level only
ALTER INDEX IX_Orders_CustomerId ON Orders REORGANIZE;

-- REBUILD: heavy maintenance, defragments completely, updates statistics
-- ONLINE rebuild (Enterprise edition): does not lock the table
ALTER INDEX IX_Orders_CustomerId ON Orders REBUILD WITH (ONLINE = ON);

-- OFFLINE rebuild (Standard edition): locks the table during rebuild
ALTER INDEX IX_Orders_CustomerId ON Orders REBUILD;

-- Rebuild ALL indexes on a table
ALTER INDEX ALL ON Orders REBUILD;

-- General rule of thumb:
-- 5-30% fragmentation  → REORGANIZE
-- > 30% fragmentation  → REBUILD
```

### Missing Index DMVs

- SQL Server tracks queries that would benefit from indexes but do not have them
- The **missing index DMVs** (Dynamic Management Views) provide index recommendations based on actual workload
- These are starting points, not gospel — always evaluate the recommendation before implementing

```sql
-- Top 10 missing indexes by estimated improvement
SELECT TOP 10
    CONVERT(DECIMAL(18,2), migs.avg_total_user_cost *
            migs.avg_user_impact *
            (migs.user_seeks + migs.user_scans)) AS ImprovementMeasure,
    mid.statement AS TableName,
    mid.equality_columns AS EqualityColumns,
    mid.inequality_columns AS InequalityColumns,
    mid.included_columns AS IncludedColumns,
    migs.user_seeks,
    migs.user_scans,
    migs.last_user_seek
FROM sys.dm_db_missing_index_groups mig
JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY ImprovementMeasure DESC;
```

### Over-Indexing Problem

- Having **too many indexes** on a single table is a real problem that degrades overall performance
- Every index must be updated during INSERT, UPDATE, and DELETE operations
- A table with 15 indexes means every INSERT must update 15 index structures — this adds up fast in high-throughput systems
- Too many indexes cause **buffer pool pollution** — the cache fills with index pages instead of data pages
- The query optimizer has more choices to evaluate, which increases compile time and can lead to suboptimal plans
- **Rule of thumb**: If a table has more than 10-15 indexes, audit which ones are actually used and drop the unused ones

```sql
-- Find indexes that are never used
SELECT
    o.name AS TableName,
    i.name AS IndexName,
    i.type_desc,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates
FROM sys.indexes i
JOIN sys.objects o ON i.object_id = o.object_id
LEFT JOIN sys.dm_db_index_usage_stats ius
    ON i.object_id = ius.object_id AND i.index_id = ius.index_id
WHERE o.type = 'U'
    AND i.type_desc <> 'HEAP'
    AND (ius.user_seeks = 0 OR ius.user_seeks IS NULL)
    AND (ius.user_scans = 0 OR ius.user_scans IS NULL)
    AND (ius.user_lookups = 0 OR ius.user_lookups IS NULL)
ORDER BY ius.user_updates DESC;
```

---

## Execution Plans

### What is an Execution Plan?

- An **execution plan** (query plan) is the step-by-step strategy SQL Server's query optimizer generates to execute a SQL statement
- The optimizer evaluates multiple candidate plans, estimates their cost, and selects the one with the **lowest estimated cost**
- Understanding execution plans is the single most important skill for diagnosing and fixing slow queries
- Execution plans show you **what SQL Server actually did**, not what you wrote — this is critical for understanding performance

### How to View Execution Plans

```sql
-- Method 1: Actual execution plan (includes actual row counts and costs)
SET STATISTICS PROFILE ON;
SELECT * FROM Orders WHERE CustomerId = 42;
SET STATISTICS PROFILE OFF;

-- Method 2: Estimated execution plan (no execution, just the plan)
SET SHOWPLAN_ALL ON;
SELECT * FROM Orders WHERE CustomerId = 42;
SET SHOWPLAN_ALL OFF;

-- Method 3: XML execution plan
SET SHOWPLAN_XML ON;
SELECT * FROM Orders WHERE CustomerId = 42;
SET SHOWPLAN_XML OFF;

-- Method 4: Include actual execution plan in SSMS
-- Press Ctrl+M or click "Include Actual Execution Plan" button, then run query

-- Method 5: Query Store (persistent plan history)
-- Enabled at the database level
ALTER DATABASE MyDatabase SET QUERY_STORE = ON;
```

### Table Scan vs Index Scan vs Index Seek

- **Table Scan** (Clustered Index Scan on a table with clustered index) — SQL Server reads every row in the table from beginning to end. This is the worst-case access method for large tables. Happens when there is no useful index or the optimizer estimates a large percentage of rows will be returned.
- **Index Scan** (Non-Clustered Index Scan) — SQL Server reads every entry in a non-clustered index. If the query needs columns not in the index, it must do a Key Lookup for each row. Better than a Table Scan for narrow indexes, but still reads the entire index.
- **Index Seek** — SQL Server traverses the B-Tree from root to leaf, directly locating the rows that match the filter condition. This is the **most efficient** access method — it reads only the rows it needs. Requires a useful index on the filtered column.

```sql
-- This query causes a Table/Clustered Index Scan (no useful index on Status)
SELECT * FROM Orders WHERE Status = 'PENDING';

-- After creating an index on Status, this query causes an Index Seek
CREATE NONCLUSTERED INDEX IX_Orders_Status ON Orders(Status);
SELECT * FROM Orders WHERE Status = 'PENDING';
```

### Key Lookup vs RID Lookup

- **Key Lookup** — occurs when a non-clustered index does not contain all the columns needed by the query, so SQL Server must look up the actual row in the clustered index using the clustered key. This is a random I/O operation for each row — expensive in large result sets.
- **RID Lookup** — same concept but for a **heap table** (table without a clustered index). SQL Server uses the Row Identifier (RID) to look up the actual row in the heap. Generally less efficient than Key Lookup because RID lookups cannot leverage the clustered index structure.
- **How to eliminate lookups**: Create a **covering index** with the `INCLUDE` clause to add the needed columns to the index leaf level

```sql
-- This query may need a Key Lookup to fetch Total
SELECT CustomerId, OrderDate, Total FROM Orders WHERE CustomerId = 42;

-- Create a covering index to eliminate the Key Lookup
CREATE NONCLUSTERED INDEX IX_Orders_Covering
    ON Orders(CustomerId, OrderDate)
    INCLUDE (Total);
```

### Hash Join vs Merge Join vs Nested Loop

- **Nested Loop** — for each row in the outer (smaller) input, scans the inner input for matches. Best when the outer input is small and the inner input has an efficient index. O(n × m) worst case, but very fast when the outer input is tiny (e.g., 10 rows × index seek = fast).
- **Hash Join** — builds a hash table on the smaller input, then probes it with the larger input. Best for large, unsorted datasets where neither input has a useful index. Can spill to disk if the hash table exceeds memory grant. Good for equi-joins on large tables.
- **Merge Join** — both inputs must be sorted on the join key. Advances through both inputs simultaneously, matching rows. Very efficient when both inputs are pre-sorted (e.g., both have clustered indexes on the join key). If inputs are not sorted, an explicit Sort operator is added, which can be expensive.

```sql
-- Nested Loop: when CustomerId is the primary key (small outer, indexed inner)
SELECT c.Name, o.OrderDate
FROM Customers c
JOIN Orders o ON c.CustomerId = o.CustomerId;

-- Hash Join: when both tables are large and unsorted
SELECT o.OrderId, p.ProductName
FROM Orders o
JOIN OrderItems oi ON o.OrderId = oi.OrderId
JOIN Products p ON oi.ProductId = p.ProductId;

-- Merge Join: when both tables are sorted on the join key
-- (both have clustered indexes on the join column)
SELECT c.Name, o.OrderDate
FROM Customers c  -- clustered index on CustomerId
JOIN Orders o ON c.CustomerId = o.CustomerId  -- clustered index on CustomerId
ORDER BY c.CustomerId;
```

### Cost Percentage in Execution Plan

- Each operator in the execution plan has a **cost percentage** representing its share of the total query cost
- The cost is an **estimate**, not actual runtime — it is the optimizer's prediction based on statistics
- Focus on the **most expensive operators** (highest cost %) for optimization opportunities
- A single operator consuming >50% of the total cost is usually the bottleneck

### How to Read a Query Plan

- Plans are displayed as **trees** — read **right-to-left** (inner to outer operations) and **top-to-bottom** for execution order
- **Leaf nodes** (rightmost) are table/index access operations
- **Internal nodes** are joins, sorts, aggregations, and other processing operations
- The **root node** (leftmost) is the final result returned to the client
- Data flows **right to left** — each node's output becomes input to the node on its left
- Look for: **Table Scans** (should be Index Seeks), **Key Lookups** (should be covered), **Sorts** (should use indexes), **Hash Matches** (check if better join type exists)

---

## Stored Procedures

### What is a Stored Procedure?

- A **stored procedure** is a precompiled collection of SQL statements stored in the database with a name
- Think of it as a function in the database — it can accept parameters, perform operations, and return results
- Stored procedures are compiled and optimized once, then the **execution plan is cached** and reused for subsequent calls
- They live in the database, not in application code, which means changes to a stored procedure do not require a code deployment
- A stored procedure can call other stored procedures, use temporary tables, cursors, variables, and control flow

### Benefits of Stored Procedures

- **Performance** — execution plan is cached after the first call, so subsequent executions skip compilation and optimization. This is significant for complex queries executed thousands of times per minute.
- **Security** — users can be granted `EXECUTE` permission on a stored procedure without direct access to the underlying tables. Parameters are naturally handled, preventing SQL injection when used correctly.
- **Reusability** — the same logic can be called from multiple applications (web, desktop, jobs) without duplication. Centralizes business logic in the database layer.
- **Network efficiency** — instead of sending a large SQL batch over the network, the application sends a single `EXEC` or `CALL` statement with parameters.
- **Maintainability** — database changes can be made inside the stored procedure without changing application code (within reason).

### CREATE PROCEDURE Syntax

```sql
-- Basic stored procedure
CREATE PROCEDURE usp_GetOrdersByCustomer
    @CustomerId INT,
    @StartDate  DATETIME = NULL,  -- optional parameter with default
    @EndDate    DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;  -- suppress "(x rows affected)" message

    SELECT OrderId, OrderDate, Total
    FROM Orders
    WHERE CustomerId = @CustomerId
        AND (@StartDate IS NULL OR OrderDate >= @StartDate)
        AND (@EndDate IS NULL OR OrderDate <= @EndDate)
    ORDER BY OrderDate DESC;
END;
GO

-- Execute it
EXEC usp_GetOrdersByCustomer @CustomerId = 42;
EXEC usp_GetOrdersByCustomer @CustomerId = 42, @StartDate = '2024-01-01';
```

### Input Parameters, Output Parameters, and Return Values

```sql
CREATE PROCEDURE usp_GetOrderSummary
    -- Input parameters
    @CustomerId INT,
    @MinTotal   DECIMAL(10,2) = 0,

    -- Output parameters
    @OrderCount    INT OUTPUT,
    @TotalAmount   DECIMAL(12,2) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        @OrderCount = COUNT(*),
        @TotalAmount = ISNULL(SUM(Total), 0)
    FROM Orders
    WHERE CustomerId = @CustomerId
        AND Total >= @MinTotal;

    -- Return value (convention: 0 = success, non-zero = error)
    RETURN 0;
END;
GO

-- Calling with output parameters
DECLARE @Count INT, @Total DECIMAL(12,2);
EXEC usp_GetOrderSummary
    @CustomerId = 42,
    @MinTotal = 100,
    @OrderCount = @Count OUTPUT,
    @TotalAmount = @Total OUTPUT;
SELECT @Count AS OrderCount, @Total AS TotalAmount;

-- Check return value
DECLARE @ReturnValue INT;
EXEC @ReturnValue = usp_GetOrderSummary @CustomerId = 42;
SELECT @ReturnValue AS ReturnValue;  -- 0 = success
```

- **Input parameters** (default) — values passed into the procedure
- **Output parameters** (`OUTPUT` keyword) — values returned from the procedure to the caller
- **Return values** — integer status code; conventionally 0 for success, non-zero for error; distinct from output parameters

### Variables and Control Flow

```sql
CREATE PROCEDURE usp_ProcessOrder
    @OrderId INT
AS
BEGIN
    SET NOCOUNT ON;

    -- Variable declaration
    DECLARE @Total DECIMAL(10,2);
    DECLARE @Status NVARCHAR(20);
    DECLARE @CustomerTier NVARCHAR(20);

    -- Initialize variables
    SELECT @Total = Total, @Status = Status
    FROM Orders WHERE OrderId = @OrderId;

    -- IF/ELSE control flow
    IF @Status = 'COMPLETED'
    BEGIN
        PRINT 'Order already processed';
        RETURN;
    END;

    IF @Total > 1000
        SET @CustomerTier = 'GOLD';
    ELSE IF @Total > 500
        SET @CustomerTier = 'SILVER';
    ELSE
        SET @CustomerTier = 'BRONZE';

    -- WHILE loop
    DECLARE @RetryCount INT = 0;
    WHILE @RetryCount < 3
    BEGIN
        BEGIN TRY
            -- Process order logic
            UPDATE Orders SET Status = 'PROCESSING' WHERE OrderId = @OrderId;
            BREAK;  -- exit loop on success
        END TRY
        BEGIN CATCH
            SET @RetryCount = @RetryCount + 1;
            IF @RetryCount >= 3
                THROW;
        END CATCH
    END;

    PRINT 'Order processed with tier: ' + @CustomerTier;
END;
GO
```

### Error Handling in Stored Procedures

```sql
CREATE PROCEDURE usp_TransferFunds
    @FromAccount INT,
    @ToAccount   INT,
    @Amount      DECIMAL(10,2)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Validate inputs
        IF @Amount <= 0
            THROW 50001, 'Amount must be positive', 1;

        -- Check sufficient balance
        DECLARE @Balance DECIMAL(10,2);
        SELECT @Balance = Balance FROM Accounts WHERE AccountId = @FromAccount;

        IF @Balance < @Amount
            THROW 50002, 'Insufficient funds', 1;

        -- Perform transfer
        UPDATE Accounts SET Balance = Balance - @Amount WHERE AccountId = @FromAccount;
        UPDATE Accounts SET Balance = Balance + @Amount WHERE AccountId = @ToAccount;

        COMMIT TRANSACTION;
        PRINT 'Transfer successful';
    END TRY
    BEGIN CATCH
        -- Rollback if transaction is open
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        -- Log error details
        INSERT INTO ErrorLog (ErrorMessage, ErrorNumber, ErrorSeverity, ErrorState, ErrorLine, ProcedureName)
        VALUES (
            ERROR_MESSAGE(),
            ERROR_NUMBER(),
            ERROR_SEVERITY(),
            ERROR_STATE(),
            ERROR_LINE(),
            ERROR_PROCEDURE()
        );

        -- Re-throw the error to the caller
        THROW;
    END CATCH;
END;
GO
```

### How to Optimize Slow Stored Procedures

- **Check the execution plan** first — always start by looking at the actual execution plan for the slow statement inside the procedure
- **Update statistics** — stale statistics cause the optimizer to make poor cardinality estimates
- **Look for table scans** — if a table scan is occurring on a large table, add an appropriate index
- **Watch for parameter sniffing** — a cached plan optimized for one parameter value may be terrible for another
- **Use SET NOCOUNT ON** — prevents the row count message from being sent back, reducing network traffic
- **Minimize result sets** — return only the columns and rows the application actually needs
- **Avoid cursors** — set-based operations are almost always faster than row-by-row processing

### Execution Plan Caching and Parameter Sniffing

- SQL Server **caches the execution plan** after the first call and reuses it for all subsequent calls
- **Parameter sniffing** occurs when the cached plan is optimized for the first parameter values, which may not be representative of typical values
- Example: the first call uses `@CustomerId = 1` (a customer with 2 orders), and the plan is optimized for a small result. Later calls use `@CustomerId = 42` (a customer with 50,000 orders) — the plan is suboptimal.

```sql
-- Force recompilation to get a fresh plan for each call
CREATE PROCEDURE usp_GetOrders
    @CustomerId INT
WITH RECOMPILE  -- recompiles every time (use sparingly)
AS
BEGIN
    SELECT * FROM Orders WHERE CustomerId = @CustomerId;
END;

-- Or use OPTION (RECOMPILE) on specific statements
CREATE PROCEDURE usp_GetOrders_Flexible
    @CustomerId INT
AS
BEGIN
    SELECT * FROM Orders WHERE CustomerId = @CustomerId
    OPTION (RECOMPILE);  -- recompile just this statement
END;
```

---

## Transactions and ACID

### What is a Transaction?

- A **transaction** is a logical unit of work that groups one or more SQL statements into a single atomic operation
- Either **all statements succeed** (COMMIT) or **all are undone** (ROLLBACK) — there is no partial completion
- Transactions are essential for maintaining data consistency when multiple operations must succeed or fail together

### ACID Properties

- **Atomicity** — all operations in the transaction either complete successfully as a group, or none of them are applied. If any statement fails, the entire transaction is rolled back, leaving the database in its pre-transaction state.
- **Consistency** — a transaction takes the database from one valid state to another valid state. All constraints, rules, and triggers must be satisfied before and after the transaction. If a constraint would be violated, the transaction is rolled back.
- **Isolation** — concurrent transactions execute as if they were running sequentially. One transaction's intermediate state is not visible to other transactions until it commits. Isolation levels (READ COMMITTED, REPEATABLE READ, SERIALIZABLE) define the degree of isolation.
- **Durability** — once a transaction is committed, its effects are permanent even if the system crashes immediately after. SQL Server achieves durability through write-ahead logging (WAL) — the log is flushed to disk before the data pages.

```sql
-- Transfer money: both updates must succeed or both must fail
BEGIN TRANSACTION;

UPDATE Accounts SET Balance = Balance - 500 WHERE AccountId = 1;
UPDATE Accounts SET Balance = Balance + 500 WHERE AccountId = 2;

-- Verify no constraint violations
IF @@ERROR = 0
    COMMIT TRANSACTION;
ELSE
    ROLLBACK TRANSACTION;
```

### TRY/CATCH with Transactions

```sql
BEGIN TRY
    BEGIN TRANSACTION;

    UPDATE Accounts SET Balance = Balance - 500 WHERE AccountId = 1;

    -- This will fail if AccountId = 2 does not exist
    UPDATE Accounts SET Balance = Balance + 500 WHERE AccountId = 2;

    COMMIT TRANSACTION;
    PRINT 'Transfer committed successfully';
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;

    PRINT 'Transfer failed: ' + ERROR_MESSAGE();
    -- Optionally re-throw
    THROW;
END CATCH;
```

### Nested Transactions and Savepoints

```sql
BEGIN TRANSACTION outer_transaction;

    UPDATE Accounts SET Balance = Balance - 100 WHERE AccountId = 1;

    SAVE TRANSACTION savepoint1;  -- create a savepoint

    UPDATE Accounts SET Balance = Balance + 100 WHERE AccountId = 2;

    -- If something goes wrong, we can rollback to savepoint instead of the whole transaction
    -- ROLLBACK TRANSACTION savepoint1;

    -- The outer COMMIT actually commits everything (including savepoint)
    COMMIT TRANSACTION outer_transaction;
```

- **Important**: SQL Server does **not truly support nested transactions** — `@@TRANCOUNT` increments on each `BEGIN TRANSACTION` and decrements on each `COMMIT`. Only the outermost `COMMIT` actually writes to the log. A `ROLLBACK` (without a savepoint name) rolls back the **entire transaction** regardless of nesting level.
- Use `SAVE TRANSACTION` (savepoints) when you need to roll back part of a transaction while keeping the rest.

---

## Isolation Levels

### READ UNCOMMITTED

- The lowest isolation level — reads data that has been modified by other transactions but **not yet committed**
- Allows **dirty reads** — you can read data that might be rolled back
- No shared locks are acquired, so reads never block writers and writers never block readers
- Useful for approximate counts or monitoring queries where stale/dirty data is acceptable
- Rarely used in production — the risk of reading nonexistent data is usually too high

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT COUNT(*) FROM Orders;  -- may include uncommitted (possibly rolled back) orders
```

### READ COMMITTED (Default)

- SQL Server's **default** isolation level
- Reads only **committed data** — dirty reads are prevented
- Non-repeatable reads and phantom reads are still possible
- Uses shared locks that are released after reading — does not hold locks for the duration of the transaction
- Most applications should use this as the baseline

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Transaction A reads a row → gets committed value
-- Transaction B updates the same row and commits
-- Transaction A reads the same row again → gets NEW committed value (non-repeatable read)
```

### REPEATABLE READ

- Guarantees that if you read a row twice in the same transaction, you get the **same value both times**
- Shared locks are held for the **duration of the transaction**, not just for the read
- Prevents dirty reads and non-repeatable reads
- Phantom reads are still possible — new rows can appear between reads
- Higher isolation means **lower concurrency** — holding locks longer blocks other transactions

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;
SELECT * FROM Products WHERE CategoryId = 1;  -- gets rows A, B, C
-- Another transaction inserts row D into CategoryId = 1 and commits
SELECT * FROM Products WHERE CategoryId = 1;  -- still gets A, B, C (no phantoms? Actually phantoms ARE possible in SQL Server REPEATABLE READ)
COMMIT;
```

### SERIALIZABLE

- The **highest** isolation level — transactions execute as if they ran one after another (serially)
- Prevents dirty reads, non-repeatable reads, **and** phantom reads
- Uses **range locks** that prevent other transactions from inserting rows within the range you queried
- Most restrictive — significantly reduces concurrency and can cause **deadlocks** and **timeout** errors
- Use only when strict consistency is required and the performance impact is acceptable

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;
SELECT * FROM Products WHERE CategoryId = 1;  -- locks the range
-- Another transaction tries to INSERT into CategoryId = 1 → BLOCKED
SELECT * FROM Products WHERE CategoryId = 1;  -- same rows, no phantoms
COMMIT;
```

### SNAPSHOT

- Uses **row versioning** — each transaction sees a consistent snapshot from the point it started
- Readers do **not** block writers, and writers do **not** block readers
- Requires `ALLOW_SNAPSHOT_ISOLATION ON` at the database level
- Prevents dirty reads, non-repeatable reads, and phantom reads
- Write-write conflicts are detected using a **first-committer-wins** strategy

```sql
ALTER DATABASE MyDatabase SET ALLOW_SNAPSHOT_ISOLATION ON;

SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;
SELECT * FROM Orders WHERE CustomerId = 42;  -- sees snapshot from transaction start
-- Other transactions can modify Orders freely
SELECT * FROM Orders WHERE CustomerId = 42;  -- same snapshot, same results
COMMIT;
```

### READ COMMITTED SNAPSHOT

- Similar to SNAPSHOT, but each **statement** sees a fresh snapshot (not the transaction start)
- This is the **recommended** replacement for the default READ COMMITTED in most modern applications
- Requires `READ_COMMITTED_SNAPSHOT ON` at the database level
- Eliminates reader-writer blocking while keeping the familiar READ COMMITTED semantics

```sql
ALTER DATABASE MyDatabase SET READ_COMMITTED_SNAPSHOT ON;

-- Now READ COMMITTED uses row versioning instead of locking
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Readers never block writers, writers never block readers
```

### Why Isolation Levels Matter for Concurrency

- Higher isolation → more consistency but **less concurrency** (more blocking, more deadlocks)
- Lower isolation → more concurrency but **less consistency** (dirty reads, phantom reads)
- The right isolation level depends on your application's tolerance for stale data vs. its concurrency requirements
- Most web applications work well with **READ COMMITTED SNAPSHOT** — good concurrency with reasonable consistency
- Financial systems may require **SERIALIZABLE** for critical operations to prevent anomalies

### Trade-off: Isolation vs Performance

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible | Highest |
| READ COMMITTED | No | Possible | Possible | High |
| REPEATABLE READ | No | No | Possible | Medium |
| SERIALIZABLE | No | No | No | Lowest |
| SNAPSHOT | No | No | No | High (row versioning) |

---

## Deadlocks

### What is a Deadlock?

- A **deadlock** is a situation where two or more transactions are **permanently blocked** — each waiting for the other to release a lock
- Transaction A holds a lock on Table1 and needs a lock on Table2
- Transaction B holds a lock on Table2 and needs a lock on Table1
- Neither can proceed — they are stuck forever without intervention
- SQL Server detects deadlocks automatically and **terminates one victim** transaction to break the cycle

### Why Deadlocks Happen

- **Inconsistent lock ordering** — the most common cause. Transaction A locks Table1 then Table2, while Transaction B locks Table2 then Table1.
- **Long-running transactions** — the longer a transaction holds locks, the more likely it is to deadlock with another transaction
- **Lock escalation** — SQL Server may escalate row locks to page or table locks, increasing the scope of blocking
- **Resource deadlocks** — less common, but two transactions can deadlock on non-table resources like memory, CPU, or tempdb space

### How to Prevent Deadlocks

- **Consistent lock ordering** — always access tables in the same order across all transactions and stored procedures. If every transaction locks Table1 before Table2, deadlocks between these tables are impossible.
- **Minimize lock duration** — keep transactions as short as possible. Do not perform user input, file I/O, or web service calls inside a transaction.
- **Use lower isolation levels** — READ COMMITTED SNAPSHOT reduces locking significantly. SERIALIZABLE increases lock scope and deadlock risk.
- **Use covering indexes** — reduce the number of locks needed by avoiding Key Lookups (which require additional locks on the clustered index).
- **Lock at the row level** — avoid table-level locks; use row-level granularity when possible.
- **Add indexes on foreign keys** — without indexes, SQL Server may use table locks for cascading operations.

```sql
-- BAD: inconsistent lock ordering
-- Transaction 1: UPDATE Accounts, then UPDATE Orders
-- Transaction 2: UPDATE Orders, then UPDATE Accounts

-- GOOD: consistent lock ordering
-- Both transactions: UPDATE Accounts first, then UPDATE Orders
BEGIN TRANSACTION;
UPDATE Accounts SET Balance = Balance - 100 WHERE AccountId = 1;
UPDATE Orders SET Status = 'PAID' WHERE OrderId = 42;
COMMIT;
```

### How SQL Server Detects Deadlocks

- SQL Server uses a background process called the **Lock Monitor** that periodically checks for deadlocks
- When a deadlock is detected, SQL Server selects a **victim** (usually the transaction with the lowest rollback cost) and terminates it with error **1205**
- The victim transaction is rolled back, releasing its locks and allowing the other transaction(s) to proceed
- The application receives error 1205 and can retry the transaction

### DEADLOCK_PRIORITY

```sql
-- Set deadlock priority for a session
SET DEADLOCK_PRIORITY LOW;    -- this session is more likely to be chosen as victim
SET DEADLOCK_PRIORITY NORMAL; -- default
SET DEADLOCK_PRIORITY HIGH;   -- this session is less likely to be chosen as victim

-- Or use numeric values: -10 (LOW) to 10 (HIGH)
SET DEADLOCK_PRIORITY 5;
```

### TRY/CATCH to Handle Deadlock Error 1205

```sql
CREATE PROCEDURE usp_TransferFunds_Safe
    @FromAccount INT,
    @ToAccount   INT,
    @Amount      DECIMAL(10,2)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @MaxRetries INT = 3;
    DECLARE @RetryCount INT = 0;

    WHILE @RetryCount < @MaxRetries
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION;

            UPDATE Accounts SET Balance = Balance - @Amount WHERE AccountId = @FromAccount;
            UPDATE Accounts SET Balance = Balance + @Amount WHERE AccountId = @ToAccount;

            COMMIT TRANSACTION;
            RETURN 0;  -- success
        END TRY
        BEGIN CATCH
            IF @@TRANCOUNT > 0
                ROLLBACK TRANSACTION;

            IF ERROR_NUMBER() = 1205  -- deadlock victim
            BEGIN
                SET @RetryCount = @RetryCount + 1;
                PRINT 'Deadlock detected, retry attempt ' + CAST(@RetryCount AS NVARCHAR(10));
                -- Optionally add a small delay before retry
                WAITFOR DELAY '00:00:00.100';  -- 100ms
            END
            ELSE
            BEGIN
                -- Non-deadlock error, re-throw
                THROW;
            END
        END CATCH
    END;

    -- If we get here, all retries failed
    THROW 50010, 'Transfer failed after maximum retries due to deadlocks', 1;
END;
GO
```

---

## Query Optimization

### Parameter Sniffing

- **Parameter sniffing** occurs when SQL Server compiles a query plan using the parameter values from the first execution, then reuses that plan for all subsequent executions
- If the first execution uses an atypical parameter (e.g., a customer with 2 orders), the plan is optimized for a small result set
- Later executions with typical parameters (e.g., a customer with 50,000 orders) use the same suboptimal plan
- This is one of the **most common causes of sudden performance degradation** in production

```sql
-- First execution: @CustomerId = 1 (small result, plan optimized for few rows)
EXEC usp_GetOrders @CustomerId = 1;

-- Later execution: @CustomerId = 42 (large result, but uses the plan for small results)
EXEC usp_GetOrders @CustomerId = 42;  -- suddenly slow!
```

### How to Handle Parameter Sniffing

```sql
-- Option 1: OPTIMIZE FOR a specific value
CREATE PROCEDURE usp_GetOrders_Optimized
    @CustomerId INT
AS
BEGIN
    SELECT * FROM Orders WHERE CustomerId = @CustomerId
    OPTION (OPTIMIZE FOR (@CustomerId = 42));  -- optimize for the "typical" case
END;

-- Option 2: OPTIMIZE FOR UNKNOWN — let SQL Server use average statistics
CREATE PROCEDURE usp_GetOrders_Average
    @CustomerId INT
AS
BEGIN
    SELECT * FROM Orders WHERE CustomerId = @CustomerId
    OPTION (OPTIMIZE FOR UNKNOWN);
END;

-- Option 3: Local variable trick — defeats parameter sniffing entirely
CREATE PROCEDURE usp_GetOrders_LocalVar
    @CustomerId INT
AS
BEGIN
    DECLARE @LocalCustomerId INT = @CustomerId;
    SELECT * FROM Orders WHERE CustomerId = @LocalCustomerId;
    -- SQL Server cannot sniff the local variable, so it uses average density estimates
END;

-- Option 4: RECOMPILE — generate a fresh plan every time (use sparingly)
CREATE PROCEDURE usp_GetOrders_Recompile
    @CustomerId INT
AS
BEGIN
    SELECT * FROM Orders WHERE CustomerId = @CustomerId
    OPTION (RECOMPILE);
END;
```

### Statistics

- **Statistics** are objects that contain information about the distribution of values in one or more columns
- SQL Server uses statistics to estimate **cardinality** (how many rows will match a filter) and choose the best execution plan
- SQL Server **auto-updates statistics** when about 20% of the table data changes (threshold varies by table size)
- Stale statistics lead to poor cardinality estimates, which lead to bad plan choices (e.g., choosing a Table Scan over an Index Seek)

```sql
-- View statistics for a table
SELECT name, auto_created, user_created, update_date
FROM sys.stats
WHERE object_id = OBJECT_ID('Orders');

-- Manually update statistics
UPDATE STATISTICS Orders;

-- Update a specific statistics object
UPDATE STATISTICS Orders IX_Orders_CustomerId;

-- Full scan statistics (more accurate but slower)
UPDATE STATISTICS Orders WITH FULLSCAN;

-- Check when statistics were last updated
SELECT
    s.name AS StatsName,
    sp.last_updated
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id = OBJECT_ID('Orders');
```

### When to Use Index Hints

- An **index hint** tells SQL Server to use a specific index instead of letting the optimizer choose
- Use index hints **very rarely** — they override the optimizer and can make performance worse if the data distribution changes
- Valid use cases: when you know the optimizer consistently picks the wrong index and you have validated the hint improves performance

```sql
-- Index hint syntax
SELECT * FROM Orders WITH (INDEX(IX_Orders_CustomerId))
WHERE CustomerId = 42;

-- Force a table scan
SELECT * FROM Orders WITH (TABLOCK, INDEX(0))
WHERE Status = 'PENDING';
```

### SET STATISTICS IO and TIME

```sql
-- Shows logical reads, physical reads, and read-ahead reads
SET STATISTICS IO ON;
SELECT * FROM Orders WHERE CustomerId = 42;
SET STATISTICS IO OFF;

-- Output example:
-- Table 'Orders'. Scan count 1, logical reads 12, physical reads 2, read-ahead reads 0

-- Shows CPU time and elapsed time
SET STATISTICS TIME ON;
SELECT * FROM Orders WHERE CustomerId = 42;
SET STATISTICS TIME OFF;

-- Output example:
-- SQL Server Execution Times: CPU time = 15 ms, elapsed time = 23 ms
```

- **Logical reads** — pages read from the buffer pool (memory). This is the most important metric. Fewer logical reads = less work.
- **Physical reads** — pages read from disk. Lower is better (should be near zero if data is cached).
- **Read-ahead reads** — pages fetched speculatively in anticipation of needing them.

### Common Table Expressions (CTE)

- A **CTE** (Common Table Expression) is a temporary named result set defined within a single `SELECT`, `INSERT`, `UPDATE`, or `DELETE` statement
- Improves readability of complex queries by breaking them into logical named blocks
- The CTE exists only for the duration of the single statement — it is not stored anywhere
- Essential for **recursive queries** (e.g., traversing hierarchical data)

```sql
-- Non-recursive CTE: simplify a complex query
WITH CustomerOrderStats AS (
    SELECT
        CustomerId,
        COUNT(*) AS OrderCount,
        SUM(Total) AS TotalSpent,
        MAX(OrderDate) AS LastOrderDate
    FROM Orders
    GROUP BY CustomerId
)
SELECT
    c.CustomerId,
    c.Name,
    cos.OrderCount,
    cos.TotalSpent,
    cos.LastOrderDate
FROM Customers c
JOIN CustomerOrderStats cos ON c.CustomerId = cos.CustomerId
WHERE cos.TotalSpent > 1000;

-- Recursive CTE: traverse an employee hierarchy
WITH EmployeeHierarchy AS (
    -- Anchor: top-level manager (no reports_to)
    SELECT EmployeeId, Name, ReportsTo, 0 AS Level
    FROM Employees
    WHERE ReportsTo IS NULL

    UNION ALL

    -- Recursive: employees who report to someone already in the result
    SELECT e.EmployeeId, e.Name, e.ReportsTo, eh.Level + 1
    FROM Employees e
    JOIN EmployeeHierarchy eh ON e.ReportsTo = eh.EmployeeId
)
SELECT * FROM EmployeeHierarchy
ORDER BY Level, Name
OPTION (MAXRECURSION 100);  -- prevent infinite loops
```

### Window Functions

- **Window functions** perform calculations across a set of rows related to the current row, without collapsing them into a single output row (unlike GROUP BY)
- Defined using the `OVER()` clause, which specifies the partition and ordering of the "window" of rows

```sql
-- ROW_NUMBER: assigns a unique sequential number to each row
SELECT
    OrderId,
    CustomerId,
    Total,
    ROW_NUMBER() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC) AS RowNum
FROM Orders;
-- Result: each customer's orders numbered 1, 2, 3, ... by most recent

-- RANK: assigns the same rank to ties, skips numbers after ties
SELECT
    ProductId,
    TotalSales,
    RANK() OVER (ORDER BY TotalSales DESC) AS SalesRank
FROM ProductSales;

-- DENSE_RANK: like RANK but does not skip numbers after ties
SELECT
    ProductId,
    TotalSales,
    DENSE_RANK() OVER (ORDER BY TotalSales DESC) AS DenseRank
FROM ProductSales;

-- LAG and LEAD: access previous and next rows
SELECT
    OrderDate,
    Total,
    LAG(Total, 1) OVER (ORDER BY OrderDate) AS PreviousOrderTotal,
    LEAD(Total, 1) OVER (ORDER BY OrderDate) AS NextOrderTotal,
    Total - LAG(Total, 1) OVER (ORDER BY OrderDate) AS ChangeFromPrevious
FROM Orders;

-- Running total
SELECT
    OrderDate,
    Total,
    SUM(Total) OVER (ORDER BY OrderDate ROWS UNBOUNDED PRECEDING) AS RunningTotal
FROM Orders;

-- Moving average (last 3 rows)
SELECT
    OrderDate,
    Total,
    AVG(Total) OVER (ORDER BY OrderDate ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS MovingAvg3
FROM Orders;
```

---

## SQL Injection

### What is SQL Injection?

- **SQL injection** is an attack where malicious SQL code is inserted into application queries through user input
- If user input is concatenated directly into a SQL string, an attacker can manipulate the query to access, modify, or delete data
- SQL injection is one of the **most dangerous and most common** web application vulnerabilities (consistently in OWASP Top 10)

```sql
-- Normal query
-- Input: customerId = "42"
SELECT * FROM Orders WHERE CustomerId = 42;

-- SQL injection attack
-- Input: customerId = "42; DROP TABLE Orders;--"
SELECT * FROM Orders WHERE CustomerId = 42; DROP TABLE Orders;--
-- The injected DROP TABLE destroys your data!
```

### How Parameterized Queries Prevent It

- **Parameterized queries** separate the SQL code from the data — the database never interprets parameter values as code
- The SQL statement is compiled first, then parameters are bound as data values — they cannot alter the query structure
- This is the **only reliable defense** against SQL injection

```sql
-- DANGEROUS: string concatenation (vulnerable to SQL injection)
string sql = "SELECT * FROM Orders WHERE CustomerId = " + customerId;
 SqlCommand cmd = new SqlCommand(sql, connection);

-- SAFE: parameterized query (SQL injection-proof)
string sql = "SELECT * FROM Orders WHERE CustomerId = @CustomerId";
SqlCommand cmd = new SqlCommand(sql, connection);
cmd.Parameters.AddWithValue("@CustomerId", customerId);
// Or even better, specify the type explicitly:
cmd.Parameters.Add("@CustomerId", SqlDbType.Int).Value = customerId;
```

### Why String Concatenation in SQL is Dangerous

- When you build SQL strings by concatenating user input, you are giving the user the ability to **rewrite your query**
- Even with input validation, concatenation is dangerous because it is impossible to escape all edge cases across all character sets
- Validation is a **defense-in-depth** measure, not a substitute for parameterization
- Stored procedures with `EXEC(@sql)` or `sp_executesql` with concatenated parameters are also vulnerable

```sql
-- DANGEROUS: dynamic SQL with concatenation
DECLARE @SearchTerm NVARCHAR(100) = N'%; DROP TABLE Users;--';
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM Products WHERE Name LIKE ''%' + @SearchTerm + '%''';
EXEC(@sql);  -- SQL injection!

-- SAFE: dynamic SQL with sp_executesql and parameters
DECLARE @SearchTerm NVARCHAR(100) = N'%laptop%';
DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM Products WHERE Name LIKE @SearchTerm';
EXEC sp_executesql @sql, N'@SearchTerm NVARCHAR(100)', @SearchTerm = @SearchTerm;
```

### Stored Procedures and SQL Injection

- Stored procedures **can** still be vulnerable to SQL injection if they use dynamic SQL with string concatenation
- The key is that stored procedures should use **parameterized queries** internally, not string concatenation

```sql
-- VULNERABLE stored procedure (dynamic SQL with concatenation)
CREATE PROCEDURE usp_SearchUsers
    @Username NVARCHAR(50)
AS
BEGIN
    DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM Users WHERE Username = ''' + @Username + '''';
    EXEC(@sql);  -- SQL injection possible!
END;

-- SAFE stored procedure (parameterized dynamic SQL)
CREATE PROCEDURE usp_SearchUsers_Safe
    @Username NVARCHAR(50)
AS
BEGIN
    DECLARE @sql NVARCHAR(MAX) = N'SELECT * FROM Users WHERE Username = @Username';
    EXEC sp_executesql @sql, N'@Username NVARCHAR(50)', @Username = @Username;
END;
```

### Dynamic SQL with sp_executesql

- `sp_executesql` is the **preferred** way to execute dynamic SQL in SQL Server
- It supports parameterization, which prevents SQL injection
- It also benefits from execution plan caching — the same dynamic SQL template with different parameters can reuse a cached plan

```sql
-- sp_executesql: safe, cached, and parameterized
DECLARE @sql NVARCHAR(MAX) = N'
    SELECT OrderId, Total
    FROM Orders
    WHERE CustomerId = @CustomerId
        AND OrderDate >= @StartDate
    ORDER BY OrderDate DESC';

DECLARE @CustomerId INT = 42;
DECLARE @StartDate DATETIME = '2024-01-01';

EXEC sp_executesql @sql,
    N'@CustomerId INT, @StartDate DATETIME',
    @CustomerId = @CustomerId,
    @StartDate = @StartDate;
```

---

## Connection Pooling

### What is Connection Pooling?

- **Connection pooling** is the practice of maintaining a cache of open database connections that can be reused across multiple application requests
- Creating a new TCP connection to SQL Server is expensive — it involves TCP handshake, TDS protocol negotiation, authentication, and session setup (typically 50-200ms)
- Connection pooling eliminates this overhead by keeping connections open and handing them out to new requests
- When an application "closes" a connection, it is **returned to the pool** rather than actually closed — the TCP connection stays alive for the next request

### How ADO.NET Connection Pooling Works

- ADO.NET manages connection pools **per connection string** — each unique connection string gets its own pool
- When a connection is opened, ADO.NET checks if a connection is available in the pool. If yes, it is reused. If no, a new connection is created (up to the pool's max size).
- When a connection is closed/disposed, it is returned to the pool with its transaction context reset
- The pool maintains **minimum** and **maximum** connections and periodically cleans up idle connections

```csharp
// Connection string with pooling parameters
string connectionString = "Server=myserver;Database=mydb;Trusted_Connection=true;" +
    "Max Pool Size=100;" +       // max connections in pool (default 100)
    "Min Pool Size=10;" +        // min connections kept alive (default 0)
    "Connection Timeout=30;" +   // seconds to wait for a connection
    "Connection Lifetime=300";   // seconds before a connection is removed from pool

// This reuses a pooled connection — very fast
using (SqlConnection conn = new SqlConnection(connectionString))
{
    conn.Open();  // gets a connection from the pool (fast)
    // ... use connection ...
}  // returns connection to the pool (does NOT close TCP connection)
```

### Connection String Parameters

- **Max Pool Size** — maximum number of connections in the pool (default: 100). When the pool is exhausted, new connection requests wait until a connection becomes available or the timeout expires.
- **Min Pool Size** — minimum number of connections maintained in the pool (default: 0). Setting this higher ensures connections are always available but uses more server resources.
- **Connection Lifetime** — maximum lifetime in seconds for a connection in the pool (default: infinite). After this time, the connection is closed and removed. Useful for load-balanced scenarios.
- **Pooling** — set to `false` to disable pooling entirely (default: `true`). Never disable pooling in production.
- **Connection Timeout** — seconds to wait for a connection to become available before throwing an exception (default: 15 seconds).

### Pool Exhaustion and How to Handle It

- **Pool exhaustion** occurs when all connections in the pool are in use and new requests must wait
- Common causes: long-running queries holding connections, not disposing connections properly, too many concurrent requests
- Symptoms: `Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool`

```csharp
// BAD: connection not disposed on exception
SqlConnection conn = new SqlConnection(connectionString);
conn.Open();
// If an exception occurs here, the connection leaks!
ExecuteQuery(conn);
conn.Close();

// GOOD: using statement ensures disposal
using (SqlConnection conn = new SqlConnection(connectionString))
{
    conn.Open();
    ExecuteQuery(conn);
}  // always returns connection to pool, even on exception
```

### Why Closing/Disposing Connections Returns Them to the Pool

- When you call `conn.Close()` or the `Dispose()` method (via `using`), the connection is **not actually closed** — it is marked as available in the connection pool
- The underlying TCP connection to SQL Server remains open
- The next time `conn.Open()` is called with the same connection string, the pooled connection is reused
- This is why it is **critical** to always close/dispose connections — if you forget, the connection is never returned to the pool, and eventually the pool is exhausted

```csharp
// Both Close and Dispose return the connection to the pool
using (var conn = new SqlConnection(connectionString))
{
    conn.Open();
    // use connection
}  // Dispose() called → returns to pool

// Equivalent without using:
var conn = new SqlConnection(connectionString);
try
{
    conn.Open();
    // use connection
}
finally
{
    conn.Dispose();  // or conn.Close()
}
```

---

## Common Mistakes

### SELECT * in Production Queries

- `SELECT *` retrieves **every column** from the table, including large `VARCHAR(MAX)`, `VARBINARY`, and `XML` columns you may not need
- It breaks applications when columns are added, removed, or reordered in the table schema
- It prevents **covering indexes** — SQL Server cannot use an index-only scan if the query needs columns not in the index
- It increases network traffic, memory usage, and I/O
- Always select only the columns you actually need

```sql
-- BAD
SELECT * FROM Orders WHERE CustomerId = 42;

-- GOOD
SELECT OrderId, OrderDate, Total FROM Orders WHERE CustomerId = 42;
```

### Missing Indexes on JOIN/WHERE Columns

- Foreign keys used in JOINs should have indexes — without them, the inner side of a join becomes a table scan
- Columns frequently used in WHERE clauses should have indexes
- SQL Server does **not** automatically create indexes on foreign keys — you must do it manually
- Missing indexes on JOIN columns are one of the most common causes of slow queries

```sql
-- Always index foreign keys
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId ON Orders(CustomerId);
CREATE NONCLUSTERED INDEX IX_OrderItems_OrderId ON OrderItems(OrderId);
```

### Not Parameterizing Queries

- Building SQL strings by concatenating user input creates SQL injection vulnerabilities
- It also prevents execution plan reuse — every unique string is a separate query in the plan cache
- Always use parameterized queries or `sp_executesql`

### Using NOLOCK Everywhere

- `NOLOCK` (or `WITH (NOLOCK)`) is equivalent to `READ UNCOMMITTED` — it reads uncommitted (dirty) data
- It can return rows that were rolled back, read the same row twice, or miss rows entirely
- Use it only for approximate counts or monitoring where stale data is acceptable
- Prefer `READ COMMITTED SNAPSHOT` for a clean, blocking-free alternative

```sql
-- DANGEROUS: reads dirty data
SELECT COUNT(*) FROM Orders WITH (NOLOCK);

-- BETTER: use snapshot isolation
ALTER DATABASE MyDatabase SET READ_COMMITTED_SNAPSHOT ON;
SELECT COUNT(*) FROM Orders;  -- clean, consistent, non-blocking
```

### Ignoring Execution Plans

- Many developers write queries without ever looking at the execution plan
- The execution plan tells you **exactly** what SQL Server did — where the bottlenecks are, which indexes are used, and which operations are expensive
- Always check the execution plan when a query is slow — it will point you to the problem immediately

### Not Handling Parameter Sniffing

- A query that runs fast in development but suddenly becomes slow in production is often a parameter sniffing problem
- The development environment uses a different "first" parameter than production
- Be aware of parameter sniffing and test with **production-like parameter distributions**

### Large Result Sets Without Pagination

- Returning thousands or millions of rows at once overwhelms the application, network, and database
- Use `OFFSET...FETCH` for server-side pagination in SQL Server

```sql
-- Server-side pagination
DECLARE @PageSize INT = 20;
DECLARE @PageNumber INT = 3;

SELECT OrderId, OrderDate, Total
FROM Orders
ORDER BY OrderDate DESC
OFFSET (@PageNumber - 1) * @PageSize ROWS
FETCH NEXT @PageSize ROWS ONLY;
```

---

## Interview Questions

### Question 1: What is the difference between a clustered and non-clustered index?

- A **clustered index** determines the physical order of data on disk — the table data IS the clustered index. Only one clustered index per table. The leaf nodes contain the actual data rows.
- A **non-clustered index** is a separate structure with the indexed column values plus a pointer back to the data row (either the clustered index key or RID). You can have up to 999 non-clustered indexes per table.
- **Performance difference**: clustered index seeks return rows directly (no additional lookup), while non-clustered index seeks may require a Key Lookup to fetch columns not in the index.

### Question 2: When would you use a clustered index vs. a heap?

- Use a **clustered index** when you have an auto-incrementing primary key, when you frequently do range queries on a column, or when you need sorted retrieval. Most tables should have a clustered index.
- Use a **heap** (no clustered index) only for staging tables, very large temporary tables, or tables where inserts are extremely high-throughput and reads are minimal. Heaps are generally slower for reads because RID lookups are less efficient than key lookups.

### Question 3: What is a covering index and why does it matter?

- A **covering index** includes all the columns a query needs within the index itself (using the `INCLUDE` clause for non-key columns), so SQL Server never needs to perform a Key Lookup.
- This reduces I/O from two operations (index seek + key lookup) to one (index seek only), which can dramatically improve performance for frequently executed queries.
- Example: `CREATE NONCLUSTERED INDEX IX_Orders_Covering ON Orders(CustomerId, OrderDate) INCLUDE (Total)` covers the query `SELECT CustomerId, OrderDate, Total FROM Orders WHERE CustomerId = 42`.

### Question 4: How do you read an execution plan?

- Read **right-to-left** and **top-to-bottom**. Leaf nodes (rightmost) are table/index access operations. Internal nodes are joins, sorts, and aggregations. Data flows right to left.
- Focus on the **most expensive operators** (highest cost percentage). Look for Table Scans (should be Index Seeks), Key Lookups (should be covered), Hash Matches (check join strategy), and Sorts (check if index can eliminate the sort).

### Question 5: What is a deadlock and how do you prevent it?

- A **deadlock** is a circular wait where two transactions each hold a lock the other needs, blocking each other permanently. SQL Server detects deadlocks and rolls back the victim.
- **Prevention strategies**: (1) Always lock tables in the same order across all transactions. (2) Keep transactions as short as possible. (3) Use lower isolation levels (READ COMMITTED SNAPSHOT). (4) Add indexes on foreign keys to avoid lock escalation. (5) Implement retry logic for deadlock error 1205.

### Question 6: Explain parameter sniffing and how to handle it.

- **Parameter sniffing** occurs when SQL Server caches a plan optimized for the first parameter values, then reuses it for all subsequent calls with different parameter distributions.
- **Solutions**: (1) `OPTION (RECOMPILE)` for per-execution compilation. (2) `OPTION (OPTIMIZE FOR @param = value)` to target a specific value. (3) `OPTION (OPTIMIZE FOR UNKNOWN)` to use average statistics. (4) Local variable trick to defeat sniffing. (5) Query Store plan forcing for persistent fixes.

### Question 7: What isolation level should I use?

- **READ COMMITTED SNAPSHOT** is the recommended default for most web applications — it provides good concurrency without dirty reads or blocking between readers and writers.
- Use **READ COMMITTED** (the SQL Server default) for simple applications where some blocking is acceptable.
- Use **SERIALIZABLE** only when strict consistency is required for critical operations (e.g., financial calculations).
- Use **SNAPSHOT** when you need transaction-level consistent reads without blocking.

### Question 8: How does SQL injection work and how do you prevent it?

- SQL injection occurs when user input is concatenated into SQL strings, allowing an attacker to inject malicious SQL code.
- **Prevention**: Always use **parameterized queries** (`sp_executesql` or `SqlCommand.Parameters`). Never use string concatenation. When dynamic SQL is required, always use `sp_executesql` with parameters. Stored procedures with dynamic SQL concatenation are also vulnerable.

### Question 9: What is connection pooling and why is it important?

- **Connection pooling** maintains a cache of open database connections that are reused across requests, avoiding the overhead of creating new TCP connections (50-200ms each).
- ADO.NET manages pools per connection string. When you call `conn.Close()` or `Dispose()`, the connection returns to the pool (TCP stays alive). The next `Open()` reuses it.
- **Critical**: Always dispose connections (use `using` statements). If you forget, the connection leaks and the pool eventually exhausts, causing `Timeout expired` errors.

### Question 10: When should you NOT create an index?

- On columns with **low selectivity** (e.g., a boolean column where 99% of rows have the same value) — use a filtered index instead.
- On tables with **very high write throughput** and few reads — the write overhead may outweigh the read benefit.
- When the table is **very small** (< 1000 rows) — a table scan is faster than an index seek because SQL Server reads entire pages anyway.
- When you already have **too many indexes** (> 15) — audit and drop unused ones before adding more.

### Question 11: What is the difference between DELETE, TRUNCATE, and DROP?

- **DELETE** — removes rows one at a time, logs each deletion, fires triggers, can be rolled back, does not reset identity seed. Slower for large tables.
- **TRUNCATE** — removes all rows by deallocating data pages, logs only page deallocations (minimal logging), does not fire triggers, can be rolled back, resets identity seed. Much faster than DELETE.
- **DROP** — removes the entire table structure and data from the database. The table no longer exists. Cannot be rolled back in most cases (schema change is committed immediately).

### Question 12: What is the difference between WHERE and HAVING?

- **WHERE** filters rows **before** GROUP BY — it cannot use aggregate functions because aggregates have not been computed yet.
- **HAVING** filters groups **after** GROUP BY — it can use aggregate functions because the groups have been computed.
- Use WHERE to filter individual rows and HAVING to filter aggregated results.

```sql
-- WHERE filters rows, HAVING filters groups
SELECT CustomerId, COUNT(*) AS OrderCount
FROM Orders
WHERE OrderDate >= '2024-01-01'  -- filters rows first
GROUP BY CustomerId
HAVING COUNT(*) > 5;  -- then filters groups
```

### Question 13: What is the difference between UNION and UNION ALL?

- **UNION** combines results from two queries and **removes duplicates** (requires a sort or hash operation to deduplicate). Slower.
- **UNION ALL** combines results from two queries **without removing duplicates**. Faster because no deduplication step.
- Use UNION ALL unless you specifically need to eliminate duplicates. If you know the queries return distinct results, UNION ALL is always preferred.

### Question 14: What are statistics and why do they matter?

- **Statistics** contain information about the distribution of column values and are used by the query optimizer to estimate cardinality and choose the best execution plan.
- Stale or missing statistics lead to poor cardinality estimates, which cause the optimizer to choose wrong plan types (e.g., Table Scan instead of Index Seek, or Nested Loop instead of Hash Join).
- SQL Server auto-updates statistics, but you may need to manually update them for large tables or after significant data changes: `UPDATE STATISTICS Orders WITH FULLSCAN`.

### Question 15: What is the difference between INNER JOIN, LEFT JOIN, RIGHT JOIN, and FULL OUTER JOIN?

- **INNER JOIN** — returns only rows that have matching values in both tables. Rows without matches are excluded.
- **LEFT JOIN** — returns all rows from the left table and matching rows from the right table. If no match, the right side columns are NULL.
- **RIGHT JOIN** — returns all rows from the right table and matching rows from the left table. If no match, the left side columns are NULL.
- **FULL OUTER JOIN** — returns all rows from both tables. If no match on either side, the missing side columns are NULL.

### Question 16: What is the difference between a stored procedure and a function?

- A **stored procedure** can return multiple result sets, use output parameters, return an integer status code, and perform transactions. It is called with `EXEC`.
- A **user-defined function** (scalar or table-valued) returns a single value or table, cannot modify database state (no INSERT/UPDATE/DELETE unless it's an CLR function), and is called like `dbo.FunctionName()`.
- Use stored procedures for complex business logic and operations; use functions for reusable calculations within queries.

### Question 17: How do you optimize a slow stored procedure?

- (1) Check the **execution plan** to identify the most expensive operators. (2) Look for table scans and add appropriate indexes. (3) Check for **parameter sniffing** and use OPTIMIZE FOR or RECOMPILE if needed. (4) Update **statistics**. (5) Avoid cursors — use set-based operations. (6) Use `SET NOCOUNT ON`. (7) Return only necessary columns (avoid SELECT *). (8) Use covering indexes. (9) Check for unnecessary sorts or hash operations.

### Question 18: What is the difference between temp tables and table variables?

- **Local temp tables** (`#TempTable`) — stored in tempdb, visible only to the current session, support indexes and statistics, participate in transactions, can be used in dynamic SQL. Dropped automatically when the session ends.
- **Table variables** (`@TempTable`) — historically stored in memory (though SQL Server 2014+ can spill to tempdb for large table variables), limited index support (only primary key in older versions), transaction logging is minimal. Scope is limited to the batch/procedure.
- **General rule**: Use table variables for small datasets (< 1000 rows). Use temp tables for larger datasets or when you need indexes and statistics.

### Question 19: What is the Query Store and when should you use it?

- The **Query Store** is a persistent historical storage of query execution plans and performance statistics in SQL Server 2016+.
- It captures every query's execution history, allowing you to: (1) identify performance regressions, (2) force a specific execution plan for a query, (3) compare plan performance over time.
- Enable it on production databases: `ALTER DATABASE MyDatabase SET QUERY_STORE = ON`.
- It is the **first place to look** when a query's performance degrades without code changes — it shows plan changes over time.

### Question 20: Explain the difference between optimistic and pessimistic concurrency.

- **Pessimistic concurrency** — SQL Server uses locks to prevent conflicts. When a transaction reads data, it acquires shared locks. When it modifies data, it acquires exclusive locks. Other transactions must wait for locks to be released. Default behavior in SQL Server (READ COMMITTED, REPEATABLE READ, SERIALIZABLE).
- **Optimistic concurrency** — SQL Server uses row versioning instead of locks. Readers never block writers, writers never block readers. Conflicts are detected at commit time (if two transactions modify the same row, one is rolled back). Used with SNAPSHOT and READ COMMITTED SNAPSHOT isolation levels.
- **Trade-off**: pessimistic ensures consistency by blocking others (lower concurrency). Optimistic allows higher concurrency but may require retry logic for write conflicts.

### Question 21: What is lock escalation and when does it happen?

- **Lock escalation** is the process where SQL Server converts many fine-grained locks (row or page locks) into a single coarse-grained lock (table lock) to reduce lock memory overhead.
- SQL Server escalates when a single statement acquires more than **5,000 locks** on a single table (the threshold can vary).
- Lock escalation can cause severe **blocking** — a table lock blocks all other transactions on that table.
- **Prevention**: (1) Keep transactions short. (2) Update data in batches. (3) Use partitioning. (4) Use `ROWLOCK` hint (use sparingly). (5) Ensure proper indexes so SQL Server locks fewer rows.

### Question 22: What is the difference between IDENTITY and SEQUENCE?

- **IDENTITY** — a table property that auto-generates sequential numbers for a column. Limited to one identity column per table. Cannot be reused or reset without DBCC CHECKIDENT. Tied to the table.
- **SEQUENCE** — a standalone database object (SQL Server 2012+) that generates sequential numbers. Can be shared across multiple tables. Supports custom increment, min/max values, and cycle behavior. More flexible than IDENTITY.
- Use SEQUENCE when you need auto-generated IDs shared across multiple tables or when you need more control over the number generation.

```sql
-- SEQUENCE example
CREATE SEQUENCE OrderNumberSeq
    START WITH 1000
    INCREMENT BY 1
    MINVALUE 1000
    NO MAXVALUE
    NO CYCLE;

-- Use it in an INSERT
INSERT INTO Orders (OrderNumber, Total)
VALUES (NEXT VALUE FOR OrderNumberSeq, 250.00);
```

### Question 23: How does SQL Server handle a query that selects 90% of a table?

- When a query selects a large percentage of rows (typically > 30-50%), the optimizer often chooses a **Table Scan** (or Clustered Index Scan) over an Index Seek, because scanning the entire table is cheaper than doing an index seek + thousands of individual Key Lookups.
- This is **not necessarily a problem** — the optimizer is making the right choice. An index seek with 100,000 key lookups is slower than a single sequential scan.
- **The solution is not always "add an index"** — if you truly need 90% of the rows, a scan is often the fastest approach. Instead, optimize the query to return fewer rows or use a covering index if you only need certain columns.
