# Database and SQL Questions

## Questions

1. What is DBMS?
2. What is RDBMS?
3. What is primary key?
4. What is foreign key?
5. What is unique key?
6. Difference between primary key and unique key.
7. What is index?
8. How does indexing improve performance?
9. What are the disadvantages of indexes?
10. What is clustered index?
11. What is non-clustered index?
12. What is composite index?
13. What is covering index?
14. What is filtered index?
15. What is index selectivity?
16. What is query execution plan?
17. How do you analyze an execution plan?
18. What is table scan?
19. What is index seek?
20. What is index scan?
21. What is key lookup?
22. What are joins?
23. Difference between inner join and left join.
24. Difference between left join and right join.
25. Difference between full join and cross join.
26. What is normalization?
27. What are normal forms?
28. What is denormalization?
29. When should you denormalize?
30. What is stored procedure?
31. Advantages and disadvantages of stored procedures.
32. What is function in SQL?
33. Difference between stored procedure and function.
34. What is trigger?
35. What is view?
36. What is materialized view?
37. What is transaction?
38. What are ACID properties?
39. What is isolation level?
40. Difference between read committed and repeatable read.
41. What is dirty read?
42. What is non-repeatable read?
43. What is phantom read?
44. What is deadlock?
45. How do you prevent deadlocks?
46. What is locking?
47. What is row lock?
48. What is table lock?
49. What is optimistic locking?
50. What is pessimistic locking?
51. What is connection pooling?
52. How do you optimize a slow query?
53. How do you optimize a stored procedure?
54. How do you handle large data processing?
55. What is pagination in SQL?
56. Difference between offset pagination and keyset pagination.
57. What is query parameterization?
58. What is SQL injection?
59. How do you prevent SQL injection?
60. What is ETL?
61. What is SSIS?
62. What is data validation in ETL?
63. How do you handle duplicate records?
64. How do you handle missing data?
65. How do you reconcile source and target data?

---

## Answers

1. What is DBMS?
   - **Answer:**
      - A DBMS is software that stores, manages, and retrieves data systematically
      - MSSQL is a commonly used RDBMS for enterprise data management
      - Execution plans and indexing are essential to keep queries fast
      - Wait statistics in MSSQL can identify where the DBMS is spending most of its time during long-running stored procedure executions
2. What is RDBMS?
   - **Answer:**
      - An RDBMS stores data in related tables using keys and indexes
      - MSSQL is a widely used RDBMS for designing normalized schemas for inventory data
      - Foreign keys link related entities such as partners, products, and transactions
      - RDBMS enforces strong ACID compliance which is critical for inventory accuracy and reporting; NoSQL may be considered but rejected when relational integrity and transactional guarantees are needed
3. What is primary key?
   - **Answer:**
      - A primary key uniquely identifies each row in a table
      - A `SerialNumber` column can serve as the primary key to ensure every serial record is unique
      - Records can be traced back to a specific shipment or transaction
      - Primary key choice directly affects clustered index performance since the PK creates the clustered index by default in MSSQL; narrow integer keys are preferred over wide composite keys in high-volume tables because they reduce page size and improve insert throughput
4. What is foreign key?
   - **Answer:**
      - A foreign key links two tables and enforces referential integrity
      - Foreign keys ensure each report record points to a valid parent entity ID
      - Orphan data is prevented when parent records are deactivated
      - Missing foreign key indexes can cause nested loop joins in long-running procedures; adding them is one of the first optimization steps and immediately reduces join costs
5. What is unique key?
   - **Answer:**
      - A unique key ensures all values in a column or column set are distinct
      - A unique constraint on `(PartnerID, SerialNumber)` prevents duplicate serial records from entering the system
      - In MSSQL, a unique constraint creates a unique index underneath; both enforce uniqueness but a constraint is a logical declaration while an index is the physical implementation
      - Filtered unique constraints on active records only can prevent deactivated records from conflicting with new entries
6. Difference between primary key and unique key.
   - **Answer:**
      - A table has only one primary key, which is clustered by default in MSSQL
      - You can have multiple unique keys
      - Primary keys are used for entity identity and unique keys for business-level uniqueness like partner codes
      - Choosing the wrong clustered index — on a wide unique key instead of a narrow primary key — degrades insert performance in batch loads because wider keys increase page size and cause more page splits during sequential inserts
7. What is index?
   - **Answer:**
      - An index is a database structure that speeds up data retrieval
      - Composite indexes on columns used in WHERE and JOIN conditions of slow stored procedures can change table scans to index seeks
      - The Database Engine Tuning Advisor can recommend indexes based on workload analysis but manual review is needed to avoid over-indexing
8. How does indexing improve performance?
   - **Answer:**
      - Indexing creates a sorted data structure (B-tree in MSSQL) that allows the engine to locate rows without scanning the entire table
      - Adding a covering index can dramatically reduce query execution time
      - The query optimizer chooses between index seek, scan, and lookup based on index structure, statistics density, and the cost model; a seek traverses the B-tree directly, a scan reads the full index, and a lookup fetches individual rows from the clustered index
9. What are the disadvantages of indexes?
   - **Answer:**
      - Indexes slow down INSERT, UPDATE, and DELETE operations because the index must be maintained
      - Read performance for validation queries must be balanced against write overhead during batch ingestion of large volumes of records
      - Index fragmentation can be monitored using `sys.dm_db_index_physical_stats` and weekly rebuild jobs can prevent performance degradation over time; fragmented indexes cause more I/O because pages are split across the disk
10. What is clustered index?
    - **Answer:**
       - A clustered index determines the physical order of data in a table
       - In MSSQL, the primary key creates a clustered index by default
       - The clustered index key should be chosen based on the most common query patterns — for example, choosing a date column for range-based reporting queries
       - Choosing clustered indexes on incrementing columns (like identity columns) avoids page splits during inserts because new rows append to the end; page splits and insert hotspots can occur during high-volume inserts when the clustered key is non-sequential
11. What is non-clustered index?
    - **Answer:**
       - A non-clustered index is a separate structure that contains key values and pointers to the actual rows
       - Non-clustered indexes on foreign key columns speed up JOIN operations between related tables
       - Include columns in non-clustered indexes can make them covering indexes; adding the SELECT list columns as INCLUDE columns eliminates the need to access the base table, which is used extensively to eliminate key lookups
12. What is composite index?
    - **Answer:**
       - A composite index is an index on multiple columns, with the column order determining its effectiveness
       - A composite index on `(PartnerID, ReportDate)` is effective when a procedure always filters by partner first, then by date range
       - Column order matters — leading with the most selective column gives the best performance because the B-tree is sorted by the first column first; this can be verified by comparing estimated execution plans before and after reordering columns
13. What is covering index?
    - **Answer:**
       - A covering index includes all columns referenced by a query, eliminating the need to access the table data
       - Non-clustered indexes can be rebuilt with INCLUDE columns for the SELECT list
       - This eliminates expensive key lookups
       - Covering indexes are larger and increase maintenance cost since every INSERT/UPDATE must maintain the wider index; they should be created only for the most frequent queries where the performance gain justifies the storage overhead
14. What is filtered index?
    - **Answer:**
       - A filtered index indexes only a subset of rows based on a WHERE condition
       - A filtered index on an inventory table where `Status = 'Active'` keeps the index small and fast for active-record lookups
       - Filtered indexes maintain better statistics for the relevant data subset since the statistics only cover indexed rows; this helps the optimizer choose better plans because the cardinality estimates are more accurate for active records
15. What is index selectivity?
    - **Answer:**
       - Selectivity measures how many rows match a given value — high selectivity means few rows per value
       - Column selectivity should be analyzed to determine optimal index column order: high-selectivity columns should come first in composite indexes
       - Low-selectivity indexes (like on a boolean flag with only two distinct values) are often ignored by the optimizer because a table scan is cheaper than an index seek followed by thousands of key lookups; this can be confirmed through actual execution plan analysis where the optimizer chooses scans over low-selectivity indexes
16. What is query execution plan?
    - **Answer:**
       - An execution plan shows how the database engine processes a query — which indexes it uses, how it joins tables, and where the cost is
       - When optimizing slow queries, start by examining the actual execution plan for each subquery in the stored procedure
       - Reading an actual plan involves looking for table scans, hash joins with large row estimates, and operators where estimated vs actual row counts diverge significantly — large divergence indicates stale statistics that need updating
17. How do you analyze an execution plan?
    - **Answer:**
       - Start with the most expensive operator (highest % of cost), check if it's a scan vs seek, look for missing index suggestions, and compare estimated vs actual rows
       - A Cartesian product can occur when a JOIN predicate is missing — it shows as a massive nested loop with 100M+ rows
       - Enable actual execution plans in SSMS via `Ctrl+M` or the toolbar; use `SET STATISTICS TIME/IO ON` for detailed CPU and I/O metrics; always compare plans before and after changes to verify that optimizations actually improved the plan
18. What is table scan?
    - **Answer:**
       - A table scan reads every row in the table to find matching data
       - Table scans on large tables indicate missing indexes — for example, a scan on a 5-million-row transaction table because there was no index on the join column
       - This can be the primary reason a report takes hours to complete
       - Table scans are acceptable for small tables (under ~10K rows) but disastrous for large ones; adding a single index on the join column can convert the scan to a seek and reduce the query from hours to seconds
19. What is index seek?
    - **Answer:**
       - An index seek navigates the B-tree to find only the relevant rows, making it highly efficient
       - After adding a composite index on `(PartnerID, ReportDate)`, the execution plan can change from a table scan to an index seek
       - Query time can drop from minutes to seconds
       - Seeks depend on sargable WHERE conditions; non-sargable functions like `YEAR(DateCol)` prevent index usage, so rewriting them as range comparisons like `DateCol >= '2024-01-01' AND DateCol < '2025-01-01'` makes them seekable
20. What is index scan?
    - **Answer:**
       - An index scan reads all rows in an index (non-clustered) rather than the table
       - Faster than a table scan but slower than a seek
       - Index scans can occur on a date index when queries don't filter selectively enough
       - An index scan reads fewer pages than a table scan since the index is narrower (contains only indexed columns plus included columns), but it is still expensive for large tables because it traverses every leaf page of the index
21. What is key lookup?
    - **Answer:**
       - A key lookup happens when a non-clustered index doesn't cover the query, so the engine fetches the actual row from the clustered index
       - Key lookups can be eliminated by adding INCLUDE columns to existing non-clustered indexes
       - Multiple key lookups per row compound into thousands of random I/O operations since each lookup requires a separate B-tree traversal to the clustered index; the index tuning advisor can identify which lookups are most costly
22. What are joins?
    - **Answer:**
       - Joins combine rows from two or more tables based on related columns
       - Joins should be optimized by ensuring both sides have proper indexes and that join predicates use the correct columns
       - A missing predicate can cause a cross join between large tables
       - The three physical join operators are nested loop (best for small sets), hash match (best for large unsorted sets), and merge join (best for pre-sorted inputs); the optimizer's choice is influenced through indexing and occasionally through query hints when the optimizer picks a suboptimal plan
23. Difference between inner join and left join.
    - **Answer:**
       - Inner join returns only matching rows from both tables
       - Left join returns all rows from the left table and matching from the right
       - Left joins can be used to find records that exist in the source but not in the target
       - Using a LEFT JOIN where an INNER JOIN suffices can cause incorrect row counts because NULLs from unmatched right-table rows inflate aggregates; this can be caught by comparing before-and-after output
24. Difference between left join and right join.
    - **Answer:**
       - Left join preserves all left-table rows; right join preserves all right-table rows
       - Almost always use left joins for readability and consistency, since right joins can make query logic harder to follow
       - A RIGHT JOIN inside a CTE can produce confusing results because the anchor table is on the right side, making the query read backwards; refactoring to left joins improves clarity
25. Difference between full join and cross join.
    - **Answer:**
       - Full join returns all rows from both tables with matches where available
       - Cross join returns the Cartesian product of both tables
       - Omitting the ON clause can accidentally produce a cross join, which may generate millions of rows instead of the expected few thousand
       - Always inspect actual execution plans to catch accidental Cartesian products; the optimizer shows them clearly as nested loops with massively inflated row estimates
26. What is normalization?
    - **Answer:**
       - Normalization reduces data redundancy by splitting tables into related entities
       - Partner data can be normalized into separate tables (Partners, Products, Transactions) to avoid duplicate storage and update anomalies
       - Normalization helps data integrity but requires careful indexing on foreign keys to maintain join performance; every foreign key column needs a non-clustered index to avoid scans during joins
27. What are normal forms?
    - **Answer:**
       - Normal forms are progressive rules to eliminate redundancy: 1NF removes repeating groups, 2NF removes partial dependencies, 3NF removes transitive dependencies
       - A schema in 3NF reduces redundancy and should be verified before designing indexes
       - Report tables may be denormalized for performance while keeping the transactional schema in 3NF; the reporting tables store pre-computed aggregates to avoid expensive joins during dashboard rendering
28. What is denormalization?
    - **Answer:**
       - Denormalization intentionally adds redundancy for read performance
       - Computed columns can be added for frequently calculated metrics so stored procedures don't have to compute them on the fly across millions of rows
       - The trade-off is that denormalization speeds up reads but complicates writes since redundant data must be kept in sync; it should only be applied to reporting tables, not transactional ones
29. When should you denormalize?
    - **Answer:**
       - Denormalize when read performance is critical and the data is mostly static, like pre-aggregated reporting tables
       - Denormalizing frequently accessed aggregations into summary tables improves performance — for example, monthly partner summaries in a separate table that the dashboard queries instead of aggregating raw data each time
       - Indexed views in MSSQL serve as a middle ground — they look like views but are physically stored and maintained by the engine; the engine automatically updates them when underlying data changes, giving denormalization benefits without manual ETL
30. What is stored procedure?
    - **Answer:**
       - A stored procedure is a pre-compiled batch of SQL statements stored in the database
       - Reporting systems often rely heavily on stored procedures
       - A main reporting procedure can be optimized from hours to minutes by rewriting queries and fixing indexes
       - Profile each section of the procedure using `SET STATISTICS TIME ON` to find per-statement CPU and elapsed time; break it down to find the individual expensive queries rather than guessing
31. Advantages and disadvantages of stored procedures.
    - **Answer:**
       - Advantages: pre-compiled execution plans, reduced network traffic, centralized business logic
       - Disadvantages: harder to version control, debug, and test compared to application code
       - Stored procedures lack native versioning, making it difficult to diff procedure versions
       - A good approach: keep reporting logic in procedures where set-based operations are fast; keep business validation in application code where it is testable with unit tests and easier to version control
32. What is function in SQL?
    - **Answer:**
       - A function returns a scalar value or table
       - Scalar functions provide reusable logic like date formatting
       - Scalar functions can be replaced with inline table-valued functions because scalar functions cause row-by-row execution
       - Scalar functions in WHERE clauses make queries non-sargable because the engine must evaluate the function for every row; converting them to inline TVFs eliminates the performance issue since inline TVFs are expanded into the query like a view
33. Difference between stored procedure and function.
    - **Answer:**
       - Functions must return a value and cannot modify data
       - Stored procedures can modify data and have side effects
       - Stored procedures are suited for ETL operations and functions for computed columns or reusable lookups
       - Functions in FROM clauses can be inline (optimized like views, expanded into the query) or multi-statement (materialized into temp tables with a defined return type); prefer inline TVFs for performance since the optimizer can push predicates into them
34. What is trigger?
    - **Answer:**
       - A trigger runs automatically on INSERT, UPDATE, or DELETE
       - An AFTER INSERT trigger on a serial records table can update a partner's last-activity timestamp automatically
       - Complex triggers cause cascading issues — a trigger chain where an INSERT trigger fires another INSERT which fires an UPDATE can lock a table for minutes during bulk inserts
35. What is view?
    - **Answer:**
       - A view is a saved SQL query that looks like a table
       - Views can be created for frequently joined table combinations so report developers don't have to remember complex join conditions
       - Views can hide complexity but also hide performance problems; a view joining 10 tables looks simple to the caller but runs all 10 joins every time it is queried
36. What is materialized view?
    - **Answer:**
       - In MSSQL, indexed views persist the result set physically, updated automatically
       - Indexed views can precompute aggregations for dashboards
       - This keeps dashboard queries fast without manual ETL
       - Indexed views require SCHEMABINDING and cannot use DISTINCT in aggregates; this can be worked around by using `COUNT_BIG` instead of `COUNT` since `COUNT_BIG` is allowed with SCHEMABINDING
37. What is transaction?
    - **Answer:**
       - A transaction groups multiple operations into an atomic unit — all succeed or all roll back
       - Batch operations should be wrapped in transactions — for example, inserting 10,000 serial records in a single transaction
       - This ensures partial failures don't corrupt the data state
       - Transaction scope should be kept short to avoid holding locks; deadlocks can be handled when concurrent batches run by catching the 1205 error and retrying with a backoff strategy
38. What are ACID properties?
    - **Answer:**
       - Atomicity (all-or-nothing), Consistency (valid state before and after), Isolation (concurrent transactions don't interfere), Durability (committed data survives failures)
       - MSSQL's ACID compliance is why it is chosen for systems where data loss or corruption is unacceptable — such as inventory-level serial tracking
       - Isolation levels trade strictness for performance; snapshot isolation is useful for read-only reporting to avoid writer blocking, giving readers a consistent point-in-time snapshot without holding shared locks
39. What is isolation level?
    - **Answer:**
       - Isolation level controls how transactions see each other's changes
       - READ UNCOMMITTED (with NOLOCK hints) can be used on read-only report queries when zero blocking on writes is required
       - Slight dirty reads are acceptable in such scenarios
       - The spectrum runs from READ UNCOMMITTED (dirty reads possible, no shared locks) to SERIALIZABLE (full isolation, range locks, no concurrency); choose based on whether accuracy or speed matters for each query
40. Difference between read committed and repeatable read.
    - **Answer:**
       - Read committed prevents dirty reads but allows non-repeatable reads
       - Repeatable read prevents both
       - Repeatable read can be used to ensure two successive reads of the same baseline data match during validation
       - Repeatable read holds shared locks until the transaction ends, which increases blocking but is necessary for reconciliation accuracy
41. What is dirty read?
    - **Answer:**
       - A dirty read occurs when a transaction reads uncommitted data from another transaction
       - Dirty reads via NOLOCK hints can be acceptable for monitoring dashboards where stale data by a few seconds is acceptable
       - Should not be used in transaction-critical systems
       - NOLOCK can also cause missed or double-read rows due to page splits moving rows between reads; this is why it is avoided for any financial or count-based reports
42. What is non-repeatable read?
    - **Answer:**
       - A non-repeatable read happens when a row changes between two reads in the same transaction
       - This can occur when baseline data is being updated by another process during validation
       - Snapshot isolation can be used to resolve this
       - Different isolation levels handle this differently: READ COMMITTED allows it (re-reads get the latest committed version), REPEATABLE READ prevents it by holding shared locks, and snapshot isolation gives consistency without blocking by reading from the version store
43. What is phantom read?
    - **Answer:**
       - A phantom read occurs when new rows appear between two reads in the same transaction
       - Phantom reads may not significantly affect aggregate reports
       - Most reporting can stay at READ COMMITTED
       - SERIALIZABLE prevents phantoms by acquiring range locks on index key ranges, and snapshot isolation prevents them through row versioning; both have performance costs, so the choice depends on whether phantom consistency is required
44. What is deadlock?
    - **Answer:**
       - A deadlock occurs when two transactions each hold a lock the other needs, and SQL Server chooses a victim
       - Concurrent batch ingestion processes can occasionally deadlock on shared tables
       - One process is killed automatically
       - Deadlocks can be reduced by ensuring all transactions access tables in the same order and keeping transaction durations short to minimize the lock hold window
45. How do you prevent deadlocks?
    - **Answer:**
       - Prevent deadlocks by accessing tables in a consistent order across transactions, keeping transactions short, and using appropriate isolation levels
       - Row versioning can reduce lock contention during concurrent processing
       - SQL Server Profiler can capture deadlock graphs (or Extended Events in newer versions), which can be visualized with deadlock graph viewers to understand which queries and objects were involved and where the lock cycle occurred
46. What is locking?
    - **Answer:**
       - Locking controls concurrent access to data
       - MSSQL uses row, page, and table locks based on the operation
       - Lock escalation from row to table locks can occur during bulk operations, blocking other queries; breaking the batch into smaller chunks prevents this
       - The lock hierarchy goes row -> page -> table; lock escalation happens when a single transaction accumulates too many row or page locks on one table, and MSSQL escalates to a table lock; locks can be monitored using `sys.dm_tran_locks` and `sys.dm_exec_requests`
47. What is row lock?
    - **Answer:**
       - A row lock locks a single row to allow maximum concurrency
       - Row locks allow multiple concurrent feeds to be processed on different records without blocking each other
       - Row locks consume memory for each lock resource and can escalate to table locks when a transaction accumulates too many; lock granularity can be checked using execution plans and `sys.dm_tran_locks`
48. What is table lock?
    - **Answer:**
       - A table lock locks the entire table, preventing any concurrent access
       - Long-running procedures may escalate to table locks on large tables during report generation
       - This blocks all incoming data feeds
       - `ALTER TABLE ... SET LOCK_ESCALATION = AUTO` can control escalation behavior; partitioning large tables also limits lock escalation scope to individual partitions rather than the entire table
49. What is optimistic locking?
    - **Answer:**
       - Optimistic locking assumes conflicts are rare and checks at commit time using a version column
       - A rowversion column in an entity can detect concurrent modifications during reconciliation without holding long locks
       - If a `DbUpdateConcurrencyException` occurs in the application layer, handle it by refreshing the data, merging changes, and retrying the operation rather than blocking readers during the entire transaction
50. What is pessimistic locking?
    - **Answer:**
       - Pessimistic locking assumes conflicts are likely and locks resources upfront using `WITH (UPDLOCK)` or `SELECT ... FOR UPDATE`
       - UPDLOCK hints in a critical section can prevent double-allocation of the same serial number
       - Pessimistic locking reduces throughput because locks are held for the entire transaction, but it guarantees correctness; it is acceptable for low-concurrency critical operations like serial allocation
51. What is connection pooling?
    - **Answer:**
       - Connection pooling reuses database connections instead of creating new ones per request
       - In a Spring Boot application, HikariCP can be configured with a max pool size of 20
       - Pool size should be tuned based on the number of concurrent API requests and database capacity
       - Connection exhaustion can be diagnosed by monitoring active connections in MSSQL using `sys.dm_exec_sessions`; increase the pool size gradually while watching for contention and `ConnectionIsTimeout` errors
52. How do you optimize a slow query?
    - **Answer:**
       - Start by examining the actual execution plan for table scans, high-cost operators, and missing index hints
       - Then check indexes, rewrite non-sargable conditions, and add INCLUDE columns
       - Systematic optimization of each subquery can dramatically reduce execution time
       - Full playbook: enable `SET STATISTICS TIME/IO ON`, check wait stats for I/O or CPU bottlenecks, look for parameter sniffing issues (where a cached plan is optimal for one parameter value but terrible for another), and test with `OPTION (RECOMPILE)` if parameter sniffing is the root cause
53. How do you optimize a stored procedure?
    - **Answer:**
       - Profile the procedure by running it with `SET STATISTICS TIME ON` and capturing the actual plan
       - Identify the most expensive statements, optimize them individually
       - Re-run the whole procedure to confirm the cumulative improvement
       - Systematic optimization can reduce execution time from hours to minutes
       - Parameterize the procedure to prevent plan caching issues where different parameter values produce vastly different row counts; use `WITH RECOMPILE` for procedures with highly varying parameters so the engine generates a fresh plan per execution
54. How do you handle large data processing?
    - **Answer:**
       - Use batch processing with set-based operations, avoid cursors and row-by-row processing, and ensure proper indexing
       - Large volumes of records can be processed in batches of 1,000 using `INSERT ... SELECT` with `ROWNUMBER()` filtering
       - Batch size tuning matters: too small causes many round-trips and transaction log overhead, too large causes lock escalation and memory pressure; load testing can help find the optimal batch size for a given table
55. What is pagination in SQL?
    - **Answer:**
       - Pagination retrieves a subset of rows using `OFFSET` and `FETCH NEXT` in MSSQL
       - APIs should implement pagination — for example, `ORDER BY PartnerID OFFSET 0 ROWS FETCH NEXT 50 ROWS ONLY`
       - This avoids loading thousands of records at once
       - Offset pagination degrades with large offsets because the engine still reads and discards all skipped rows; for page 10,000 with a page size of 50, it reads 500,000 rows just to skip them — switch to keyset pagination when this becomes a problem
56. Difference between offset pagination and keyset pagination.
    - **Answer:**
       - Offset pagination skips N rows using `OFFSET`
       - Keyset pagination uses `WHERE id > @lastSeenId`
       - Offset pagination becomes slow on large datasets because it reads all skipped rows each time
       - Keyset pagination can be used for audit logs and similar append-only data for faster queries
       - Keyset pagination is ideal for infinite scroll on sorted unique columns but cannot skip to arbitrary pages; offset pagination supports arbitrary page access but pays the cost of scanning skipped rows
57. What is query parameterization?
    - **Answer:**
       - Parameterization uses placeholders instead of hardcoded values in SQL, which allows plan reuse and prevents SQL injection
       - In Spring Boot JPA repositories, queries are automatically parameterized via prepared statements
       - Queries without parameterization can cause plan cache bloat because each unique literal value generates a separate cached plan; forced parameterization using `ALTER DATABASE SET PARAMETERIZATION FORCED` groups similar queries under one plan
58. What is SQL injection?
    - **Answer:**
       - SQL injection is an attack where malicious SQL is inserted through user input
       - This is prevented by never concatenating user input into SQL strings
       - All queries should use JPA repositories or parameterized stored procedures
       - String concatenation like `"WHERE name = '" + userInput + "'"` is vulnerable because the user can close the quote and inject additional SQL; parameterized queries like `WHERE name = @name` prevent it because the parameter value is treated as data, never parsed as SQL
59. How do you prevent SQL injection?
    - **Answer:**
       - Always use parameterized queries, stored procedures, or ORM frameworks that handle escaping
       - Strictly use Spring Data JPA or `@Query` with named parameters
       - Validate inputs at the controller layer before they reach the database
       - Stored procedures with `EXEC` or `sp_executesql` built from concatenated strings inside the procedure can still be vulnerable; never build dynamic SQL inside procedures — instead use CASE statements or multiple queries to handle conditional logic
60. What is ETL?
    - **Answer:**
       - ETL (Extract, Transform, Load) is the process of moving data from source systems to a target database
       - SSIS is commonly used for ETL pipelines that extract partner data from various formats
       - Data is transformed (cleaned, validated, aggregated) and loaded into MSSQL reporting tables
       - A typical flow: extract CSV files from partner uploads, transform serial numbers to a standardized format (trimming whitespace, validating against regex patterns), and load them into inventory tables with row-level error logging for any records that fail validation
61. What is SSIS?
    - **Answer:**
       - SSIS (SQL Server Integration Services) is Microsoft's ETL tool for data migration and integration
       - SSIS packages can automate scheduled data loads from partner files into staging and reporting tables
       - SSIS data flow tasks with lookup transformations can validate partner codes against a reference table; invalid rows can be redirected to an error output file so the main flow continues while invalid records are logged for follow-up
62. What is data validation in ETL?
    - **Answer:**
       - Data validation in ETL ensures that transformed data meets business rules before loading
       - During ETL, serial number formats, partner codes against a master list, and date ranges should be validated before inserting into the target table
       - Three validation layers: schema validation (data types, lengths, required fields), business rule validation (serial format matches regex, dates within valid range), and referential integrity checks (partner ID exists in the master table); each failure type should be logged separately for reporting
63. How do you handle duplicate records?
    - **Answer:**
       - Use `ROW_NUMBER()` with `PARTITION BY` to identify duplicates, then decide whether to deduplicate (keep first/latest) or reject
       - Records can be partitioned by serial number and the record with the latest timestamp kept
       - Rejected duplicates should be logged for notification
       - The business trade-off: sometimes duplicates should be rejected (unique serials must remain unique), sometimes merged (duplicate partner submissions should be combined); depends on the data domain and whether the duplicates represent true errors or valid overlapping submissions
64. How do you handle missing data?
    - **Answer:**
       - For missing data, first check if it's truly required or can be defaulted
       - If a serial record is missing the partner ID, it can be logged to an error table and skipped
       - Ownership cannot be determined without the partner
       - NULL handling strategies differ: `COALESCE` provides a default fallback value, `ISNULL` checks for NULL before processing; missing required fields should fail fast rather than propagate bad data downstream, while missing optional fields can be safely defaulted
65. How do you reconcile source and target data?
    - **Answer:**
       - Reconciliation compares row counts, checksums, or key values between source and target after ETL
       - `EXCEPT` and `INTERSECT` can be used to find mismatches between datasets
       - Reconciliation can be automated as the final step of the ETL pipeline; an email with the mismatch count can be sent so the team can investigate before downstream reports are generated
