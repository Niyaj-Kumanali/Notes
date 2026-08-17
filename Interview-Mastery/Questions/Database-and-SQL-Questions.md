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
      - In CDMS, used MSSQL as the DBMS to handle Lenovo's channel partner data
      - Worked heavily with execution plans and indexing to keep queries fast
      - Analyzed wait statistics in MSSQL to identify where the DBMS was spending most of its time during the 5-hour stored procedure runs
2. What is RDBMS?
   - **Answer:**
      - An RDBMS stores data in related tables using keys and indexes
      - MSSQL is the RDBMS used at Talentpace, where I designed normalized schemas for inventory data
      - Foreign keys linked partners, products, and transactions
      - RDBMS enforces strong ACID compliance which was critical for inventory accuracy and reporting; NoSQL was considered but rejected because we needed relational integrity and transactional guarantees
3. What is primary key?
   - **Answer:**
      - A primary key uniquely identifies each row in a table
      - In the inventory system, used `SerialNumber` as the primary key to ensure every serial record was unique
      - Could be traced back to a specific partner shipment
      - Primary key choice directly affects clustered index performance since the PK creates the clustered index by default in MSSQL; narrow integer keys are preferred over wide composite keys in high-volume tables because they reduce page size and improve insert throughput
4. What is foreign key?
   - **Answer:**
      - A foreign key links two tables and enforces referential integrity
      - In CDMS, used foreign keys to ensure each report record pointed to a valid partner ID
      - Prevented orphan data when partners were deactivated
      - Missing foreign key indexes caused nested loop joins in the 5-hour procedure; adding them was one of the first optimization steps and immediately reduced join costs
5. What is unique key?
   - **Answer:**
      - A unique key ensures all values in a column or column set are distinct
      - In the inventory validation engine, applied a unique constraint on `(PartnerID, SerialNumber)` to prevent duplicate serial records from entering the system
      - In MSSQL, a unique constraint creates a unique index underneath; both enforce uniqueness but a constraint is a logical declaration while an index is the physical implementation
      - Used filtered unique constraints on active records only so that deactivated records wouldn't conflict with new entries
6. Difference between primary key and unique key.
   - **Answer:**
      - A table has only one primary key, which is clustered by default in MSSQL
      - You can have multiple unique keys
      - Used primary keys for entity identity and unique keys for business-level uniqueness like partner codes
      - Choosing the wrong clustered index — on a wide unique key instead of a narrow primary key — degraded insert performance in the inventory batch load because wider keys increased page size and caused more page splits during sequential inserts
7. What is index?
   - **Answer:**
      - An index is a database structure that speeds up data retrieval
      - In CDMS, created composite indexes on columns used in WHERE and JOIN conditions of the slow stored procedures
      - This changed table scans to index seeks
      - Used the Database Engine Tuning Advisor recommendations alongside manual analysis to decide which indexes to create; the advisor suggests indexes based on workload analysis but manual review is needed to avoid over-indexing
8. How does indexing improve performance?
   - **Answer:**
      - Indexing creates a sorted data structure (B-tree in MSSQL) that allows the engine to locate rows without scanning the entire table
      - Saw this firsthand when adding a covering index reduced a 45-minute query in CDMS to under 30 seconds
      - The query optimizer chooses between index seek, scan, and lookup based on index structure, statistics density, and the cost model; a seek traverses the B-tree directly, a scan reads the full index, and a lookup fetches individual rows from the clustered index
9. What are the disadvantages of indexes?
   - **Answer:**
      - Indexes slow down INSERT, UPDATE, and DELETE operations because the index must be maintained
      - In the inventory system, had to balance read performance for validation queries against the write overhead during the batch ingestion of 10,000+ serial records
      - Monitored index fragmentation using `sys.dm_db_index_physical_stats` and set up weekly rebuild jobs to prevent performance degradation over time; fragmented indexes cause more I/O because pages are split across the disk
10. What is clustered index?
    - **Answer:**
       - A clustered index determines the physical order of data in a table
       - In MSSQL, the primary key creates a clustered index by default
       - In CDMS, chose the report date as the clustered index key for a large fact table to optimize range-based reporting queries
       - Choosing clustered indexes on incrementing columns (like identity columns) avoids page splits during inserts because new rows append to the end; observed page splits and insert hotspots during high-volume inserts in the inventory pipeline when the clustered key was non-sequential
11. What is non-clustered index?
    - **Answer:**
       - A non-clustered index is a separate structure that contains key values and pointers to the actual rows
       - Created non-clustered indexes on foreign key columns in CDMS to speed up JOIN operations between the partner and transaction tables
       - Include columns in non-clustered indexes can make them covering indexes; adding the SELECT list columns as INCLUDE columns eliminates the need to access the base table, which was used extensively to eliminate key lookups
12. What is composite index?
    - **Answer:**
       - A composite index is an index on multiple columns, with the column order determining its effectiveness
       - In CDMS, created a composite index on `(PartnerID, ReportDate)` because the slow procedure always filtered by partner first, then by date range
       - Column order matters — leading with the most selective column gives the best performance because the B-tree is sorted by the first column first; verified this by comparing estimated execution plans before and after reordering columns
13. What is covering index?
    - **Answer:**
       - A covering index includes all columns referenced by a query, eliminating the need to access the table data
       - In CDMS, rebuilt several non-clustered indexes with INCLUDE columns for the SELECT list
       - This eliminated expensive key lookups
       - Covering indexes are larger and increase maintenance cost since every INSERT/UPDATE must maintain the wider index; only created them for the most frequent queries where the performance gain justified the storage overhead
14. What is filtered index?
    - **Answer:**
       - A filtered index indexes only a subset of rows based on a WHERE condition
       - Used a filtered index on the inventory table where `Status = 'Active'` to keep the index small and fast for active-record lookups
       - Filtered indexes maintain better statistics for the relevant data subset since the statistics only cover indexed rows; this helped the optimizer choose better plans because the cardinality estimates were more accurate for active records
15. What is index selectivity?
    - **Answer:**
       - Selectivity measures how many rows match a given value — high selectivity means few rows per value
       - In CDMS, analyzed column selectivity to decide index order: PartnerID had high selectivity, so I put it first in composite indexes
       - Low-selectivity indexes (like on a boolean flag with only two distinct values) are often ignored by the optimizer because a table scan is cheaper than an index seek followed by thousands of key lookups; confirmed this through actual execution plan analysis where the optimizer chose scans over low-selectivity indexes
16. What is query execution plan?
    - **Answer:**
       - An execution plan shows how the database engine processes a query — which indexes it uses, how it joins tables, and where the cost is
       - In the CDMS optimization, spent the first week reading actual execution plans for each subquery in the stored procedure
       - Reading an actual plan involves looking for table scans, hash joins with large row estimates, and operators where estimated vs actual row counts diverge significantly — large divergence indicates stale statistics that need updating
17. How do you analyze an execution plan?
    - **Answer:**
       - Start with the most expensive operator (highest % of cost), check if it's a scan vs seek, look for missing index suggestions, and compare estimated vs actual rows
       - In CDMS, found a Cartesian product because a join predicate was missing — it showed as a massive nested loop with 100M+ rows
       - Enable actual execution plans in SSMS via `Ctrl+M` or the toolbar; use `SET STATISTICS TIME/IO ON` for detailed CPU and I/O metrics; always compare plans before and after changes to verify that optimizations actually improved the plan
18. What is table scan?
    - **Answer:**
       - A table scan reads every row in the table to find matching data
       - In CDMS, found table scans on a 5-million-row transaction table because there was no index on the join column
       - This was the primary reason the report took 5 hours
       - Table scans are acceptable for small tables (under ~10K rows) but disastrous for large ones; adding a single index on the join column converted the scan to a seek and reduced the query from hours to seconds
19. What is index seek?
    - **Answer:**
       - An index seek navigates the B-tree to find only the relevant rows, making it highly efficient
       - After adding the composite index on `(PartnerID, ReportDate)` in CDMS, the execution plan changed from a table scan to an index seek
       - Query time dropped from minutes to seconds
       - Seeks depend on sargable WHERE conditions; non-sargable functions like `YEAR(DateCol)` prevent index usage, so I rewrote them as range comparisons like `DateCol >= '2024-01-01' AND DateCol < '2025-01-01'` to make them seekable
20. What is index scan?
    - **Answer:**
       - An index scan reads all rows in an index (non-clustered) rather than the table
       - Faster than a table scan but slower than a seek
       - In the inventory system, saw index scans on a date index when queries didn't filter selectively enough
       - An index scan reads fewer pages than a table scan since the index is narrower (contains only indexed columns plus included columns), but it is still expensive for large tables because it traverses every leaf page of the index
21. What is key lookup?
    - **Answer:**
       - A key lookup happens when a non-clustered index doesn't cover the query, so the engine fetches the actual row from the clustered index
       - In CDMS, eliminated key lookups by adding INCLUDE columns to existing non-clustered indexes
       - Multiple key lookups per row compound into thousands of random I/O operations since each lookup requires a separate B-tree traversal to the clustered index; used the index tuning advisor to identify which lookups were most costly
22. What are joins?
    - **Answer:**
       - Joins combine rows from two or more tables based on related columns
       - In CDMS, optimized joins by ensuring both sides had proper indexes and that join predicates used the correct columns
       - One missing predicate had caused a cross join between a 500K and 200K row table
       - The three physical join operators are nested loop (best for small sets), hash match (best for large unsorted sets), and merge join (best for pre-sorted inputs); influenced the optimizer's choice through indexing and occasionally through query hints when the optimizer picked a suboptimal plan
23. Difference between inner join and left join.
    - **Answer:**
       - Inner join returns only matching rows from both tables
       - Left join returns all rows from the left table and matching from the right
       - In the inventory reconciliation, used left joins to find serial records that existed in the source but not in the target
       - Accidentally using a left join where an inner join sufficed caused incorrect row counts in CDMS reports because NULLs from unmatched right-table rows inflated aggregates; caught this by comparing before-and-after output
24. Difference between left join and right join.
    - **Answer:**
       - Left join preserves all left-table rows; right join preserves all right-table rows
       - Almost always use left joins for readability and consistency, since right joins can make query logic harder to follow
       - A right join inside a CTE caused confusing results in CDMS because the anchor table was on the right side, making the query read backwards; refactored it to left joins for clarity
25. Difference between full join and cross join.
    - **Answer:**
       - Full join returns all rows from both tables with matches where available
       - Cross join returns the Cartesian product of both tables
       - In a CDMS data validation script, accidentally wrote a cross join by omitting the ON clause, which produced 50M rows instead of 5K
       - Always inspect actual execution plans to catch accidental Cartesian products; the optimizer shows them clearly as nested loops with massively inflated row estimates
26. What is normalization?
    - **Answer:**
       - Normalization reduces data redundancy by splitting tables into related entities
       - In the inventory system, normalized partner data into separate tables (Partners, Products, Transactions) to avoid duplicate storage and update anomalies
       - Normalization helped data integrity but required careful indexing on foreign keys to maintain join performance; every foreign key column needed a non-clustered index to avoid scans during joins
27. What are normal forms?
    - **Answer:**
       - Normal forms are progressive rules to eliminate redundancy: 1NF removes repeating groups, 2NF removes partial dependencies, 3NF removes transitive dependencies
       - The CDMS schema was in 3NF, which I verified before designing indexes
       - Sometimes denormalized specific report tables for performance while keeping the transactional schema in 3NF; the reporting tables stored pre-computed aggregates to avoid expensive joins during dashboard rendering
28. What is denormalization?
    - **Answer:**
       - Denormalization intentionally adds redundancy for read performance
       - In CDMS, added computed columns for frequently calculated metrics so the stored procedures didn't have to compute them on the fly across millions of rows
       - The trade-off is that denormalization speeds up reads but complicates writes since redundant data must be kept in sync; only applied it to reporting tables, not transactional ones
29. When should you denormalize?
    - **Answer:**
       - Denormalize when read performance is critical and the data is mostly static, like pre-aggregated reporting tables
       - In CDMS, denormalized monthly partner summaries into a separate table that the dashboard queried instead of aggregating raw data each time
       - Indexed views in MSSQL serve as a middle ground — they look like views but are physically stored and maintained by the engine; the engine automatically updates them when underlying data changes, giving denormalization benefits without manual ETL
30. What is stored procedure?
    - **Answer:**
       - A stored procedure is a pre-compiled batch of SQL statements stored in the database
       - The CDMS reporting system relied entirely on stored procedures
       - Optimized the main one from 5 hours to 12 minutes by rewriting queries and fixing indexes
       - Profiled each section of the procedure using `SET STATISTICS TIME ON` to find per-statement CPU and elapsed time; broke it down to find the individual expensive queries rather than guessing
31. Advantages and disadvantages of stored procedures.
    - **Answer:**
       - Advantages: pre-compiled execution plans, reduced network traffic, centralized business logic
       - Disadvantages: harder to version control, debug, and test compared to application code
       - In CDMS, felt this pain when I couldn't easily diff procedure versions
       - My approach: keep reporting logic in procedures where set-based operations are fast; keep business validation in application code where it is testable with unit tests and easier to version control
32. What is function in SQL?
    - **Answer:**
       - A function returns a scalar value or table
       - In CDMS, used scalar functions for reusable logic like date formatting
       - Later replaced them with inline table-valued functions because scalar functions caused row-by-row execution
       - Scalar functions in WHERE clauses make queries non-sargable because the engine must evaluate the function for every row; converting them to inline TVFs eliminated the performance issue since inline TVFs are expanded into the query like a view
33. Difference between stored procedure and function.
    - **Answer:**
       - Functions must return a value and cannot modify data
       - Stored procedures can modify data and have side effects
       - Used stored procedures for ETL operations and functions for computed columns or reusable lookups
       - Functions in FROM clauses can be inline (optimized like views, expanded into the query) or multi-statement (materialized into temp tables with a defined return type); prefer inline TVFs for performance since the optimizer can push predicates into them
34. What is trigger?
    - **Answer:**
       - A trigger runs automatically on INSERT, UPDATE, or DELETE
       - In the inventory system, used an AFTER INSERT trigger on the serial records table to update the partner's last-activity timestamp automatically
       - Complex triggers cause cascading issues — once debugged a trigger chain where an INSERT trigger fired another INSERT which fired an UPDATE, locking the inventory table for minutes during bulk inserts
35. What is view?
    - **Answer:**
       - A view is a saved SQL query that looks like a table
       - In CDMS, created views for frequently joined table combinations so report developers didn't have to remember complex join conditions
       - Views can hide complexity but also hide performance problems; a view joining 10 tables looks simple to the caller but runs all 10 joins every time it is queried
36. What is materialized view?
    - **Answer:**
       - In MSSQL, indexed views persist the result set physically, updated automatically
       - Used an indexed view for the monthly partner sales summary in CDMS
       - Kept the dashboard queries fast without manual ETL
       - Indexed views require SCHEMABINDING and cannot use DISTINCT in aggregates; worked around this by using `COUNT_BIG` instead of `COUNT` since `COUNT_BIG` is allowed with SCHEMABINDING
37. What is transaction?
    - **Answer:**
       - A transaction groups multiple operations into an atomic unit — all succeed or all roll back
       - In the inventory validation pipeline, wrapped the batch insert of 10,000 serial records in a transaction
       - Ensured partial failures didn't corrupt the inventory state
       - Transaction scope was kept short to avoid holding locks; handled deadlocks when concurrent batches ran by catching the 1205 error and retrying with a backoff strategy
38. What are ACID properties?
    - **Answer:**
       - Atomicity (all-or-nothing), Consistency (valid state before and after), Isolation (concurrent transactions don't interfere), Durability (committed data survives failures)
       - MSSQL's ACID compliance was why we chose it for inventory — couldn't lose or corrupt serial-level data
       - Isolation levels trade strictness for performance; used snapshot isolation in CDMS for read-only reporting to avoid writer blocking, giving readers a consistent point-in-time snapshot without holding shared locks
39. What is isolation level?
    - **Answer:**
       - Isolation level controls how transactions see each other's changes
       - In CDMS reporting, used READ UNCOMMITTED (with NOLOCK hints) on the read-only report queries
       - Slight dirty reads were acceptable and we needed zero blocking on writes
       - The spectrum runs from READ UNCOMMITTED (dirty reads possible, no shared locks) to SERIALIZABLE (full isolation, range locks, no concurrency); chose based on whether accuracy or speed mattered for each query
40. Difference between read committed and repeatable read.
    - **Answer:**
       - Read committed prevents dirty reads but allows non-repeatable reads
       - Repeatable read prevents both
       - In inventory reconciliation, used repeatable read to ensure two successive reads of the same baseline data matched during validation
       - Repeatable read holds shared locks until the transaction ends, which increased blocking but was necessary for reconciliation accuracy
41. What is dirty read?
    - **Answer:**
       - A dirty read occurs when a transaction reads uncommitted data from another transaction
       - Allowed dirty reads in the CDMS dashboard (via NOLOCK hints) because stale data by a few seconds was acceptable for monitoring
       - Never used in the inventory validation engine
       - NOLOCK can also cause missed or double-read rows due to page splits moving rows between reads; this is why I avoided it for any financial or count-based reports
42. What is non-repeatable read?
    - **Answer:**
       - A non-repeatable read happens when a row changes between two reads in the same transaction
       - In the inventory pipeline, encountered this when the baseline data was being updated by another process during validation
       - Switched to snapshot isolation
       - Different isolation levels handle this differently: READ COMMITTED allows it (re-reads get the latest committed version), REPEATABLE READ prevents it by holding shared locks, and snapshot isolation gives consistency without blocking by reading from the version store
43. What is phantom read?
    - **Answer:**
       - A phantom read occurs when new rows appear between two reads in the same transaction
       - In CDMS, phantom reads didn't affect aggregate reports significantly
       - Stayed at READ COMMITTED for most reporting
       - SERIALIZABLE prevents phantoms by acquiring range locks on index key ranges, and snapshot isolation prevents them through row versioning; both have performance costs, so the choice depends on whether phantom consistency is required
44. What is deadlock?
    - **Answer:**
       - A deadlock occurs when two transactions each hold a lock the other needs, and SQL Server chooses a victim
       - In the inventory system, concurrent batch ingestion processes occasionally deadlocked on the serial records table
       - One process was killed automatically
       - Reduced deadlocks by ensuring all transactions accessed tables in the same order and keeping transaction durations short to minimize the lock hold window
45. How do you prevent deadlocks?
    - **Answer:**
       - Prevent deadlocks by accessing tables in a consistent order across transactions, keeping transactions short, and using appropriate isolation levels
       - In the inventory pipeline, also used row versioning to reduce lock contention
       - Used SQL Server Profiler to capture deadlock graphs (or Extended Events in newer versions), then visualized them with deadlock graph viewers to understand which queries and objects were involved and where the lock cycle occurred
46. What is locking?
    - **Answer:**
       - Locking controls concurrent access to data
       - MSSQL uses row, page, and table locks based on the operation
       - In CDMS, saw lock escalation from row to table locks during bulk report generation, which blocked other queries until I broke the batch into smaller chunks
       - The lock hierarchy goes row → page → table; lock escalation happens when a single transaction accumulates too many row or page locks on one table, and MSSQL escalates to a table lock; monitored locks using `sys.dm_tran_locks` and `sys.dm_exec_requests`
47. What is row lock?
    - **Answer:**
       - A row lock locks a single row to allow maximum concurrency
       - In the inventory system, row locks allowed multiple partner feeds to be processed concurrently on different serial records without blocking each other
       - Row locks consume memory for each lock resource and can escalate to table locks when a transaction accumulates too many; checked lock granularity using execution plans and `sys.dm_tran_locks`
48. What is table lock?
    - **Answer:**
       - A table lock locks the entire table, preventing any concurrent access
       - In CDMS, the original stored procedure escalated to table locks on the transactions table during the long-running report
       - This blocked all incoming data feeds
       - Used `ALTER TABLE ... SET LOCK_ESCALATION = AUTO` to control escalation behavior; also partitioned large tables to limit lock escalation scope to individual partitions rather than the entire table
49. What is optimistic locking?
    - **Answer:**
       - Optimistic locking assumes conflicts are rare and checks at commit time using a version column
       - Used a rowversion column in the inventory entity to detect concurrent modifications during reconciliation without holding long locks
       - If a `DbUpdateConcurrencyException` occurs in the application layer, handle it by refreshing the data, merging changes, and retrying the operation rather than blocking readers during the entire transaction
50. What is pessimistic locking?
    - **Answer:**
       - Pessimistic locking assumes conflicts are likely and locks resources upfront using `WITH (UPDLOCK)` or `SELECT ... FOR UPDATE`
       - Used UPDLOCK hints in the critical section of the inventory allocation to prevent double-allocation of the same serial number
       - Pessimistic locking reduces throughput because locks are held for the entire transaction, but it guarantees correctness; was acceptable for low-concurrency critical operations like serial allocation
51. What is connection pooling?
    - **Answer:**
       - Connection pooling reuses database connections instead of creating new ones per request
       - In the Spring Boot application, configured HikariCP with a max pool size of 20
       - Tuned based on the number of concurrent API requests and database capacity
       - Diagnosed connection exhaustion by monitoring active connections in MSSQL using `sys.dm_exec_sessions`; increased the pool size gradually while watching for contention and `ConnectionIsTimeout` errors
52. How do you optimize a slow query?
    - **Answer:**
       - Start by examining the actual execution plan for table scans, high-cost operators, and missing index hints
       - Then check indexes, rewrite non-sargable conditions, and add INCLUDE columns
       - In CDMS, systematically applied this to each subquery of the 5-hour procedure
       - Full playbook: enable `SET STATISTICS TIME/IO ON`, check wait stats for I/O or CPU bottlenecks, look for parameter sniffing issues (where a cached plan is optimal for one parameter value but terrible for another), and test with `OPTION (RECOMPILE)` if parameter sniffing is the root cause
53. How do you optimize a stored procedure?
    - **Answer:**
       - Profile the procedure by running it with `SET STATISTICS TIME ON` and capturing the actual plan
       - Identify the most expensive statements, optimize them individually
       - Re-run the whole procedure to confirm the cumulative improvement
       - That's exactly how CDMS was brought from 5 hours to 12 minutes
       - Parameterized the procedure to prevent plan caching issues where different parameter values produce vastly different row counts; used `WITH RECOMPILE` for procedures with highly varying parameters so the engine generates a fresh plan per execution
54. How do you handle large data processing?
    - **Answer:**
       - Use batch processing with set-based operations, avoid cursors and row-by-row processing, and ensure proper indexing
       - In the inventory system, processed 10,000+ serial records in batches of 1,000 using `INSERT ... SELECT` with `ROWNUMBER()` filtering
       - Batch size tuning matters: too small causes many round-trips and transaction log overhead, too large causes lock escalation and memory pressure; based on load testing, found 1,000-row batches hit the sweet spot for the inventory table size
55. What is pagination in SQL?
    - **Answer:**
       - Pagination retrieves a subset of rows using `OFFSET` and `FETCH NEXT` in MSSQL
       - In the CDMS partner list API, paginated results with `ORDER BY PartnerID OFFSET 0 ROWS FETCH NEXT 50 ROWS ONLY`
       - Avoided loading thousands of partners at once
       - Offset pagination degrades with large offsets because the engine still reads and discards all skipped rows; for page 10,000 with a page size of 50, it reads 500,000 rows just to skip them — switched to keyset pagination when this became a problem
56. Difference between offset pagination and keyset pagination.
    - **Answer:**
       - Offset pagination skips N rows using `OFFSET`
       - Keyset pagination uses `WHERE id > @lastSeenId`
       - Offset pagination becomes slow on large datasets because it reads all skipped rows each time
       - In the inventory audit log, switched to keyset pagination for faster queries
       - Keyset pagination is ideal for infinite scroll on sorted unique columns but cannot skip to arbitrary pages; offset pagination supports arbitrary page access but pays the cost of scanning skipped rows
57. What is query parameterization?
    - **Answer:**
       - Parameterization uses placeholders instead of hardcoded values in SQL, which allows plan reuse and prevents SQL injection
       - In the Spring Boot JPA repositories, all queries were automatically parameterized via prepared statements
       - Ad-hoc queries without parameterization caused plan cache bloat in CDMS because each unique literal value generated a separate cached plan; forced parameterization using `ALTER DATABASE SET PARAMETERIZATION FORCED` to group similar queries under one plan
58. What is SQL injection?
    - **Answer:**
       - SQL injection is an attack where malicious SQL is inserted through user input
       - In the inventory API, prevented this by never concatenating user input into SQL strings
       - All queries used JPA repositories or parameterized stored procedures
       - String concatenation like `"WHERE name = '" + userInput + "'"` is vulnerable because the user can close the quote and inject additional SQL; parameterized queries like `WHERE name = @name` prevent it because the parameter value is treated as data, never parsed as SQL
59. How do you prevent SQL injection?
    - **Answer:**
       - Always use parameterized queries, stored procedures, or ORM frameworks that handle escaping
       - In all projects, strictly use Spring Data JPA or `@Query` with named parameters
       - Validate inputs at the controller layer before they reach the database
       - Stored procedures with `EXEC` or `sp_executesql` built from concatenated strings inside the procedure can still be vulnerable; never build dynamic SQL inside procedures — instead use CASE statements or multiple queries to handle conditional logic
60. What is ETL?
    - **Answer:**
       - ETL (Extract, Transform, Load) is the process of moving data from source systems to a target database
       - In CDMS, worked on an SSIS-based ETL pipeline that extracted partner data from various formats
       - Transformed it (cleaning, validating, aggregating) and loaded it into MSSQL reporting tables
       - Specific flow: extracted CSV files from partner uploads, transformed serial numbers to a standardized format (trimming whitespace, validating against regex patterns), and loaded them into the inventory tables with row-level error logging for any records that failed validation
61. What is SSIS?
    - **Answer:**
       - SSIS (SQL Server Integration Services) is Microsoft's ETL tool for data migration and integration
       - In CDMS, used SSIS packages to automate the nightly data load from partner files into the staging and reporting tables
       - Used SSIS data flow tasks with lookup transformations to validate partner codes against the reference table; redirected invalid rows to an error output file so the main flow continued while invalid records were logged for partner follow-up
62. What is data validation in ETL?
    - **Answer:**
       - Data validation in ETL ensures that transformed data meets business rules before loading
       - In the inventory ETL, validated serial number formats, partner codes against the master list, and date ranges before inserting into the target table
       - Three validation layers: schema validation (data types, lengths, required fields), business rule validation (serial format matches regex, dates within valid range), and referential integrity checks (partner ID exists in the master table); logged each failure type separately for partner reporting
63. How do you handle duplicate records?
    - **Answer:**
       - Use `ROW_NUMBER()` with `PARTITION BY` to identify duplicates, then decide whether to deduplicate (keep first/latest) or reject
       - In the inventory pipeline, partitioned by serial number and kept the record with the latest timestamp
       - Logged the rejected duplicates for partner notification
       - The business trade-off: sometimes duplicates should be rejected (unique serials must remain unique), sometimes merged (duplicate partner submissions should be combined); depends on the data domain and whether the duplicates represent true errors or valid overlapping submissions
64. How do you handle missing data?
    - **Answer:**
       - For missing data, first check if it's truly required or can be defaulted
       - In the inventory ETL, if a serial record was missing the partner ID, logged it to an error table and skipped it
       - Couldn't determine ownership without the partner
       - NULL handling strategies differ: `COALESCE` provides a default fallback value, `ISNULL` checks for NULL before processing; missing required fields should fail fast rather than propagate bad data downstream, while missing optional fields can be safely defaulted
65. How do you reconcile source and target data?
    - **Answer:**
       - Reconciliation compares row counts, checksums, or key values between source and target after ETL
       - In CDMS, built a reconciliation script using `EXCEPT` and `INTERSECT` operators to find mismatches between the source partner files and the loaded reporting tables
       - Automated reconciliation as the final step of the ETL pipeline; sent an email with the mismatch count so the team could investigate before downstream reports were generated
