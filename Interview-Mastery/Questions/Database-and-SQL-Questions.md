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
   - **If asked more:**
      - Would explain how I analyzed wait statistics in MSSQL to identify where the DBMS was spending most of its time during the 5-hour stored procedure runs
2. What is RDBMS?
   - **Answer:**
      - An RDBMS stores data in related tables using keys and indexes
      - MSSQL is the RDBMS used at Talentpace, where I designed normalized schemas for inventory data
      - Foreign keys linked partners, products, and transactions
   - **If asked more:**
      - Would contrast RDBMS with NoSQL and explain why we chose MSSQL — strong ACID compliance was critical for inventory accuracy and reporting
3. What is primary key?
   - **Answer:**
      - A primary key uniquely identifies each row in a table
      - In the inventory system, used `SerialNumber` as the primary key to ensure every serial record was unique
      - Could be traced back to a specific partner shipment
   - **If asked more:**
      - Would discuss how primary key choice affects clustered index performance
      - Would explain why I prefer narrow integer keys over wide composite keys in high-volume tables
4. What is foreign key?
   - **Answer:**
      - A foreign key links two tables and enforces referential integrity
      - In CDMS, used foreign keys to ensure each report record pointed to a valid partner ID
      - Prevented orphan data when partners were deactivated
   - **If asked more:**
      - Would explain how missing foreign key indexes caused nested loop joins in the 5-hour procedure
      - Adding them was one of my first optimization steps
5. What is unique key?
   - **Answer:**
      - A unique key ensures all values in a column or column set are distinct
      - In the inventory validation engine, applied a unique constraint on `(PartnerID, SerialNumber)` to prevent duplicate serial records from entering the system
   - **If asked more:**
      - Would explain the difference between unique constraint and unique index in MSSQL
      - Would explain how I used filtered unique constraints for active records only
6. Difference between primary key and unique key.
   - **Answer:**
      - A table has only one primary key, which is clustered by default in MSSQL
      - You can have multiple unique keys
      - Used primary keys for entity identity and unique keys for business-level uniqueness like partner codes
   - **If asked more:**
      - Would talk about how choosing the wrong clustered index (on a wide unique key instead of a narrow primary key) degraded insert performance in the inventory batch load
7. What is index?
   - **Answer:**
      - An index is a database structure that speeds up data retrieval
      - In CDMS, created composite indexes on columns used in WHERE and JOIN conditions of the slow stored procedures
      - This changed table scans to index seeks
   - **If asked more:**
      - Would explain how I used the Database Engine Tuning Advisor recommendations alongside manual analysis to decide which indexes to create
8. How does indexing improve performance?
   - **Answer:**
      - Indexing creates a sorted data structure (B-tree in MSSQL) that allows the engine to locate rows without scanning the entire table
      - Saw this firsthand when adding a covering index reduced a 45-minute query in CDMS to under 30 seconds
   - **If asked more:**
      - Would explain how the query optimizer chooses between index seek, scan, and lookup based on index structure and statistics
9. What are the disadvantages of indexes?
   - **Answer:**
      - Indexes slow down INSERT, UPDATE, and DELETE operations because the index must be maintained
      - In the inventory system, had to balance read performance for validation queries against the write overhead during the batch ingestion of 10,000+ serial records
   - **If asked more:**
      - Would discuss how I monitored index fragmentation and set up weekly rebuild jobs to prevent performance degradation over time
10. What is clustered index?
    - **Answer:**
       - A clustered index determines the physical order of data in a table
       - In MSSQL, the primary key creates a clustered index by default
       - In CDMS, chose the report date as the clustered index key for a large fact table to optimize range-based reporting queries
    - **If asked more:**
       - Would warn about choosing clustered indexes on incrementing columns to avoid page splits
       - Observed this during high-volume inserts in the inventory pipeline
11. What is non-clustered index?
    - **Answer:**
       - A non-clustered index is a separate structure that contains key values and pointers to the actual rows
       - Created non-clustered indexes on foreign key columns in CDMS to speed up JOIN operations between the partner and transaction tables
    - **If asked more:**
       - Would explain how include columns in non-clustered indexes can make them covering indexes
       - Used this extensively to eliminate key lookups
12. What is composite index?
    - **Answer:**
       - A composite index is an index on multiple columns, with the column order determining its effectiveness
       - In CDMS, created a composite index on `(PartnerID, ReportDate)` because the slow procedure always filtered by partner first, then by date range
    - **If asked more:**
       - Would explain how column order matters — leading with the most selective column gives the best performance
       - Verified this by comparing estimated execution plans
13. What is covering index?
    - **Answer:**
       - A covering index includes all columns referenced by a query, eliminating the need to access the table data
       - In CDMS, rebuilt several non-clustered indexes with INCLUDE columns for the SELECT list
       - This eliminated expensive key lookups
    - **If asked more:**
       - Would discuss the trade-off — covering indexes are larger and increase maintenance cost
       - Only created them for the most frequent queries
14. What is filtered index?
    - **Answer:**
       - A filtered index indexes only a subset of rows based on a WHERE condition
       - Used a filtered index on the inventory table where `Status = 'Active'` to keep the index small and fast for active-record lookups
    - **If asked more:**
       - Would explain how filtered indexes also maintain better statistics for the relevant data subset
       - This helped the optimizer choose better plans
15. What is index selectivity?
    - **Answer:**
       - Selectivity measures how many rows match a given value — high selectivity means few rows per value
       - In CDMS, analyzed column selectivity to decide index order: PartnerID had high selectivity, so I put it first in composite indexes
    - **If asked more:**
       - Would explain how low-selectivity indexes (like on a boolean flag) are often ignored by the optimizer
       - Confirmed this through actual execution plan analysis
16. What is query execution plan?
    - **Answer:**
       - An execution plan shows how the database engine processes a query — which indexes it uses, how it joins tables, and where the cost is
       - In the CDMS optimization, spent the first week reading actual execution plans for each subquery in the stored procedure
    - **If asked more:**
       - Would walk through how I read an actual plan: looking for table scans, hash joins with large row estimates
       - Would explain operators with high estimated vs actual row counts indicating stale statistics
17. How do you analyze an execution plan?
    - **Answer:**
       - Start with the most expensive operator (highest % of cost), check if it's a scan vs seek, look for missing index suggestions, and compare estimated vs actual rows
       - In CDMS, found a Cartesian product because a join predicate was missing — it showed as a massive nested loop with 100M+ rows
    - **If asked more:**
       - Would explain how I enable actual execution plans in SSMS
       - Would explain using SET STATISTICS TIME/IO ON for detailed metrics
       - Would explain comparing plans before and after changes
18. What is table scan?
    - **Answer:**
       - A table scan reads every row in the table to find matching data
       - In CDMS, found table scans on a 5-million-row transaction table because there was no index on the join column
       - This was the primary reason the report took 5 hours
    - **If asked more:**
       - Would explain that table scans are acceptable for small tables but disastrous for large ones
       - Adding a single index converted the scan to a seek
19. What is index seek?
    - **Answer:**
       - An index seek navigates the B-tree to find only the relevant rows, making it highly efficient
       - After adding the composite index on `(PartnerID, ReportDate)` in CDMS, the execution plan changed from a table scan to an index seek
       - Query time dropped from minutes to seconds
    - **If asked more:**
       - Would explain how seeks depend on sargable WHERE conditions
       - Would explain how I rewrote non-sargable functions like `YEAR(DateCol)` to range comparisons
20. What is index scan?
    - **Answer:**
       - An index scan reads all rows in an index (non-clustered) rather than the table
       - Faster than a table scan but slower than a seek
       - In the inventory system, saw index scans on a date index when queries didn't filter selectively enough
    - **If asked more:**
       - Would explain the difference between table scan and index scan — index scan reads fewer pages since the index is narrower
       - Still expensive for large tables
21. What is key lookup?
    - **Answer:**
       - A key lookup happens when a non-clustered index doesn't cover the query, so the engine fetches the actual row from the clustered index
       - In CDMS, eliminated key lookups by adding INCLUDE columns to existing non-clustered indexes
    - **If asked more:**
       - Would explain how multiple key lookups per row compound into thousands of random I/O operations
       - Used the index tuning advisor to identify them
22. What are joins?
    - **Answer:**
       - Joins combine rows from two or more tables based on related columns
       - In CDMS, optimized joins by ensuring both sides had proper indexes and that join predicates used the correct columns
       - One missing predicate had caused a cross join between a 500K and 200K row table
    - **If asked more:**
       - Would explain the three physical join operators (nested loop, hash match, merge join)
       - Would explain how I influenced the optimizer's choice through indexing and query hints
23. Difference between inner join and left join.
    - **Answer:**
       - Inner join returns only matching rows from both tables
       - Left join returns all rows from the left table and matching from the right
       - In the inventory reconciliation, used left joins to find serial records that existed in the source but not in the target
    - **If asked more:**
       - Would explain how accidentally using a left join where an inner join sufficed caused incorrect row counts in CDMS reports
       - Caught this by comparing before-and-after output
24. Difference between left join and right join.
    - **Answer:**
       - Left join preserves all left-table rows; right join preserves all right-table rows
       - Almost always use left joins for readability and consistency, since right joins can make query logic harder to follow
    - **If asked more:**
       - Would share a real debugging story where a right join inside a CTE caused confusing results in CDMS
       - Refactored it to left joins for clarity
25. Difference between full join and cross join.
    - **Answer:**
       - Full join returns all rows from both tables with matches where available
       - Cross join returns the Cartesian product of both tables
       - In a CDMS data validation script, accidentally wrote a cross join by omitting the ON clause, which produced 50M rows instead of 5K
    - **If asked more:**
       - Would explain how I now always inspect actual execution plans to catch accidental Cartesian products
       - The optimizer shows them clearly
26. What is normalization?
    - **Answer:**
       - Normalization reduces data redundancy by splitting tables into related entities
       - In the inventory system, normalized partner data into separate tables (Partners, Products, Transactions) to avoid duplicate storage and update anomalies
    - **If asked more:**
       - Would discuss how normalization helped data integrity but required careful indexing on foreign keys to maintain join performance
27. What are normal forms?
    - **Answer:**
       - Normal forms are progressive rules to eliminate redundancy: 1NF removes repeating groups, 2NF removes partial dependencies, 3NF removes transitive dependencies
       - The CDMS schema was in 3NF, which I verified before designing indexes
    - **If asked more:**
       - Would explain how I sometimes denormalized specific report tables for performance while keeping the transactional schema in 3NF
28. What is denormalization?
    - **Answer:**
       - Denormalization intentionally adds redundancy for read performance
       - In CDMS, added computed columns for frequently calculated metrics so the stored procedures didn't have to compute them on the fly across millions of rows
    - **If asked more:**
       - Would explain the trade-off: denormalization speeds up reads but complicates writes
       - Only applied it to reporting tables, not transactional ones
29. When should you denormalize?
    - **Answer:**
       - Denormalize when read performance is critical and the data is mostly static, like pre-aggregated reporting tables
       - In CDMS, denormalized monthly partner summaries into a separate table that the dashboard queried instead of aggregating raw data each time
    - **If asked more:**
       - Would explain how I used indexed views in MSSQL as a middle ground — they look like views but are physically stored and maintained by the engine
30. What is stored procedure?
    - **Answer:**
       - A stored procedure is a pre-compiled batch of SQL statements stored in the database
       - The CDMS reporting system relied entirely on stored procedures
       - Optimized the main one from 5 hours to 12 minutes by rewriting queries and fixing indexes
    - **If asked more:**
       - Would explain how I profiled each section of the procedure using SET STATISTICS TIME
       - Broke it down to find the individual expensive queries
31. Advantages and disadvantages of stored procedures.
    - **Answer:**
       - Advantages: pre-compiled execution plans, reduced network traffic, centralized business logic
       - Disadvantages: harder to version control, debug, and test compared to application code
       - In CDMS, felt this pain when I couldn't easily diff procedure versions
    - **If asked more:**
       - Would explain my approach: keep reporting logic in procedures (where set-based operations are fast)
       - Keep business validation in application code (where it's testable)
32. What is function in SQL?
    - **Answer:**
       - A function returns a scalar value or table
       - In CDMS, used scalar functions for reusable logic like date formatting
       - Later replaced them with inline table-valued functions because scalar functions caused row-by-row execution
    - **If asked more:**
       - Would explain how scalar functions in WHERE clauses made queries non-sargable
       - Converting them to inline TVFs eliminated the performance issue
33. Difference between stored procedure and function.
    - **Answer:**
       - Functions must return a value and cannot modify data
       - Stored procedures can modify data and have side effects
       - Used stored procedures for ETL operations and functions for computed columns or reusable lookups
    - **If asked more:**
       - Would explain that functions in FROM clauses can be inline (optimized like views) or multi-statement (materialized temp tables)
       - Prefer inline for performance
34. What is trigger?
    - **Answer:**
       - A trigger runs automatically on INSERT, UPDATE, or DELETE
       - In the inventory system, used an AFTER INSERT trigger on the serial records table to update the partner's last-activity timestamp automatically
    - **If asked more:**
       - Would warn against complex triggers that cause cascading issues
       - Once debugged a trigger chain that locked the inventory table for minutes during bulk inserts
35. What is view?
    - **Answer:**
       - A view is a saved SQL query that looks like a table
       - In CDMS, created views for frequently joined table combinations so report developers didn't have to remember complex join conditions
    - **If asked more:**
       - Would explain how views can hide complexity but also hide performance problems
       - A view joining 10 tables looks simple but runs like 10 tables
36. What is materialized view?
    - **Answer:**
       - In MSSQL, indexed views persist the result set physically, updated automatically
       - Used an indexed view for the monthly partner sales summary in CDMS
       - Kept the dashboard queries fast without manual ETL
    - **If asked more:**
       - Would explain the restrictions on indexed views (must use SCHEMABINDING, no DISTINCT in aggregates)
       - Worked around them by using COUNT_BIG instead of COUNT
37. What is transaction?
    - **Answer:**
       - A transaction groups multiple operations into an atomic unit — all succeed or all roll back
       - In the inventory validation pipeline, wrapped the batch insert of 10,000 serial records in a transaction
       - Ensured partial failures didn't corrupt the inventory state
    - **If asked more:**
       - Would explain how I chose transaction scope — keeping it short to avoid holding locks
       - Would explain how I handled deadlocks when concurrent batches ran
38. What are ACID properties?
    - **Answer:**
       - Atomicity (all-or-nothing), Consistency (valid state before and after), Isolation (concurrent transactions don't interfere), Durability (committed data survives failures)
       - MSSQL's ACID compliance was why we chose it for inventory — couldn't lose or corrupt serial-level data
    - **If asked more:**
       - Would explain how isolation levels trade strictness for performance
       - Would explain how I used snapshot isolation in CDMS for read-only reporting to avoid writer blocking
39. What is isolation level?
    - **Answer:**
       - Isolation level controls how transactions see each other's changes
       - In CDMS reporting, used READ UNCOMMITTED (with NOLOCK hints) on the read-only report queries
       - Slight dirty reads were acceptable and we needed zero blocking on writes
    - **If asked more:**
       - Would explain the spectrum from READ UNCOMMITTED (dirty reads possible) to SERIALIZABLE (no concurrency)
       - Would explain how I chose based on whether accuracy or speed mattered
40. Difference between read committed and repeatable read.
    - **Answer:**
       - Read committed prevents dirty reads but allows non-repeatable reads
       - Repeatable read prevents both
       - In inventory reconciliation, used repeatable read to ensure two successive reads of the same baseline data matched during validation
    - **If asked more:**
       - Would explain how repeatable read holds shared locks until the transaction ends
       - Increased blocking but was necessary for reconciliation accuracy
41. What is dirty read?
    - **Answer:**
       - A dirty read occurs when a transaction reads uncommitted data from another transaction
       - Allowed dirty reads in the CDMS dashboard (via NOLOCK hints) because stale data by a few seconds was acceptable for monitoring
       - Never used in the inventory validation engine
    - **If asked more:**
       - Would explain how NOLOCK can also cause missed or double-read rows due to page splits
       - This is why I avoided it for any financial or count-based reports
42. What is non-repeatable read?
    - **Answer:**
       - A non-repeatable read happens when a row changes between two reads in the same transaction
       - In the inventory pipeline, encountered this when the baseline data was being updated by another process during validation
       - Switched to snapshot isolation
    - **If asked more:**
       - Would compare how different isolation levels handle this
       - Would explain why snapshot isolation gives consistency without blocking
43. What is phantom read?
    - **Answer:**
       - A phantom read occurs when new rows appear between two reads in the same transaction
       - In CDMS, phantom reads didn't affect aggregate reports significantly
       - Stayed at READ COMMITTED for most reporting
    - **If asked more:**
       - Would explain how SERIALIZABLE or snapshot isolation prevents phantoms by locking ranges or using row versioning
44. What is deadlock?
    - **Answer:**
       - A deadlock occurs when two transactions each hold a lock the other needs, and SQL Server chooses a victim
       - In the inventory system, concurrent batch ingestion processes occasionally deadlocked on the serial records table
       - One process was killed automatically
    - **If asked more:**
       - Would explain how I reduced deadlocks by ensuring all transactions accessed tables in the same order
       - Kept transaction durations short
45. How do you prevent deadlocks?
    - **Answer:**
       - Prevent deadlocks by accessing tables in a consistent order across transactions, keeping transactions short, and using appropriate isolation levels
       - In the inventory pipeline, also used row versioning to reduce lock contention
    - **If asked more:**
       - Would explain how I used SQL Server Profiler to capture deadlock graphs
       - Visualized them to understand which queries and objects were involved
46. What is locking?
    - **Answer:**
       - Locking controls concurrent access to data
       - MSSQL uses row, page, and table locks based on the operation
       - In CDMS, saw lock escalation from row to table locks during bulk report generation, which blocked other queries until I broke the batch into smaller chunks
    - **If asked more:**
       - Would explain the lock hierarchy and how lock escalation works
       - Would explain how I monitored locks with `sys.dm_tran_locks`
47. What is row lock?
    - **Answer:**
       - A row lock locks a single row to allow maximum concurrency
       - In the inventory system, row locks allowed multiple partner feeds to be processed concurrently on different serial records without blocking each other
    - **If asked more:**
       - Would explain how row locks consume memory and can escalate
       - Would explain how I checked lock granularity using execution plans
48. What is table lock?
    - **Answer:**
       - A table lock locks the entire table, preventing any concurrent access
       - In CDMS, the original stored procedure escalated to table locks on the transactions table during the long-running report
       - This blocked all incoming data feeds
    - **If asked more:**
       - Would explain how I used `ALTER TABLE ... SET LOCK_ESCALATION = AUTO`
       - Would explain how I partitioned large tables to limit lock escalation scope
49. What is optimistic locking?
    - **Answer:**
       - Optimistic locking assumes conflicts are rare and checks at commit time using a version column
       - Used a rowversion column in the inventory entity to detect concurrent modifications during reconciliation without holding long locks
    - **If asked more:**
       - Would explain how I handled the `DbUpdateConcurrencyException` in the application layer by retrying the operation after refreshing the data
50. What is pessimistic locking?
    - **Answer:**
       - Pessimistic locking assumes conflicts are likely and locks resources upfront using `WITH (UPDLOCK)` or `SELECT ... FOR UPDATE`
       - Used UPDLOCK hints in the critical section of the inventory allocation to prevent double-allocation of the same serial number
    - **If asked more:**
       - Would explain the trade-off: pessimistic locking reduces throughput but guarantees correctness
       - Was acceptable for low-concurrency critical operations
51. What is connection pooling?
    - **Answer:**
       - Connection pooling reuses database connections instead of creating new ones per request
       - In the Spring Boot application, configured HikariCP with a max pool size of 20
       - Tuned based on the number of concurrent API requests and database capacity
    - **If asked more:**
       - Would explain how I diagnosed connection exhaustion by monitoring active connections in MSSQL
       - Increased the pool size gradually while watching for contention
52. How do you optimize a slow query?
    - **Answer:**
       - Start by examining the actual execution plan for table scans, high-cost operators, and missing index hints
       - Then check indexes, rewrite non-sargable conditions, and add INCLUDE columns
       - In CDMS, systematically applied this to each subquery of the 5-hour procedure
    - **If asked more:**
       - Would explain my full playbook: SET STATISTICS TIME/IO ON, check wait stats
       - Would explain looking for parameter sniffing issues and testing with `OPTION (RECOMPILE)` if needed
53. How do you optimize a stored procedure?
    - **Answer:**
       - Profile the procedure by running it with `SET STATISTICS TIME ON` and capturing the actual plan
       - Identify the most expensive statements, optimize them individually
       - Re-run the whole procedure to confirm the cumulative improvement
       - That's exactly how CDMS was brought from 5 hours to 12 minutes
    - **If asked more:**
       - Would explain how I parameterized the procedure to prevent plan caching issues
       - Would explain how I used `WITH RECOMPILE` for procedures with highly varying parameters
54. How do you handle large data processing?
    - **Answer:**
       - Use batch processing with set-based operations, avoid cursors and row-by-row processing, and ensure proper indexing
       - In the inventory system, processed 10,000+ serial records in batches of 1,000 using `INSERT ... SELECT` with `ROWNUMBER()` filtering
    - **If asked more:**
       - Would explain the difference between batch size tuning — too small causes many round-trips, too large causes lock escalation
       - Based on what I observed during load testing
55. What is pagination in SQL?
    - **Answer:**
       - Pagination retrieves a subset of rows using `OFFSET` and `FETCH NEXT` in MSSQL
       - In the CDMS partner list API, paginated results with `ORDER BY PartnerID OFFSET 0 ROWS FETCH NEXT 50 ROWS ONLY`
       - Avoided loading thousands of partners at once
    - **If asked more:**
       - Would explain how offset pagination degrades with large offsets because the engine still reads all skipped rows
       - Would explain when I'd switch to keyset pagination
56. Difference between offset pagination and keyset pagination.
    - **Answer:**
       - Offset pagination skips N rows using `OFFSET`
       - Keyset pagination uses `WHERE id > @lastSeenId`
       - Offset pagination becomes slow on large datasets because it reads all skipped rows each time
       - In the inventory audit log, switched to keyset pagination for faster queries
    - **If asked more:**
       - Would show how keyset pagination is ideal for infinite scroll on sorted unique columns but can't skip to arbitrary pages
57. What is query parameterization?
    - **Answer:**
       - Parameterization uses placeholders instead of hardcoded values in SQL, which allows plan reuse and prevents SQL injection
       - In the Spring Boot JPA repositories, all queries were automatically parameterized via prepared statements
    - **If asked more:**
       - Would explain how ad-hoc queries without parameterization caused plan cache bloat in CDMS
       - Would explain how I forced parameterization using `ALTER DATABASE SET PARAMETERIZATION FORCED`
58. What is SQL injection?
    - **Answer:**
       - SQL injection is an attack where malicious SQL is inserted through user input
       - In the inventory API, prevented this by never concatenating user input into SQL strings
       - All queries used JPA repositories or parameterized stored procedures
    - **If asked more:**
       - Would demonstrate how string concatenation like `"WHERE name = '" + userInput + "'"` is vulnerable
       - Would demonstrate how parameterized queries like `WHERE name = @name` prevent it
59. How do you prevent SQL injection?
    - **Answer:**
       - Always use parameterized queries, stored procedures, or ORM frameworks that handle escaping
       - In all projects, strictly use Spring Data JPA or `@Query` with named parameters
       - Validate inputs at the controller layer before they reach the database
    - **If asked more:**
       - Would explain that stored procedures with `EXEC` inside can still be vulnerable
       - Never build dynamic SQL inside procedures — instead use CASE statements or multiple queries
60. What is ETL?
    - **Answer:**
       - ETL (Extract, Transform, Load) is the process of moving data from source systems to a target database
       - In CDMS, worked on an SSIS-based ETL pipeline that extracted partner data from various formats
       - Transformed it (cleaning, validating, aggregating) and loaded it into MSSQL reporting tables
    - **If asked more:**
       - Would walk through a specific ETL flow: extracting CSV files from partner uploads
       - Would explain transforming serial numbers to a standard format and loading them into the inventory tables with error logging
61. What is SSIS?
    - **Answer:**
       - SSIS (SQL Server Integration Services) is Microsoft's ETL tool for data migration and integration
       - In CDMS, used SSIS packages to automate the nightly data load from partner files into the staging and reporting tables
    - **If asked more:**
       - Would explain how I used SSIS data flow tasks with lookup transformations to validate partner codes against the reference table
       - Redirected invalid rows to an error output
62. What is data validation in ETL?
    - **Answer:**
       - Data validation in ETL ensures that transformed data meets business rules before loading
       - In the inventory ETL, validated serial number formats, partner codes against the master list, and date ranges before inserting into the target table
    - **If asked more:**
       - Would explain the validation layers: schema validation, business rule validation, and referential integrity checks
       - Would explain how I logged each failure type separately for partner reporting
63. How do you handle duplicate records?
    - **Answer:**
       - Use `ROW_NUMBER()` with `PARTITION BY` to identify duplicates, then decide whether to deduplicate (keep first/latest) or reject
       - In the inventory pipeline, partitioned by serial number and kept the record with the latest timestamp
       - Logged the rejected duplicates for partner notification
    - **If asked more:**
       - Would explain the business trade-off: sometimes duplicates should be rejected (unique serials), sometimes merged (duplicate partner submissions)
       - Depends on the data domain
64. How do you handle missing data?
    - **Answer:**
       - For missing data, first check if it's truly required or can be defaulted
       - In the inventory ETL, if a serial record was missing the partner ID, logged it to an error table and skipped it
       - Couldn't determine ownership without the partner
    - **If asked more:**
       - Would explain the difference between NULL handling strategies: COALESCE for defaults, ISNULL checks
       - Missing required fields should fail fast rather than propagate bad data
65. How do you reconcile source and target data?
    - **Answer:**
       - Reconciliation compares row counts, checksums, or key values between source and target after ETL
       - In CDMS, built a reconciliation script using `EXCEPT` and `INTERSECT` operators to find mismatches between the source partner files and the loaded reporting tables
    - **If asked more:**
       - Would explain how I automated reconciliation as the final step of the ETL pipeline
       - Sent an email with the mismatch count so the team could investigate before downstream reports were generated
