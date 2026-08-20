# Resume and Project Questions

## Questions

1. Explain your current role as a Software Engineer.
2. What are your main responsibilities in your current project?
3. Which project from your resume are you most confident about?
4. Explain a Channel Data Management System (CDMS).
5. What problem does a CDMS solve?
6. What are the main modules in a typical CDMS?
7. What are typical responsibilities when working on a CDMS?
8. How would you reduce stored procedure execution time from 5 hours to under 12 minutes?
9. What kind of execution plan issues would you identify?
10. What kind of indexes would you create or modify?
11. How would you validate that optimized stored procedures still return correct results?
12. How would you test performance under concurrent load?
13. What is the most difficult part of optimizing a CDMS?
14. What is partner tagging in a CDMS?
15. How does input validation improve reporting accuracy?
16. What are the 4 ETL pipeline stages in a typical CDMS?
17. How can reporting discrepancies be reduced by 90%?
18. What does a 60% throughput improvement mean in an IoT analytics project?
19. Explain a Real-Time Partner Inventory Management System.
20. What problem does an automated inventory validation engine solve?
21. How is inventory calculated from verified baseline data?
22. What are the common inventory data issues?
23. How are duplicate inventory records handled?
24. How are invalid serial records handled?
25. How would you process 10,000+ serial records?
26. How can inventory processing time be reduced to under 15 seconds?
27. How would you measure 94% inventory accuracy?
28. How can reconciliation effort be reduced by 90%?
29. What is the business impact of an automated inventory system?
30. Explain a Cold-Chain Monitoring System.
31. What is a typical backend role in a Cold-Chain Monitoring System?
32. Why is IoT needed in a cold-chain monitoring project?
33. How does data flow from gateways to dashboards?
34. Why use AWS IoT Core?
35. Why use AWS Lambda?
36. Why use Kafka in an IoT monitoring project?
37. Why run Kafka on EC2?
38. How would you design Kafka topics?
39. How do you decide partition count?
40. How are sensor data bursts handled?
41. How are delayed IoT messages handled?
42. How are duplicate IoT messages handled?
43. How are invalid sensor readings handled?
44. Why use InfluxDB?
45. Why use Grafana?
46. What kind of Grafana dashboards would you build?
47. What are excursion alerts?
48. How can temperature excursions be reduced by 85%?
49. How does the system reduce energy costs by 18%?
50. How can load time be improved by 80%?
51. How can responsiveness be improved by 30%?
52. What is the role of Redis in an IoT monitoring project?
53. How are APIs secured with Spring Security?
54. How does JWT authentication work in a project?
55. Where does OAuth2 fit in a project?
56. How is Swagger/OpenAPI used?
57. How does Jenkins reduce deployment time by 70%?
58. How does Docker help in deployment?
59. What is the role of AWS S3?
60. What is the role of PostgreSQL?
61. What is the role of MQTT?
62. What is the role of Databricks?
63. What is the role of DLT?
64. What is the most challenging type of project to work on?
65. Which type of project has the highest business impact?
66. Which type of project has the most technical complexity?
67. What would you improve if you redesigned one of your projects?
68. What production issue might you face and how would you solve it?
69. How do you explain your project to a non-technical person?
70. How do you explain your project architecture to a senior engineer?

---

## Answers
1. Explain your current role as a Software Engineer.
   - **Answer:**
      - A Software Engineer typically owns backend development end-to-end -- from designing REST APIs in Spring Boot to optimizing MSSQL queries that block business reporting
      - They also integrate real-time data pipelines using Kafka and AWS, and collaborate closely with QA and frontend teams to ensure deployments are smooth and APIs are well-documented with Swagger
      - The focus is always on writing code that is maintainable, secure, and performant under load
      - Automated validation engine: validation rules designed in MSSQL using set-based operations over cursors, which directly drives fast processing times
2. What are your main responsibilities in your current project?
   - **Answer:**
      - Developing and maintaining Spring Boot microservices that serve as the backbone for partner management and IoT monitoring platforms
      - A significant amount of time is spent analyzing MSSQL execution plans and rewriting stored procedures to eliminate bottlenecks -- for instance, a procedure can be cut from 5 hours to 12 minutes by fixing a nested loop join that should have been a hash match
      - API security with Spring Security and JWT, validation logic for incoming data streams, and CI/CD pipelines in Jenkins for automated and consistent deployments are also typical responsibilities
      - Query optimization approach: start with the slowest query, capture actual execution plan, identify the highest-cost operator, then decide between an index change, a query rewrite, or a schema adjustment
3. Which project from your resume are you most confident about?
   - **Answer:**
      - A CDMS project is particularly confidence-building because it pushes deep into MSSQL internals -- it is not just writing queries, it is reading actual execution plans, understanding row estimates versus actuals, and making surgical changes that have massive impact
      - Taking a batch process from 5 hours to under 12 minutes teaches that database optimization is less about guessing and more about systematic diagnosis: find the scan, fix the join, add the missing index, verify correctness, then move to the next bottleneck
      - Key finding: a key lookup causing 4.8 million logical reads on a 50k-row table, eliminated entirely by creating a covering index
4. Explain a Channel Data Management System (CDMS).
   - **Answer:**
      - A Channel Data Management System (CDMS) is a data consolidation platform for channel partner ecosystems
      - It aggregates sales, inventory, and incentive data from hundreds of partners into a single MSSQL-based warehouse, then runs complex ETL pipelines to generate reconciled reports
      - A primary contribution can be taking ownership of the most expensive stored procedures -- the ones that run for hours and block the nightly reporting window -- and systematically optimizing them by analyzing execution plans, restructuring joins, and adding targeted indexes until the entire batch completes in under 12 minutes
      - Four ETL stages: staging, validation, transformation, and reporting; worst bottleneck is the transformation stage with a cursor-based loop processing partners one-by-one instead of as a set
5. What problem does a CDMS solve?
   - **Answer:**
      - Before automation, channel teams often rely on manual Excel-based reconciliation with hundreds of external partners, which takes days and is error-prone
      - A CDMS automates the entire data pipeline -- ingestion from partner systems, validation against the company records, incentive calculation, and report generation -- so that the business has accurate, audit-ready numbers every morning instead of waiting a week
      - Performance optimization of stored procedures directly ensures that the automated pipeline can complete within the overnight batch window instead of spilling into business hours
      - If the reporting team starts at 8 AM, a 5-hour procedure means stale data in executive dashboards and delayed partner incentive decisions
6. What are the main modules in a typical CDMS?
   - **Answer:**
      - A typical CDMS has four core modules: the Data Ingestion module that pulls raw files from partner SFTP sites, the Validation Engine that checks data quality against the company master records, the Calculation Engine that runs stored procedures for incentives and discounts, and the Reporting module that generates PDF and Excel outputs for the channel team
      - The Calculation Engine -- specifically the stored procedures that aggregate partner-level sales and compute tier-based rebates -- is often the worst-performing part of the system
      - Data flow: staging tables -> validation stored procedures -> calculation stored procedures -> reporting views
7. What are typical responsibilities when working on a CDMS?
   - **Answer:**
      - The role typically involves optimizing MSSQL stored procedures that drive the nightly incentive calculation batch -- profiling each one, capturing actual execution plans, identifying the heaviest operators, and rewriting queries or adding indexes as needed
      - Correctness validation is also critical: after every change, the old and new procedures are run side-by-side, row counts and totals are compared at each stage, and deployment only happens when bit-for-bit identical results are proven
      - Beyond optimization, contributing to API development for the reporting dashboard and debugging production issues when the batch fails mid-cycle are also common tasks
      - Validation approach: checksums and row-count comparison scripts on production-sized test datasets
8. How would you reduce stored procedure execution time from 5 hours to under 12 minutes?
   - **Answer:**
      - Start by running the procedure with `SET STATISTICS TIME ON` and capturing the actual execution plan from SSMS -- that immediately reveals a clustered index scan on a 50-million-row fact table caused by a join predicate that is not SARGable
      - Rewrite the join condition to be index-friendly, add missing non-clustered indexes that the execution plan recommends, and replace cursor-based loops with single set-based `UPDATE` statements
      - Each change should be tested independently: measure the improvement, verify correctness, and only then move to the next bottleneck
      - Cursor processed one partner at a time across 200 partners; single `UPDATE` with `JOIN` eliminated 199 round trips, reducing the step from 45 minutes to 90 seconds
9. What kind of execution plan issues would you identify?
   - **Answer:**
      - The most common issue is table scans on large fact tables caused by implicit type conversions -- a `VARCHAR` column being compared to an `NVARCHAR` parameter forces SQL Server to scan the entire clustered index instead of seeking
      - Other common issues include nested loop joins when the outer table has millions of rows (which should have been hash matches); key lookups on non-covering indexes that add millions of unnecessary page reads; and cursor-based loops that process records one-by-one instead of as a set
      - Diagnose implicit conversions via yellow warning triangles in the execution plan, then check column vs. parameter data types in the `WHERE` clause
10. What kind of indexes would you create or modify?
    - **Answer:**
       - Non-clustered covering indexes that include all columns referenced in the `SELECT`, `WHERE`, and `JOIN` clauses are most effective, allowing SQL Server to satisfy the query entirely from the index without touching the table
       - For critical procedures, modifying an existing composite index by reordering the key columns can help -- if the original had `Status` first and `Date` second but the query filters on `Date` ranges and only then on `Status`, swapping the order can turn a scan into a seek
       - Filtered indexes for queries that always target a specific status value keep the index small and fast
       - Used the `INCLUDE` clause to add payload columns without bloating the index key; verified improvement by confirming key lookups disappeared from the execution plan
11. How would you validate that optimized stored procedures still return correct results?
    - **Answer:**
       - A validation harness in T-SQL can be built to run old and new versions of the procedure side-by-side against the same production-mirror database
       - For every output table, row counts, checksums, and min/max/sum of key numeric columns are compared, and any discrepancy is flagged down to the individual row level
       - Edge cases should also be tested -- empty input sets, duplicate records, NULL values in join columns -- to ensure optimizations do not silently break business logic
       - Used `CHECKSUM_AGG(BINARY_CHECKSUM(*))` per table to get a fingerprint of the entire result set and compared fingerprints between old and new runs
12. How would you test performance under concurrent load?
    - **Answer:**
       - A PowerShell script can launch parallel `sqlcmd` sessions to mimic the real-world scenario where multiple reports kick off at the same time, while monitoring `sys.dm_exec_requests` for blocking and wait statistics
       - This can expose blocking issues where the optimized procedure holds a table-level lock -- fix by adding index-level locking hints and breaking long-running transactions into smaller batches
       - Used `WITH (ROWLOCK)` on the target table to allow concurrent reads; verified no performance degradation under single-user load
13. What is the most difficult part of optimizing a CDMS?
    - **Answer:**
       - The hardest part is often untangling a 400-line stored procedure that has been modified by multiple developers over two years -- with nested transactions, GOTO statements, and temp tables that are created and dropped in confusing order
       - Rewriting from scratch is not always an option because every business rule needs to be preserved
       - Refactoring step by step is the approach: extract cursor logic into a set-based CTE, consolidate temp tables into a single table variable, and test each block independently before merging
       - The overall time can drop from 5 hours to 12 minutes, but the refactoring alone may take two weeks of careful work
       - A GOTO-based error handler can mask a real bug -- catching an error, logging it, then jumping back into the middle of the loop, causing duplicate processing
14. What is partner tagging in a CDMS?
    - **Answer:**
       - Partner tagging is a classification system in a CDMS where each external partner is assigned metadata tags -- like region, tier, product specialization, or incentive program -- that drive how their data is processed and reported
       - For example, a "Platinum-tier" partner in the "North" region might get different rebate calculations and a different report format than a "Silver-tier" partner in "South"
       - Stored procedure optimization directly impacts partner tagging because the procedures use these tags as join and filter criteria, and the original queries often have poor index usage on the tag lookup tables
       - Tags stored in a normalized `PartnerTag` junction table with `PartnerID` and `TagID`; missing composite index on `(TagID, PartnerID)` caused a scan for every tag-based filter
15. How does input validation improve reporting accuracy?
    - **Answer:**
       - Before validation is implemented, raw partner data is loaded directly into the calculation engine -- if a partner sends a duplicate serial number or a date in the wrong format, the procedure silently produces incorrect incentive totals
       - A validation layer in MSSQL can be added to run before the main calculation: it checks data types, looks for duplicates against the baseline inventory, flags records that reference non-existent partners, and quarantines failures into an exception table
       - This stops bad data from polluting reports and gives the operations team a clear list of what to fix, driving significant reductions in reconciliation effort
       - Duplicate detection via `ROW_NUMBER() OVER (PARTITION BY SerialNumber ORDER BY CreatedDate)`, keeping only `RowNumber = 1`
16. What are the 4 ETL pipeline stages in a typical CDMS?
    - **Answer:**
       - Stage 1 is **Extract** -- raw files from partner SFTP servers are pulled and staged into flat staging tables with minimal transformation
       - Stage 2 is **Validate** -- the validation engine checks data completeness, format correctness, and business rule compliance, flagging failures into exception queues
       - Stage 3 is **Transform** -- the clean data is aggregated, joined with master data, and run through calculation stored procedures to compute incentives and rebates
       - Stage 4 is **Load** -- the final results are written to reporting tables and pushed to the partner portal and executive dashboards
       - Stage 3 (Transform) is often the bottleneck; incentive calculation can take hours due to cursor-based partner loops and missing indexes
17. How can reporting discrepancies be reduced by 90%?
    - **Answer:**
       - This is achieved by shifting from manual partner-sent spreadsheets to an automated, validated pipeline where discrepancies are caught at ingestion time rather than at reconciliation time
       - Before automation, operations teams export partner data, manually compare it against the company records in Excel, and chase partners for corrections -- a process that takes days and is error-prone
       - After automation, if a partner sends a serial number that is already claimed, the system flags it instantly, holds it in an exception table, and the report only includes verified records
       - Tracked post-report manual correction tickets from the finance team; quarterly average dropped from 40+ tickets to under 4
18. What does a 60% throughput improvement mean in an IoT analytics project?
    - **Answer:**
       - In an IoT analytics project, a 60% throughput improvement means the system can handle 60% more sensor messages per second from the IoT gateways without any increase in infrastructure cost
       - Before optimization, Kafka consumers might process about 500 messages per second before lag starts building up; after tuning consumer batch sizes, adjusting the Spring Boot thread pool, and adding Redis caching for duplicate detection, the same consumers can handle 800 messages per second with stable lag
       - Duplicate-detection query on MSSQL added 15ms latency per message; moving dedup to Redis TTL-based set cut it to under 1ms
19. Explain a Real-Time Partner Inventory Management System.
    - **Answer:**
       - This type of system is built to solve a trust problem between a company and its external partners -- each partner reports their inventory independently, and the numbers never match the company baseline
       - A centralized MSSQL validation engine can take the company verified baseline data as the single source of truth and run every incoming partner inventory record against it
       - The engine validates serial numbers, flags duplicates, rejects records that do not match the baseline, and calculates the true inventory in under 15 seconds per partner -- all in set-based T-SQL so it scales linearly
       - Validation joins incoming serial list against baseline using `LEFT JOIN` and `NOT EXISTS`; identifies valid, duplicate, and invalid records in a single pass
20. What problem does an automated inventory validation engine solve?
    - **Answer:**
       - The problem is that channel teams often have no automated way to reconcile inventory across hundreds of partners -- each partner sends their stock data in different formats, often with duplicates or serials that do not match the company distribution records
       - The manual reconciliation process took a team of five people nearly a week every month, and even then, discrepancies of 10-15% were common
       - The validation engine automates the entire process: ingesting partner data, validating every serial number against the trusted baseline, and producing a reconciled inventory count in seconds, cutting reconciliation effort by 90%
       - One large partner was double-reporting the same 500 serial numbers across two different product lines, inflating their inventory significantly until the engine caught it
21. How is inventory calculated from verified baseline data?
    - **Answer:**
       - The baseline data is a table of serial numbers that the company has shipped to each partner, with status columns like `Shipped`, `In-Transit`, `Delivered`, and `Sold`
       - When a partner submits their current inventory, their serial list is joined against the baseline using MSSQL set operations: records in both lists are marked as verified, records in the baseline but missing from the partner list are flagged as potentially sold or lost, and records in the partner list but not in the baseline are flagged as invalid
       - The final inventory count is simply the count of verified `Delivered` serials minus any serials the partner had reported as sold in previous cycles
       - Used `LAG()` to compare current vs. previous submission; automatically detected serials that moved from "in stock" to "missing" without a corresponding sale report
22. What are the common inventory data issues?
    - **Answer:**
       - The most common issue is **duplicate serials** -- the same serial number appearing in two different partner submissions, either because partners accidentally double-submitted or because serials were being passed between partners in the gray market
       - The second issue is **invalid serials** -- partners reporting serials that the company had never shipped to them, often due to data entry errors or mixing up stock from different distributors
       - Third is **missing serials** -- partners reporting fewer units than the company baseline showed, which could mean unreported sales or theft
       - If Serial X was shipped to Partner A but reported by Partner B, the system flagged it as a cross-partner exception and escalated to the channel sales manager
23. How are duplicate inventory records handled?
    - **Answer:**
       - A multi-layered deduplication strategy is typically employed
       - First, within a single partner submission, `ROW_NUMBER() PARTITION BY SerialNumber ORDER BY ReportedDate DESC` keeps only the most recent occurrence of each serial
       - Second, across submissions, a history table of all previously verified serials is maintained -- if a serial appears again in a new submission from a different partner, it is flagged as a cross-partner duplicate and quarantined for manual review
       - The dedup logic is entirely in MSSQL and runs as part of the validation stored procedure
       - Cross-partner dedup used `EXISTS` check against verified inventory history table with a filtered index on `SerialNumber` for active records only
24. How are invalid serial records handled?
    - **Answer:**
       - Invalid serials are records submitted by a partner that do not exist in the company baseline distribution table at all
       - The validation procedure uses a `LEFT JOIN` where the baseline table is on the right side, and if the join produces a NULL baseline serial, the record is flagged as invalid
       - These invalid records are written to an exception table with the partner ID, serial number, and a reason code, and the main inventory calculation excludes them entirely
       - The partner portal displays these exceptions in real time so the partner can correct their submission and resubmit
       - Exception flow: invalid record -> exception table -> email to partner -> partner resubmits corrected data -> validation re-runs -> record moves from exception to verified
25. How would you process 10,000+ serial records?
    - **Answer:**
       - Processing is done using set-based T-SQL operations -- loading all 10,000+ serials into a table-valued parameter in a single batch, then running validation, dedup, and baseline joins against that set in one pass per partner
       - The key is avoiding RBAR (row-by-agonizing-row) processing: instead of looping through each serial number, `MERGE` statements and set-based `UPDATE` with `JOIN` validate the entire batch in milliseconds
       - Even with 200 partners and 10,000+ serials each, the total processing time stays under 15 seconds because every operation is index-optimized and set-based
       - Partner submission passed as a structured TVP with columns `SerialNumber`, `ProductCode`, `ReportedDate`; MSSQL processed it as a single relational set
26. How can inventory processing time be reduced to under 15 seconds?
    - **Answer:**
       - The original approach processed each serial number individually through a cursor, which cost about 50ms per serial -- for a partner with 10,000 serials, that alone was 500 seconds
       - Replace the cursor with a single `INSERT INTO #validated SELECT s.* FROM @submitted s JOIN Baseline b ON s.SerialNumber = b.SerialNumber` -- a set-based operation that validates all 10,000 serials in under 100ms
       - Then apply the same approach to deduplication and exception handling: replace every loop with a set-based join or window function, and add covering indexes on the `SerialNumber` column in all temp and base tables
       - Cursor version: 10,000 individual index seeks (one per serial); set-based version: single hash match join with 10,000 input rows
27. How would you measure 94% inventory accuracy?
    - **Answer:**
       - Accuracy is measured by comparing the system validated inventory count against physical stock audits conducted by the channel team
       - After processing all partner submissions through the validation engine, spot-check physical audits are run for a random sample of partners each quarter
       - A 94% figure means that for audited partners, the system inventory count matches the physical count within a 1% tolerance 94% of the time -- up from approximately 72% before the validation engine was implemented
       - Stratified random sampling by partner tier (Platinum, Gold, Silver) with 95% confidence interval; weighted average accuracy calculated across all tiers
28. How can reconciliation effort be reduced by 90%?
    - **Answer:**
       - Before the engine, reconciliation means five people spending 3-4 days manually comparing Excel sheets from hundreds of partners against the company SAP exports
       - After the engine, the entire process is automated: partners submit data through a portal, the system validates and reconciles in 15 seconds, and exceptions are displayed in a dashboard
       - The operations team only has to handle the 6% of records that are flagged as exceptions, rather than manually checking 100% of submissions
       - That is where the 90% reduction in effort comes from -- going from checking 200,000 serials manually to reviewing 12,000 exceptions
       - Before: 5 people x 4 days x 8 hours = 160 person-hours per cycle; after: 1 person x 4 hours = 4 person-hours per cycle; (160-4)/160 = 97.5% reduction
29. What is the business impact of an automated inventory system?
    - **Answer:**
       - The biggest business impact is that the channel finance team can finally close the monthly inventory reconciliation in one day instead of one week -- meaning partner incentives can be calculated and paid on time, which improves partner satisfaction and reduces disputes
       - A 94% accuracy rate also means the company has reliable data for demand forecasting and production planning -- avoiding overproduction caused by inflated partner-reported inventory numbers
       - Accurate inventory reduced overproduction, saving significant working capital annually; operations team freed to focus on partner relationships instead of data entry
30. Explain a Cold-Chain Monitoring System.
    - **Answer:**
       - A real-time IoT platform can monitor temperature and humidity in cold-storage units across a supply chain
       - The data flow starts with IoT gateways in each cold room sending MQTT messages to AWS IoT Core, which triggers a Lambda function that publishes the messages to a Kafka topic running on EC2
       - Spring Boot microservices consume from Kafka, apply validation and deduplication, store the time-series data in InfluxDB, and serve it through REST APIs to a React dashboard and a Grafana instance
       - The entire backend pipeline -- Kafka consumer configuration, Spring Boot APIs, InfluxDB schema design, and Redis caching layer -- can be implemented to ultimately reduce temperature excursions by 85% and energy costs by 18%
       - Example flow: sensor reads 8C (above 6C threshold) -> IoT Core -> Lambda -> Kafka -> Spring Boot -> InfluxDB -> Grafana alert triggers -> operations team notified within 30 seconds
31. What is a typical backend role in a Cold-Chain Monitoring System?
    - **Answer:**
       - The backend data pipeline involves configuring the Kafka consumer group with 6 partitions and 3 consumers to handle sensor bursts, writing the Spring Boot service that deserializes, validates, and deduplicates incoming messages before writing to InfluxDB, and setting up the Redis cache to store the latest reading per sensor so the dashboard can show real-time data without hitting the database
       - Grafana alerting rules that detect temperature excursions and trigger email and Slack notifications are also integrated, and the Spring Boot application is containerized using Docker with a Jenkins CI/CD pipeline for automated deployment
       - Key challenge: tuning Kafka `max.poll.records` and `fetch.max.wait.ms` to balance latency vs. throughput when sensor data burst from 100 msg/s to 800 msg/s during peak seasons
32. Why is IoT needed in a cold-chain monitoring project?
    - **Answer:**
       - IoT is essential because cold-chain inventory is perishable and time-sensitive -- a refrigerator unit failing at 2 AM could spoil valuable products before anyone notices without automated monitoring
       - Manual temperature logging (someone checking a thermometer twice a day) is unreliable and cannot detect transient excursions that last only 15 minutes
       - IoT provides continuous, real-time telemetry from every cold-storage unit, automated alerts the moment a threshold is breached, and historical data for root-cause analysis and energy optimization -- all without human intervention
       - 3-4 sensors per cold room (top, middle, bottom, near door) to detect hot spots and door-open events; readings every 30 seconds
33. How does data flow from gateways to dashboards?
    - **Answer:**
       - The gateway devices in each cold room publish JSON payloads over MQTT to AWS IoT Core, which has a rule that forwards every message to a Lambda function
       - The Lambda does a lightweight parse and then produces the message to a Kafka topic on EC2
       - The Spring Boot Kafka consumer picks up the message, validates it (checks for null fields, out-of-range values, duplicate message IDs), enriches it with metadata (room name, product type), and writes it to InfluxDB
       - The Grafana dashboard queries InfluxDB directly for live charts, while the React dashboard calls Spring Boot REST APIs that aggregate data from both InfluxDB and Redis for the most recent readings
       - Kafka chosen over Lambda-to-InfluxDB because Lambda has a 15-minute timeout and no built-in retry queue; Kafka provides persistent buffering, replay capability, and ordered processing
34. Why use AWS IoT Core?
    - **Answer:**
       - AWS IoT Core is a fully managed MQTT broker that handles device authentication, TLS termination, and topic-based routing out of the box -- eliminating the need to run and scale a custom MQTT broker on EC2
       - It also integrates natively with Lambda via IoT rules, allowing message processing without writing polling code
       - The device shadow feature is useful for tracking the last known state of each gateway, and IoT Core policies allow restricting each device to only publish to its own topic
       - Each gateway had an X.509 certificate installed at manufacturing time; IoT Core validated the certificate on connect and applied the policy scoping the device to its `/<location>/<room>/<sensor-id>` topic
35. Why use AWS Lambda?
    - **Answer:**
       - Lambda serves as the lightweight bridge between AWS IoT Core and Kafka -- it receives the MQTT message from IoT Core via a rule, deserializes the JSON, adds a timestamp if missing, and publishes it to Kafka
       - Lambda is chosen over a persistent EC2 consumer because IoT message volume is spiky (near-zero at night, 800 msg/s during peak) and Lambda scales to zero when not needed
       - The function is simple -- about 50 lines of Python -- and its only job is to transform the MQTT payload into a Kafka-compatible format and handle any parsing errors gracefully
       - On Lambda parse failure or Kafka producer exception, the raw payload is written to an S3 bucket with a timestamp and error code for later replay
36. Why use Kafka in an IoT monitoring project?
    - **Answer:**
       - A durable, scalable buffer is needed between the IoT ingestion layer and the downstream analytics pipeline because the data rate is unpredictable -- sensors publish every 30 seconds per room, but with hundreds of rooms the aggregate rate can spike to 800 messages per second
       - Kafka provides the ability to absorb spikes without back-pressuring the IoT Core -> Lambda path, and decouples the producers (Lambda) from the consumers (Spring Boot) so that if the analytics pipeline goes down for maintenance, no sensor data is lost
       - A single topic with 6 partitions, messages partitioned by `room-id` for ordering; 3 Spring Boot consumers in the same group for parallel processing
37. Why run Kafka on EC2?
    - **Answer:**
       - Running Kafka on EC2 instead of using MSK can be a cost-sensitive decision -- at the time, MSK minimum cluster cost was significantly higher than running a single `m5.large` instance with Kafka and Zookeeper containerized in Docker
       - For throughput requirements of 800 msg/s peak with 500 GB retention for 7 days, a single well-tuned EC2 instance is sufficient
       - If the project scales to 10x the volume, migrating to MSK would be appropriate, but for initial deployment, EC2 is the pragmatic choice
       - EC2 config: gp3 EBS volumes with 3000 IOPS for Kafka logs, JVM heap set to 8 GB, `log.retention.hours=168`, `unclean.leader.election.enable=false` to prevent data loss
38. How would you design Kafka topics?
    - **Answer:**
       - A single topic with 6 partitions is sufficient when all sensor messages share the same schema and processing logic
       - The key for partitioning is the `room-id`, which guarantees that all messages from the same cold room go to the same partition and are thus consumed in order
       - A separate alerts topic with 3 partitions for alert events (excursions, device offline) allows alerting consumers to scale independently of the sensor data consumers
       - Used `cleanup.policy=delete` for the main topic and `log.retention.ms=604800000` for 7-day retention to support historical replays
39. How do you decide partition count?
    - **Answer:**
       - Choosing 6 partitions is based on three factors: the target throughput of 800 msg/s, the consumer processing capacity of about 300 msg/s per Spring Boot instance, and the number of unique rooms (about 200)
       - With 6 partitions and 3 consumers (2 partitions per consumer), each consumer handles roughly 270 msg/s, which is well under capacity and leaves headroom for spikes
       - The partition count should be a multiple of the expected consumer count for balanced distribution
       - With 100 partitions, each consumer handles 33 partitions; a consumer failure triggers full rebalance of 100 partitions taking 30+ seconds
40. How are sensor data bursts handled?
    - **Answer:**
       - Sensor data bursts happen primarily during morning and evening hours when cold-room doors are opened frequently for stock movement, causing the publish rate to spike from ~100 msg/s to ~800 msg/s
       - The architecture handles this naturally: AWS IoT Core accepts the burst and queues messages to Lambda, Lambda scales up concurrency, and Kafka accepts the increased write rate because it has enough partition capacity
       - On the consumer side, configuring `max.poll.records=500` and `fetch.max.wait.ms=500` allows consumers to fetch larger batches during bursts while maintaining low latency during normal periods
       - Health check endpoint in Spring Boot dynamically reduced the consumer `max.poll.records` when Kafka consumer lag grew beyond 10,000 messages, preventing memory exhaustion
41. How are delayed IoT messages handled?
    - **Answer:**
       - Delayed messages occur when a gateway loses network connectivity for a few minutes and then reconnects, sending all buffered messages at once with original timestamps
       - This is handled at the InfluxDB write layer: each data point is tagged with `ingestion_time` (when Spring Boot received it) and `sensor_time` (when the gateway recorded it)
       - InfluxDB stores data by `sensor_time`, so delayed messages are written to their correct time bucket even if they arrive late
       - Each gateway synced its clock via NTP every hour; if clock skew was greater than 5 seconds, messages carried a `clock_error` flag stored as an InfluxDB tag for data-quality filtering
42. How are duplicate IoT messages handled?
    - **Answer:**
       - Duplicate messages happen when a gateway retransmits a message because it did not receive an MQTT QoS 2 acknowledgment, or when Lambda at-least-once delivery to Kafka produces duplicates
       - Deduplication in the Spring Boot consumer can be implemented using a combination of a unique `message_id` (UUID generated by the gateway) and a Redis SET with TTL
       - When a message arrives, the consumer checks if its `message_id` already exists in Redis: if yes, the message is skipped; if no, it is added to Redis with a 24-hour TTL and then written to InfluxDB
       - MSSQL primary-key check would add 5-10ms latency per message; Redis in-memory SET operations are orders of magnitude faster at 800 msg/s
43. How are invalid sensor readings handled?
    - **Answer:**
       - A set of validation rules in the Spring Boot consumer runs before writing to InfluxDB: temperature must be between -20C and 60C, humidity between 0% and 100%, and `sensor_time` must be within 5 minutes of the current time
       - If a message fails validation, it is written to a separate `invalid-readings` Kafka topic, and the Grafana dashboard has a panel showing the count of invalid readings per room over time to help identify failing sensors
       - Example: a sensor once started reporting -40C due to a hardware fault; the validation rule caught it, the invalid reading was logged, and the maintenance team replaced the sensor before it caused a false temperature excursion alert
44. Why use InfluxDB?
    - **Answer:**
       - InfluxDB is purpose-built for time-series data -- it handles millions of data points per second with automatic downsampling, retention policies, and continuous queries that would be impractical in a relational database
       - Each sensor reading (temperature, humidity, timestamp, room-id) is stored as a single point, and InfluxDB columnar storage compresses the data by 10x
       - It also integrates natively with Grafana, so dashboard queries are simple Flux queries without custom API middleware
       - Retention: raw data for 7 days at 30-second resolution, then downsampled to 5-minute averages for 30 days, and hourly averages for 1 year, managed by InfluxDB tasks
45. Why use Grafana?
    - **Answer:**
       - Grafana provides production-ready dashboards with minimal effort -- connecting directly to InfluxDB, real-time panels for temperature trends, humidity charts, and excursion alerts can be built in a few hours
       - The alerting engine is critical: Grafana evaluates alert rules every 30 seconds, and when a temperature reading exceeds the threshold for more than 2 consecutive evaluations, it sends notifications via Slack and email
       - Dashboard layout: top row with real-time temperature gauges per cold room, middle row with 24-hour trend lines with excursion zones highlighted in red, and bottom row with alert history
46. What kind of Grafana dashboards would you build?
    - **Answer:**
       - Three main dashboards are typically built
       - The **Operations Dashboard** shows a grid of all cold rooms, each with a color-coded tile (green=normal, yellow=warning, red=excursion), and clicking a tile opens a detailed 24-hour trend chart
       - The **Alerts Dashboard** lists all active and historical excursions with room name, threshold breached, duration, and resolution time
       - The **Energy Dashboard** displays power consumption trends per cold room with a temperature overlay -- helping identify rooms that are overcooling and wasting energy
       - Example: Room A consuming 30% more power than Room B for the same temperature range led to discovery of a faulty door seal, driving the 18% cost reduction
47. What are excursion alerts?
    - **Answer:**
       - An excursion alert fires when a sensor reading stays outside an acceptable temperature or humidity range for longer than a configured duration
       - For example, if a vaccine cold room had a threshold of 2C to 8C, and the sensor reported 9C for three consecutive readings (90 seconds), the system classified that as a minor excursion
       - If the temperature exceeded 10C or stayed outside range for more than 5 minutes, it became a major excursion and triggered immediate Slack and SMS alerts to the on-call engineer
       - Severity levels: minor (informational, auto-resolved), major (requires acknowledgment within 15 minutes), critical (requires immediate action, escalates to manager if unacknowledged after 5 minutes)
48. How can temperature excursions be reduced by 85%?
    - **Answer:**
       - The 85% reduction comes from shifting from reactive to proactive monitoring
       - Before the system, the operations team does not know about an excursion until someone opens the cold room hours later
       - With real-time monitoring and alerts, the team can respond to a rising temperature trend within 30 seconds -- checking if a door was left open, if the compressor has failed, or if the setpoint has been changed
       - Additionally, historical data helps identify patterns, like solar heat gain through a window at 2 PM, and fix the root cause permanently
       - Metric: (excursions before system - excursions after system) / excursions before system x 100, comparing 6 months before deployment to 6 months after on the same set of cold rooms
49. How does the system reduce energy costs by 18%?
    - **Answer:**
       - The energy savings come from data-driven optimization of cooling setpoints
       - Historical InfluxDB data can reveal that several rooms are being overcooled -- kept at 2C when the product only requires 6C, which wastes significant energy
       - By analyzing temperature trends alongside power consumption, the optimal setpoint for each room can be identified based on its insulation quality, ambient temperature, and product requirements
       - Fixing issues like faulty door seals and insulation gaps directly reduced compressor runtime
       - Average room: 15,000/month before -> 12,300/month after; 2,700/month savings per room x 200 rooms
50. How can load time be improved by 80%?
    - **Answer:**
       - The React dashboard initial load time can be improved by implementing server-side pagination and lazy loading for the cold-room grid -- originally, the API returns all rooms with full sensor history, which can be about 15 MB of JSON and takes 12 seconds to parse and render
       - Changing the API to return only the latest reading per room with a summary status (green/yellow/red) cuts the payload to under 50 KB, loading room details on demand when the user clicks a tile
       - Redis caching for the aggregate dashboard data with a 10-second TTL also helps
       - API response: 3.2s -> 180ms; React render: 8.5s -> 400ms; total perceived load time: ~12s -> ~2s
51. How can responsiveness be improved by 30%?
    - **Answer:**
       - The 30% responsiveness improvement refers to the time between a sensor reading arriving at the backend and the dashboard reflecting that new value
       - The bottleneck is the polling interval: the React dashboard polls the REST API every 15 seconds
       - Switching to WebSocket connections allows the Spring Boot service to push new readings to connected clients as soon as they are written to InfluxDB, reducing update latency to under 500ms
       - Used Spring Boot `WebSocketHandler` with `SimpMessagingTemplate` broadcasting the latest reading to the `/topic/latest-readings` channel whenever a new data point was persisted
52. What is the role of Redis in an IoT monitoring project?
    - **Answer:**
       - Redis serves three purposes
       - First, it serves as the **deduplication cache**: storing `message_id` for each incoming message with a 24-hour TTL, so duplicate messages are detected in under 1ms
       - Second, it serves as the **latest-reading cache**: storing the most recent valid reading for each sensor so the dashboard summary API can return the current state of all rooms instantly
       - Third, it serves as the **session store** for Spring Session -- user sessions are stored in Redis so that any Spring Boot instance can serve any request without losing session state
       - Dedup cache: ~200K keys x ~100 bytes = ~20 MB; latest-reading cache: 600 keys x ~200 bytes = ~120 KB; both fit in a single t3.micro 500 MB allocation
53. How are APIs secured with Spring Security?
    - **Answer:**
       - Spring Security can be configured with a JWT-based authentication filter that intercepts all API requests except the public health-check endpoint
       - The filter extracts the `Authorization: Bearer <token>` header, validates the token signature using an RSA public key, checks the token expiration and issuer claims, and then sets the `SecurityContext` with the user roles and permissions
       - Method-level security using `@PreAuthorize` annotations can also be applied -- admin endpoints restricted to `ROLE_ADMIN`, dashboard data to `ROLE_VIEWER`
       - For the IoT ingestion API, API keys are used instead of JWT because sensors cannot manage token refresh
       - React app received 15-minute access token + 7-day refresh token; Axios interceptor auto-refreshed the access token on expiry
54. How does JWT authentication work in a project?
    - **Answer:**
       - When a user logs in through the React login page, the Spring Boot authentication endpoint validates the username/password against the MSSQL user table and returns a signed JWT access token (valid for 15 minutes) and a refresh token (valid for 7 days)
       - The JWT contains the user ID, roles, and a session ID in its claims, signed with an RSA private key so any service can verify the token using the public key
       - The React app stores the token in `localStorage` and sends it in the `Authorization` header for every API call
       - RSA was chosen over HMAC because with HMAC every service needs to share the same secret key; with RSA only the authentication service holds the private key
55. Where does OAuth2 fit in a project?
    - **Answer:**
       - OAuth2 can be used for Grafana authentication integration -- instead of managing separate Grafana user accounts, Grafana can be configured to use OAuth2 with the Spring Boot application as the authorization server
       - When a user logs into Grafana, they are redirected to the login page, authenticate with their existing credentials, and Grafana receives an access token for API calls
       - The `oauth2` section in `grafana.ini` has `auth_url`, `token_url`, and `api_url` pointing to the Spring Boot endpoints, with role mapping from JWT claims
56. How is Swagger/OpenAPI used?
    - **Answer:**
       - SpringDoc OpenAPI can be used to auto-generate Swagger documentation for all REST APIs
       - The documentation is available at `/swagger-ui.html` and `/v3/api-docs` for each microservice
       - The Swagger UI becomes the primary reference for the frontend team -- they can see every endpoint, its schema, authentication requirements, and test calls directly from the browser
       - The frontend team can use the OpenAPI spec to generate TypeScript API clients using `openapi-generator`, which eliminates manual type definitions
       - Used `@Tag` annotations to group endpoints by domain (Sensor Data, Alerts, Configuration, Users) with separate sections in Swagger UI
57. How does Jenkins reduce deployment time by 70%?
    - **Answer:**
       - Before CI/CD, deployments are manual: build the JAR locally, SCP it to EC2, SSH in, stop the service, replace the JAR, restart, verify -- about 30 minutes per deployment
       - A Jenkins pipeline can automate the entire process: on every push to `main`, Jenkins checks out code, runs tests, builds the Docker image, pushes it to Docker Hub, SSHs into EC2, pulls the new image, stops the old container, starts the new one, and runs a smoke test -- all in under 10 minutes
       - The pipeline also includes automatic rollback if the smoke test fails
       - Jenkinsfile stages: Checkout -> Test -> Build -> Dockerize -> Deploy -> Smoke Test -> Rollback (on failure); post-build Slack notifications
58. How does Docker help in deployment?
    - **Answer:**
       - Docker eliminates the "it works on my machine" problem by packaging the Spring Boot application, its dependencies, and the JVM version into a single container image that runs identically on the developer laptop, the Jenkins build server, and the production EC2 instance
       - It also makes deployments atomic and reversible -- tagging each image with the Git commit hash, and rolling back is as simple as running with the previous image tag
       - Four containers can run on a single EC2 instance: Spring Boot API, Redis, Grafana, and Nginx -- all managed with Docker Compose
       - A custom bridge network allows containers to communicate by service name (e.g., `api:8080`, `redis:6379`) instead of environment-specific IP addresses
59. What is the role of AWS S3?
    - **Answer:**
       - S3 can serve as the long-term archival store for raw sensor data
       - While InfluxDB keeps high-resolution data for 7 days and downsampled data for up to a year, all raw sensor messages can be archived to S3 in Parquet format using a nightly batch job
       - This serves two purposes: compliance (retaining raw data for regulatory periods) and reprocessing (if a bug is discovered in the data pipeline, archived data can be replayed)
       - Each object was keyed by `year/month/day/room-id/hour.parquet` for efficient range queries
       - Midnight scheduled task: queried last 24 hours from MSSQL tracking table -> serialized to Parquet -> uploaded to S3 with server-side encryption
60. What is the role of PostgreSQL?
    - **Answer:**
       - PostgreSQL can be used as the metadata and configuration database -- storing user accounts, role assignments, cold-room metadata, alert configuration rules, and audit logs
       - PostgreSQL is often chosen over MSSQL for non-time-series workloads because the operational overhead is lower and the `JSONB` column type is convenient for storing flexible alert rule configurations
       - The Spring Boot application uses JPA/Hibernate to interact with PostgreSQL, and InfluxDB queries reference room IDs that are joined against PostgreSQL metadata in Grafana
       - `alert_rules` table: `room_id`, `metric` (temperature/humidity), `operator` (gt/lt), `threshold`, `duration_seconds`, `severity`, and a `channels` JSONB column for notification targets
61. What is the role of MQTT?
    - **Answer:**
       - MQTT is the communication protocol between the IoT gateways and AWS IoT Core
       - MQTT is chosen over HTTP because it is lightweight (the binary header is only 2 bytes), supports persistent connections with minimal overhead, and has built-in QoS levels -- QoS 1 (at-least-once delivery) is commonly used for reliable delivery
       - Gateways publish messages to hierarchical topics like `/<location-id>/<room-id>/<sensor-id>`, and IoT Core uses topic filtering to route each location data to the appropriate Lambda function
       - 600 unique topics across 20 locations, 10 rooms per location, 3 sensors per room; IoT Core wildcard subscription `+/+/+` captures all messages
62. What is the role of Databricks?
    - **Answer:**
       - Databricks can be used for offline analytics and reporting on aggregated sensor data
       - While Grafana handles real-time operational dashboards, the business team needs weekly and monthly reports on energy consumption trends, excursion patterns by location, and sensor reliability statistics
       - Downsampled InfluxDB data can be exported to S3 as Parquet files daily, and Databricks notebooks read that data to generate trend analyses and predictive models
       - One use case was a linear regression on energy consumption vs. ambient temperature for each room, identifying rooms where the slope was abnormally high (indicating poor insulation) and generating a prioritized maintenance work order list
63. What is the role of DLT?
    - **Answer:**
       - DLT (Delta Live Tables) in Databricks is used to build and maintain the ETL pipeline that transformed raw sensor archives into clean, analytics-ready tables
       - Instead of writing manual Spark jobs with fragile scheduling, DLT allows defining the pipeline declaratively -- specifying the source (S3 Parquet files), the transformations (deduplication, filtering invalid readings, joining with room metadata), and the target Delta tables, with DLT handling incremental processing, data quality checks, and automatic retries
       - Three-tier medallion: "bronze" table ingested raw Parquet from S3, "silver" table applied deduplication and validation, "gold" table joined with room metadata for the business reporting layer
64. What is the most challenging type of project to work on?
    - **Answer:**
       - Stored procedure optimization in a CDMS can be the most challenging type of work because it is not just a technical problem -- understanding the business meaning of every column, every join, and every temp table in a 400-line procedure that nobody has fully documented is essential
       - The pressure is high when the nightly batch has been failing for weeks and the business team needs their reports
       - Balancing speed with safety is critical -- introducing a calculation error that affects partner incentives is not acceptable
       - The approach of making one change at a time, testing it against the full dataset, and only then moving to the next bottleneck is what makes it work
       - After three days of analysis: `WHERE CAST(date_column AS DATE) = '2024-01-01'` forced a full table scan because the `CAST` made the predicate non-SARGable; rewriting it to a range condition alone cut 45 minutes from the runtime
65. Which type of project has the highest business impact?
    - **Answer:**
       - Automated validation engines can have high direct business impact because they save the channel finance team an entire week of manual work every month and provide trustworthy inventory numbers for the first time
       - Before the system, the finance team cannot close the monthly books on time because inventory discrepancies take days to resolve
       - After the system, reconciliation is a 15-second automated process with a clear audit trail, and the finance team can close the month on the first working day
       - Before the system: dispute over 2,000 serial numbers took 3 weeks of email exchanges and physical audits; after: same scenario resolved in 15 seconds
66. Which type of project has the most technical complexity?
    - **Answer:**
       - IoT monitoring systems tend to be the most technically complex because they involve the widest range of technologies and failure modes -- hardware failures (sensors dying, gateways disconnecting), network issues (MQTT disconnections, Kafka broker restarts), data quality problems (duplicates, delayed messages, corrupt payloads), and real-time performance requirements all at once
       - It is not enough to make each component work individually -- they all must work together reliably, and a failure in any layer can cascade
       - One complex failure: Kafka broker disk filled up because retention was not configured correctly after a schema change increased message size by 3x; consumers could not commit offsets, which caused rebalancing and duplicate downstream writes
67. What would you improve if you redesigned one of your projects?
    - **Answer:**
       - If redesigning an IoT monitoring project, using Kafka with Tiered Storage or Confluent Cloud instead of self-managed Kafka on EC2 would be preferable
       - Managing Kafka means handling broker restarts, partition rebalancing, disk space monitoring, and OS patching -- all of which add operational overhead
       - Adding a schema registry from day one to prevent silent data loss from message format changes, and implementing end-to-end tracing with OpenTelemetry for faster debugging are also recommended
       - With Avro and Schema Registry, the producer would have to register the new schema and the consumer would automatically reject incompatible changes, preventing silent data loss
68. What production issue might you face and how would you solve it?
    - **Answer:**
       - One critical production issue is that the Kafka consumer lag starts growing unboundedly during peak hours, reaching 500,000 messages after 4 hours, which means the dashboard is showing data 4 hours old
       - The initial suspicion is that the consumer is too slow, but after analyzing logs, InfluxDB may be throwing `partial write` errors because write batch capacity has been exhausted
       - The root cause can be that the Spring Boot consumer `@KafkaListener` is configured with `concurrency=1` -- all 6 partitions assigned to a single thread
       - Fix by increasing `concurrency=3`, tuning `max.poll.records` from 500 to 200, and adding retry with exponential backoff for InfluxDB writes
       - Grafana panel showed Kafka consumer lag per partition; PagerDuty alert fired when lag exceeded 10,000 messages for more than 5 minutes
69. How do you explain your project to a non-technical person?
    - **Answer:**
       - For a cold-chain monitoring project: "Imagine you have 200 refrigerators spread across a region, each storing products that spoil if the temperature goes wrong for more than a few minutes. Before the system, someone had to walk to each refrigerator and check a thermometer twice a day -- which meant a broken refrigerator could go unnoticed for 12 hours and spoil everything inside. Sensors send temperature data every 30 seconds, and the system automatically alerts the maintenance team the moment something starts going wrong -- often before the product is even affected. This can cut spoiled products by 85% and save 18% on electricity bills."
       - Communication framework: "before and after" -- describe the pain (manual checking, spoilage, late discovery) then the improvement (automated, immediate, data-driven)
70. How do you explain your project architecture to a senior engineer?
    - **Answer:**
       - The architecture can be described as a six-stage event-driven pipeline: IoT gateways publish MQTT to AWS IoT Core (QoS 1), which triggers a Python Lambda that produces to a 6-partition Kafka topic on EC2
       - A Spring Boot Kafka consumer group (3 instances, each handling 2 partitions) validates, deduplicates using Redis SETs with 24-hour TTL, writes time-series data to InfluxDB (with 7-day retention and downsampling), and caches latest readings in Redis
       - Grafana queries InfluxDB for real-time dashboards and alerts, while a React dashboard uses WebSocket for live updates
       - Critical design decisions: Kafka for decoupling (handles bursts, provides replay), Redis for sub-millisecond dedup and caching (avoids InfluxDB load), and InfluxDB automatic downsampling for cost-effective long-term storage
