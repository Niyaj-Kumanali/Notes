# Resume and Project Questions

## Questions

1. Explain your current role at Talentpace Pvt Ltd.
2. What are your main responsibilities in your current project?
3. Which project from your resume are you most confident about?
4. Explain the Lenovo Channel Data Management System.
5. What problem did CDMS solve for Lenovo India?
6. What were the main modules in CDMS?
7. What were your exact responsibilities in CDMS?
8. How did you reduce stored procedure execution time from 5 hours to under 12 minutes?
9. What kind of execution plan issues did you identify?
10. What kind of indexes did you create or modify?
11. How did you validate that optimized stored procedures still returned correct results?
12. How did you test performance under concurrent load?
13. What was the most difficult part of optimizing CDMS?
14. What is partner tagging in CDMS?
15. How did input validation improve reporting accuracy?
16. What were the 4 ETL pipeline stages in CDMS?
17. How did you reduce reporting discrepancies by 90%?
18. What does 60% throughput improvement mean in your project?
19. Explain the Real-Time Partner Inventory Management System.
20. What problem did the inventory validation engine solve?
21. How did you calculate inventory from verified baseline data?
22. What were the common inventory data issues?
23. How did you handle duplicate inventory records?
24. How did you handle invalid serial records?
25. How did you process 10,000+ serial records?
26. How did you reduce inventory processing time to under 15 seconds?
27. How did you measure 94% inventory accuracy?
28. How did you reduce reconciliation effort by 90%?
29. What was the business impact of the inventory system?
30. Explain the Cold-Chain Monitoring System.
31. What was your role in the Cold-Chain Monitoring System?
32. Why was IoT needed in the cold-chain project?
33. How did data flow from gateways to dashboards?
34. Why did you use AWS IoT Core?
35. Why did you use AWS Lambda?
36. Why did you use Kafka in the cold-chain project?
37. Why was Kafka running on EC2?
38. How did you design Kafka topics?
39. How did you decide partition count?
40. How did you handle sensor data bursts?
41. How did you handle delayed IoT messages?
42. How did you handle duplicate IoT messages?
43. How did you handle invalid sensor readings?
44. Why did you use InfluxDB?
45. Why did you use Grafana?
46. What kind of Grafana dashboards did you build?
47. What are excursion alerts?
48. How did you reduce temperature excursions by 85%?
49. How did the system reduce energy costs by 18%?
50. How did you improve load time by 80%?
51. How did you improve responsiveness by 30%?
52. What was the role of Redis in the cold-chain project?
53. How did you secure APIs with Spring Security?
54. How did JWT authentication work in your project?
55. Where did OAuth2 fit in your project?
56. How did you use Swagger/OpenAPI?
57. How did Jenkins reduce deployment time by 70%?
58. How did Docker help in deployment?
59. What was the role of AWS S3?
60. What was the role of PostgreSQL?
61. What was the role of MQTT?
62. What was the role of Databricks?
63. What was the role of DLT?
64. What was the most challenging project you worked on?
65. Which project had the highest business impact?
66. Which project had the most technical complexity?
67. What would you improve if you redesigned one of your projects?
68. What production issue did you face and how did you solve it?
69. How do you explain your project to a non-technical person?
70. How do you explain your project architecture to a senior engineer?

---

## Answers

1. Explain your current role at Talentpace Pvt Ltd.
   - **Answer:**
      - At Talentpace, I work as a Software Engineer where I own backend development end-to-end — from designing REST APIs in Spring Boot to optimizing MSSQL queries that were blocking business reporting
      - I also integrate real-time data pipelines using Kafka and AWS, and I collaborate closely with QA and frontend teams to ensure our deployments are smooth and our APIs are well-documented with Swagger
      - My focus is always on writing code that is maintainable, secure, and performant under load
   - **If asked more:**
      - I'd pick the inventory validation engine and explain how I designed the validation rules in MSSQL, why I chose set-based operations over cursors, and how that decision directly drove the 15-second processing time
2. What are your main responsibilities in your current project?
   - **Answer:**
      - I develop and maintain Spring Boot microservices that serve as the backbone for our partner management and IoT monitoring platforms
      - I spend a significant amount of time analyzing MSSQL execution plans and rewriting stored procedures to eliminate bottlenecks — for instance, I once cut a procedure from 5 hours to 12 minutes just by fixing a nested loop join that should have been a hash match
      - I also handle API security with Spring Security and JWT, write validation logic for incoming data streams, and set up CI/CD pipelines in Jenkins so that deployments are automated and consistent
   - **If asked more:**
      - I start with the slowest query in the pipeline, capture its actual execution plan, identify the highest-cost operator, and only then decide whether an index change, a query rewrite, or a schema adjustment is needed
3. Which project from your resume are you most confident about?
   - **Answer:**
      - I'm most confident about the Lenovo CDMS project because it pushed me deep into MSSQL internals — I wasn't just writing queries, I was reading actual execution plans, understanding row estimates versus actuals, and making surgical changes that had a massive impact
      - Taking a batch process from 5 hours to under 12 minutes taught me that database optimization is less about guessing and more about systematic diagnosis: find the scan, fix the join, add the missing index, verify correctness, then move to the next bottleneck
   - **If asked more:**
      - One specific execution plan anomaly I found was a key lookup causing 4.8 million logical reads on a 50k-row table, and creating a covering index eliminated it entirely
4. Explain the Lenovo Channel Data Management System.
   - **Answer:**
      - CDMS is a data consolidation platform for Lenovo India's channel partner ecosystem
      - It aggregates sales, inventory, and incentive data from hundreds of partners into a single MSSQL-based warehouse, then runs complex ETL pipelines to generate reconciled reports
      - My primary contribution was taking ownership of the most expensive stored procedures — the ones that ran for hours and blocked the nightly reporting window — and systematically optimizing them by analyzing execution plans, restructuring joins, and adding targeted indexes until the entire batch completed in under 12 minutes
   - **If asked more:**
      - The four ETL stages are staging, validation, transformation, and reporting
      - The worst bottleneck was in the transformation stage where a cursor-based loop processed partners one-by-one instead of as a set
5. What problem did CDMS solve for Lenovo India?
   - **Answer:**
      - Before CDMS, Lenovo India's channel team relied on manual Excel-based reconciliation with 200+ partners, which took days and was error-prone
      - CDMS automated the entire data pipeline — ingestion from partner systems, validation against Lenovo's records, incentive calculation, and report generation — so that the business had accurate, audit-ready numbers every morning instead of waiting a week
      - My performance optimization directly ensured that this automated pipeline could complete within the overnight batch window instead of spilling into business hours
   - **If asked more:**
      - The downstream reporting team started at 8 AM, so a 5-hour procedure meant stale data in executive dashboards and delayed decision-making on partner incentives
6. What were the main modules in CDMS?
   - **Answer:**
      - CDMS had four core modules: the Data Ingestion module that pulled raw files from partner SFTP sites, the Validation Engine that checked data quality against Lenovo's master records, the Calculation Engine that ran stored procedures for incentives and discounts, and the Reporting module that generated PDF and Excel outputs for the channel team
      - I worked most heavily on the Calculation Engine — specifically the stored procedures that aggregated partner-level sales and computed tier-based rebates, which were the worst-performing part of the system
   - **If asked more:**
      - Raw partner data went through staging tables, then validation sprocs flagged mismatches, then clean data fed into the calculation sprocs I optimized, and finally the results were materialized into reporting views
7. What were your exact responsibilities in CDMS?
   - **Answer:**
      - I was responsible for optimizing the MSSQL stored procedures that drove the nightly incentive calculation batch — I profiled each one, captured actual execution plans, identified the heaviest operators, and rewrote queries or added indexes as needed
      - I also owned the correctness validation: after every change, I ran the old and new procedures side-by-side, compared row counts and totals at each stage, and only deployed when I could prove bit-for-bit identical results
      - Beyond optimization, I contributed to API development for the reporting dashboard and helped debug production issues when the batch failed mid-cycle
   - **If asked more:**
      - I used checksums and row-count comparison scripts that compared output tables before and after optimization, and I ran them against a production-sized test dataset, not just a subset
8. How did you reduce stored procedure execution time from 5 hours to under 12 minutes?
   - **Answer:**
      - I started by running the procedure with `SET STATISTICS TIME ON` and capturing the actual execution plan from SSMS — that immediately showed me a clustered index scan on a 50-million-row fact table caused by a join predicate that wasn't SARGable
      - I rewrote the join condition to be index-friendly, added two missing non-clustered indexes that the execution plan recommended, and replaced a cursor-based loop with a single set-based `UPDATE` statement
      - Each change was tested independently: I measured the improvement, verified correctness, and only then moved to the next bottleneck
   - **If asked more:**
      - The cursor processed one partner at a time across 200 partners, and replacing it with a single `UPDATE` with a `JOIN` eliminated 199 round trips and reduced the step from 45 minutes to 90 seconds
9. What kind of execution plan issues did you identify?
   - **Answer:**
      - The most common issue was table scans on large fact tables caused by implicit type conversions — a `VARCHAR` column being compared to an `NVARCHAR` parameter forced SQL Server to scan the entire clustered index instead of seeking
      - I also found nested loop joins when the outer table had millions of rows, which should have been hash matches; key lookups on non-covering indexes that added millions of unnecessary page reads; and a cursor-based loop that processed partners one-by-one instead of as a set
   - **If asked more:**
      - I diagnosed implicit conversions by looking for the yellow warning triangles in the execution plan and then checking the actual data types of the columns vs. the parameters in the `WHERE` clause
10. What kind of indexes did you create or modify?
    - **Answer:**
       - I created mostly non-clustered covering indexes that included all columns referenced in the `SELECT`, `WHERE`, and `JOIN` clauses so that SQL Server could satisfy the query entirely from the index without touching the table
       - For one critical procedure, I modified an existing composite index by reordering the key columns — the original had `Status` first and `Date` second, but the query filtered on `Date` ranges and only then on `Status`, so swapping the order turned a scan into a seek
       - I also added filtered indexes for queries that always targeted a specific status value, which kept the index small and fast
    - **If asked more:**
       - I used the `INCLUDE` clause to add payload columns without bloating the index key, and verified improvement by checking that key lookups disappeared from the execution plan
11. How did you validate that optimized stored procedures still returned correct results?
    - **Answer:**
       - I built a validation harness in T-SQL that ran the old and new versions of the procedure side-by-side against the same production-mirror database
       - For every output table, I compared row counts, checksums, and min/max/sum of key numeric columns, and I flagged any discrepancy down to the individual row level
       - I also tested edge cases — empty input sets, duplicate records, NULL values in join columns — to make sure the optimizations didn't silently break business logic
    - **If asked more:**
       - I used `CHECKSUM_AGG(BINARY_CHECKSUM(*))` per table to get a fingerprint of the entire result set and compared fingerprints between old and new runs
12. How did you test performance under concurrent load?
    - **Answer:**
       - I wrote a PowerShell script that launched 10 parallel `sqlcmd` sessions to mimic the real-world scenario where multiple reports kicked off at the same time, and I monitored `sys.dm_exec_requests` to see blocking and wait statistics
       - This exposed a blocking issue where the optimized procedure held a table-level lock — I fixed it by adding index-level locking hints and breaking a long-running transaction into smaller batches
    - **If asked more:**
       - I used `WITH (ROWLOCK)` on the target table to allow concurrent reads, and tested that it didn't degrade performance under single-user load
13. What was the most difficult part of optimizing CDMS?
    - **Answer:**
       - The hardest part was untangling a 400-line stored procedure that had been modified by four different developers over two years — it had nested transactions, GOTO statements, and temp tables that were created and dropped in confusing order
       - I couldn't just rewrite it from scratch because I needed to preserve every business rule
       - I ended up refactoring it step by step: I extracted the cursor logic into a set-based CTE, consolidated the temp tables into a single table variable, and tested each block independently before merging
       - The overall time dropped from 5 hours to 12 minutes, but the refactoring alone took two weeks of careful work
    - **If asked more:**
       - A GOTO-based error handler was masking a real bug — it caught an error, logged it, then jumped back into the middle of the loop, causing duplicate processing of certain partners
14. What is partner tagging in CDMS?
    - **Answer:**
       - Partner tagging is a classification system in CDMS where each channel partner is assigned metadata tags — like region, tier, product specialization, or incentive program — that drive how their data is processed and reported
       - For example, a "Platinum-tier" partner in the "North" region might get different rebate calculations and a different report format than a "Silver-tier" partner in "South"
       - My optimization work directly impacted partner tagging because the stored procedures I optimized used these tags as join and filter criteria, and the original queries had poor index usage on the tag lookup tables
    - **If asked more:**
       - Tags were stored in a normalized `PartnerTag` junction table with `PartnerID` and `TagID`, and a missing composite index on `(TagID, PartnerID)` caused a scan for every tag-based filter
15. How did input validation improve reporting accuracy?
    - **Answer:**
       - Before my validation work, raw partner data was loaded directly into the calculation engine — if a partner sent a duplicate serial number or a date in the wrong format, the procedure would silently produce incorrect incentive totals
       - I added a validation layer in MSSQL that ran before the main calculation: it checked data types, looked for duplicates against the baseline inventory, flagged records that referenced non-existent partners, and quarantined failures into an exception table
       - This stopped bad data from polluting the reports and gave the operations team a clear list of what to fix, which is what drove the 90% reduction in reconciliation effort
    - **If asked more:**
       - For duplicate serial numbers, I used `ROW_NUMBER() OVER (PARTITION BY SerialNumber ORDER BY CreatedDate)` and only kept `RowNumber = 1` for processing
16. What were the 4 ETL pipeline stages in CDMS?
    - **Answer:**
       - Stage 1 was **Extract** — raw files from partner SFTP servers were pulled and staged into flat staging tables with minimal transformation
       - Stage 2 was **Validate** — the validation engine checked data completeness, format correctness, and business rule compliance, flagging failures into exception queues
       - Stage 3 was **Transform** — the clean data was aggregated, joined with master data, and run through the calculation stored procedures I optimized to compute incentives and rebates
       - Stage 4 was **Load** — the final results were written to reporting tables and pushed to the partner portal and executive dashboards
    - **If asked more:**
       - Stage 3 (Transform) was the bottleneck — specifically the incentive calculation procedure that took 5 hours because of the cursor-based partner loop and missing indexes
17. How did you reduce reporting discrepancies by 90%?
    - **Answer:**
       - We achieved that by shifting from manual partner-sent spreadsheets to an automated, validated pipeline where discrepancies were caught at ingestion time rather than at reconciliation time
       - Before CDMS, the operations team would export partner data, manually compare it against Lenovo's records in Excel, and chase partners for corrections — that took days and was error-prone
       - After, if a partner sent a serial number that was already claimed, the system flagged it instantly, held it in an exception table, and the report only included verified records
    - **If asked more:**
       - We tracked the count of post-report manual correction tickets raised by the finance team before and after CDMS, and the quarterly average dropped from 40+ tickets to under 4
18. What does 60% throughput improvement mean in your project?
    - **Answer:**
       - In the cold-chain project, 60% throughput improvement means the system could handle 60% more sensor messages per second from the IoT gateways without any increase in infrastructure cost
       - Before optimization, our Kafka consumers processed about 500 messages per second before lag started building up; after we tuned the consumer batch sizes, adjusted the Spring Boot thread pool, and added Redis caching for duplicate detection, the same consumers handled 800 messages per second with stable lag
    - **If asked more:**
       - The duplicate-detection query was hitting MSSQL on every message, adding 15ms latency per message; moving the dedup check to Redis with a TTL-based set cut that to under 1ms and eliminated the bottleneck
19. Explain the Real-Time Partner Inventory Management System.
    - **Answer:**
       - This system was built to solve a trust problem between Lenovo and its 200+ channel partners — each partner reported their inventory independently, and the numbers never matched Lenovo's baseline
       - I built a centralized MSSQL validation engine that took Lenovo's verified baseline data as the single source of truth and ran every incoming partner inventory record against it
       - The engine validated serial numbers, flagged duplicates, rejected records that didn't match the baseline, and calculated the true inventory in under 15 seconds per partner — all in set-based T-SQL so it scaled linearly
    - **If asked more:**
       - For each partner batch, it joined the incoming serial list against the baseline using `LEFT JOIN` and `NOT EXISTS` to identify valid, duplicate, and invalid records in a single pass
20. What problem did the inventory validation engine solve?
    - **Answer:**
       - The problem was that Lenovo's channel team had no automated way to reconcile inventory across 200+ partners — each partner sent their stock data in different formats, often with duplicates or serials that didn't match Lenovo's distribution records
       - The manual reconciliation process took a team of five people nearly a week every month, and even then, discrepancies of 10-15% were common
       - The validation engine automated the entire process: it ingested partner data, validated every serial number against the trusted baseline, and produced a reconciled inventory count in seconds, cutting reconciliation effort by 90%
    - **If asked more:**
       - One large partner was double-reporting the same 500 serial numbers across two different product lines, inflating their inventory significantly until the engine caught it
21. How did you calculate inventory from verified baseline data?
    - **Answer:**
       - The baseline data was a table of serial numbers that Lenovo had shipped to each partner, with status columns like `Shipped`, `In-Transit`, `Delivered`, and `Sold`
       - When a partner submitted their current inventory, I joined their serial list against the baseline using MSSQL set operations: records in both lists were marked as verified, records in the baseline but missing from the partner list were flagged as potentially sold or lost, and records in the partner list but not in the baseline were flagged as invalid
       - The final inventory count was simply the count of verified `Delivered` serials minus any serials the partner had reported as sold in previous cycles
    - **If asked more:**
       - I used `LAG()` to compare the current submission against the partner's previous submission and automatically detected serials that moved from "in stock" to "missing" without a corresponding sale report
22. What were the common inventory data issues?
    - **Answer:**
       - The most common issue was **duplicate serials** — the same serial number appearing in two different partner submissions, either because partners accidentally double-submitted or because serials were being passed between partners in the gray market
       - The second issue was **invalid serials** — partners reporting serials that Lenovo had never shipped to them, often due to data entry errors or mixing up stock from different distributors
       - Third was **missing serials** — partners reporting fewer units than Lenovo's baseline showed, which could mean unreported sales or theft
    - **If asked more:**
       - If Serial X was shipped to Partner A but reported by Partner B, the system flagged it as a cross-partner exception and escalated to the channel sales manager
23. How did you handle duplicate inventory records?
    - **Answer:**
       - I built a multi-layered deduplication strategy
       - First, within a single partner submission, I used `ROW_NUMBER() PARTITION BY SerialNumber ORDER BY ReportedDate DESC` to keep only the most recent occurrence of each serial
       - Second, across submissions, I maintained a history table of all previously verified serials — if a serial appeared again in a new submission from a different partner, it was flagged as a cross-partner duplicate and quarantined for manual review
       - The dedup logic was entirely in MSSQL and ran as part of the validation stored procedure
    - **If asked more:**
       - The cross-partner dedup was a simple `EXISTS` check against the verified inventory history table indexed on `SerialNumber` with a filtered index for active records only
24. How did you handle invalid serial records?
    - **Answer:**
       - Invalid serials were records submitted by a partner that didn't exist in Lenovo's baseline distribution table at all
       - My validation procedure used a `LEFT JOIN` where the baseline table was on the right side, and if the join produced a NULL baseline serial, the record was flagged as invalid
       - These invalid records were written to an exception table with the partner ID, serial number, and a reason code, and the main inventory calculation excluded them entirely
       - The partner portal displayed these exceptions in real time so the partner could correct their submission and resubmit
    - **If asked more:**
       - The exception flow was: invalid record → exception table → email notification to partner → partner resubmits corrected data → validation re-runs → record moves from exception to verified
25. How did you process 10,000+ serial records?
    - **Answer:**
       - I processed them using set-based T-SQL operations — I loaded all 10,000+ serials into a table-valued parameter in a single batch, then ran validation, dedup, and baseline joins against that set in one pass per partner
       - The key was avoiding RBAR (row-by-agonizing-row) processing: instead of looping through each serial number, I used `MERGE` statements and set-based `UPDATE` with `JOIN` to validate the entire batch in milliseconds
       - Even with 200 partners and 10,000+ serials each, the total processing time stayed under 15 seconds because every operation was index-optimized and set-based
    - **If asked more:**
       - I passed the entire partner submission as a structured TVP with columns `SerialNumber`, `ProductCode`, `ReportedDate`, and MSSQL processed it as a single relational set
26. How did you reduce inventory processing time to under 15 seconds?
    - **Answer:**
       - The original approach processed each serial number individually through a cursor, which cost about 50ms per serial — for a partner with 10,000 serials, that alone was 500 seconds
       - I replaced the cursor with a single `INSERT INTO #validated SELECT s.* FROM @submitted s JOIN Baseline b ON s.SerialNumber = b.SerialNumber` — a set-based operation that validated all 10,000 serials in under 100ms
       - Then I applied the same approach to deduplication and exception handling: every loop was replaced with a set-based join or window function, and I added covering indexes on the `SerialNumber` column in all temp and base tables
    - **If asked more:**
       - The cursor version had 10,000 individual index seeks (one per serial), while the set-based version had a single hash match join with 10,000 input rows
27. How did you measure 94% inventory accuracy?
    - **Answer:**
       - We measured accuracy by comparing the system's validated inventory count against physical stock audits conducted by Lenovo's channel team
       - After processing all 200+ partner submissions through the validation engine, we ran spot-check physical audits for a random sample of 20 partners each quarter
       - The 94% figure means that for those audited partners, the system's inventory count matched the physical count within a 1% tolerance 94% of the time — up from approximately 72% before the validation engine was implemented
    - **If asked more:**
       - We used stratified random sampling by partner tier (Platinum, Gold, Silver) with 95% confidence interval and calculated the weighted average accuracy across all tiers
28. How did you reduce reconciliation effort by 90%?
    - **Answer:**
       - Before the engine, reconciliation meant five people spending 3-4 days manually comparing Excel sheets from 200+ partners against Lenovo's SAP exports
       - After the engine, the entire process was automated: partners submitted data through a portal, the system validated and reconciled in 15 seconds, and the exceptions were displayed in a dashboard
       - The operations team only had to handle the 6% of records that were flagged as exceptions, rather than manually checking 100% of submissions
       - That's where the 90% reduction in effort came from — they went from checking 200,000 serials manually to reviewing 12,000 exceptions
    - **If asked more:**
       - Before: 5 people × 4 days × 8 hours = 160 person-hours per cycle; after: 1 person × 4 hours = 4 person-hours per cycle; (160-4)/160 = 97.5% reduction
29. What was the business impact of the inventory system?
    - **Answer:**
       - The biggest business impact was that Lenovo's channel finance team could finally close the monthly inventory reconciliation in one day instead of one week — that meant partner incentives could be calculated and paid on time, which improved partner satisfaction and reduced disputes
       - The 94% accuracy rate also meant that Lenovo had reliable data for demand forecasting and production planning — they weren't overproducing because of inflated partner-reported inventory numbers
    - **If asked more:**
       - The reduction in overproduction due to accurate inventory data saved significant working capital annually, and the operations team was freed to focus on partner relationships instead of data entry
30. Explain the Cold-Chain Monitoring System.
    - **Answer:**
       - This was a real-time IoT platform that monitored temperature and humidity in cold-storage units across Lenovo's supply chain
       - The data flow started with IoT gateways in each cold room sending MQTT messages to AWS IoT Core, which triggered a Lambda function that published the messages to a Kafka topic running on EC2
       - Our Spring Boot microservices consumed from Kafka, applied validation and deduplication, stored the time-series data in InfluxDB, and served it through REST APIs to a React dashboard and a Grafana instance
       - I worked on the entire backend pipeline — the Kafka consumer configuration, the Spring Boot APIs, the InfluxDB schema design, and the Redis caching layer — and we ultimately reduced temperature excursions by 85% and energy costs by 18%
    - **If asked more:**
       - A sensor reading of 8°C (above the 6°C threshold) → IoT Core → Lambda → Kafka → Spring Boot → InfluxDB write → Grafana alert triggers → operations team gets notified within 30 seconds
31. What was your role in the Cold-Chain Monitoring System?
    - **Answer:**
       - I was responsible for designing and implementing the backend data pipeline — I configured the Kafka consumer group with 6 partitions and 3 consumers to handle sensor bursts, wrote the Spring Boot service that deserialized, validated, and deduplicated incoming messages before writing to InfluxDB, and set up the Redis cache to store the latest reading per sensor so the dashboard could show real-time data without hitting the database
       - I also integrated the Grafana alerting rules that detected temperature excursions and triggered email and Slack notifications, and I containerized the entire Spring Boot application using Docker and set up the Jenkins CI/CD pipeline for automated deployment
    - **If asked more:**
       - A key challenge was tuning the Kafka `max.poll.records` and `fetch.max.wait.ms` to balance latency vs. throughput when sensor data burst from 100 msg/s to 800 msg/s during peak seasons
32. Why was IoT needed in the cold-chain project?
    - **Answer:**
       - IoT was essential because cold-chain inventory is perishable and time-sensitive — a refrigerator unit failing at 2 AM could spoil products worth crores before anyone noticed without automated monitoring
       - Manual temperature logging (someone checking a thermometer twice a day) was unreliable and couldn't detect transient excursions that lasted only 15 minutes
       - IoT gave us continuous, real-time telemetry from every cold-storage unit, automated alerts the moment a threshold was breached, and historical data for root-cause analysis and energy optimization — all without human intervention
    - **If asked more:**
       - We placed 3-4 sensors per cold room (top, middle, bottom, near door) to detect hot spots and door-open events, with readings every 30 seconds
33. How did data flow from gateways to dashboards?
    - **Answer:**
       - The gateway devices in each cold room published JSON payloads over MQTT to AWS IoT Core, which had a rule that forwarded every message to a Lambda function
       - The Lambda did a lightweight parse and then produced the message to a Kafka topic on EC2
       - Our Spring Boot Kafka consumer picked up the message, validated it (checked for null fields, out-of-range values, duplicate message IDs), enriched it with metadata (room name, product type), and wrote it to InfluxDB
       - The Grafana dashboard queried InfluxDB directly for live charts, while the React dashboard called our Spring Boot REST APIs that aggregated data from both InfluxDB and Redis for the most recent readings
    - **If asked more:**
       - We didn't use Lambda straight to InfluxDB because Lambda has a 15-minute timeout and no built-in retry queue for downstream failures, while Kafka gave us persistent buffering, replay capability, and ordered processing
34. Why did you use AWS IoT Core?
    - **Answer:**
       - We used AWS IoT Core because it's a fully managed MQTT broker that handles device authentication, TLS termination, and topic-based routing out of the box — we didn't want to run and scale our own MQTT broker on EC2
       - It also integrates natively with Lambda via IoT rules, so we could process incoming messages without writing any polling code
       - The device shadow feature was useful for tracking the last known state of each gateway, and the IoT Core policies let us restrict each device to only publish to its own topic
    - **If asked more:**
       - Each gateway had an X.509 certificate installed at manufacturing time, and IoT Core validated the certificate on connect and applied the policy that scoped the device to its `/<location>/<room>/<sensor-id>` topic
35. Why did you use AWS Lambda?
    - **Answer:**
       - Lambda served as the lightweight bridge between AWS IoT Core and Kafka — it received the MQTT message from IoT Core via a rule, deserialized the JSON, added a timestamp if missing, and published it to Kafka
       - We chose Lambda over a persistent EC2 consumer because IoT message volume was spiky (near-zero at night, 800 msg/s during peak) and Lambda scales to zero when not needed
       - The function was simple — about 50 lines of Python — and its only job was to transform the MQTT payload into a Kafka-compatible format and handle any parsing errors gracefully
    - **If asked more:**
       - If Lambda failed to parse a message or the Kafka producer threw an exception, the function wrote the raw payload to an S3 bucket with a timestamp and error code for later replay
36. Why did you use Kafka in the cold-chain project?
    - **Answer:**
       - We needed a durable, scalable buffer between the IoT ingestion layer and the downstream analytics pipeline because the data rate was unpredictable — sensors published every 30 seconds per room, but with 200+ rooms the aggregate rate could spike to 800 messages per second
       - Kafka gave us the ability to absorb those spikes without back-pressuring the IoT Core → Lambda path, and it decoupled the producers (Lambda) from the consumers (Spring Boot) so that if the analytics pipeline went down for maintenance, no sensor data was lost
    - **If asked more:**
       - A single topic `cold-chain-sensor-data` with 6 partitions, messages partitioned by `room-id` for ordering, and 3 Spring Boot consumers in the same group for parallel processing
37. Why was Kafka running on EC2?
    - **Answer:**
       - We ran Kafka on EC2 instead of using MSK because the project was cost-sensitive — at the time, MSK's minimum cluster cost was significantly higher than running a single `m5.large` instance with Kafka and Zookeeper containerized in Docker
       - For our throughput requirements (800 msg/s peak, 500 GB retention for 7 days), a single well-tuned EC2 instance was sufficient
       - If the project scaled to 10x the volume, we would have migrated to MSK, but for initial deployment, EC2 was the pragmatic choice
    - **If asked more:**
       - We used gp3 EBS volumes with 3000 IOPS for Kafka logs, JVM heap set to 8 GB, `log.retention.hours=168`, and `unclean.leader.election.enable=false` to prevent data loss
38. How did you design Kafka topics?
    - **Answer:**
       - I designed a single topic called `cold-chain-sensor-data` with 6 partitions — one topic was enough because all sensor messages had the same schema and processing logic
       - The key for partitioning was the `room-id`, which guaranteed that all messages from the same cold room went to the same partition and thus were consumed in order
       - I also created a separate `cold-chain-alerts` topic with 3 partitions for alert events (excursions, device offline) so that the alerting consumers could scale independently of the sensor data consumers
    - **If asked more:**
       - We used `cleanup.policy=delete` for the main topic and `log.retention.ms=604800000` for 7-day retention to support historical replays
39. How did you decide partition count?
    - **Answer:**
       - I chose 6 partitions based on three factors: the target throughput of 800 msg/s, the consumer processing capacity of about 300 msg/s per Spring Boot instance, and the number of unique rooms (about 200)
       - With 6 partitions and 3 consumers (2 partitions per consumer), each consumer handled roughly 270 msg/s, which was well under capacity and left headroom for spikes
       - The partition count also needed to be a multiple of the expected consumer count for balanced distribution
    - **If asked more:**
       - If we had used 100 partitions for 200 rooms, each consumer would handle 33 partitions, and a consumer failure would trigger a full rebalance of 100 partitions, taking 30+ seconds
40. How did you handle sensor data bursts?
    - **Answer:**
       - Sensor data bursts happened primarily during the morning and evening hours when cold-room doors were opened frequently for stock movement, causing the publish rate to spike from ~100 msg/s to ~800 msg/s
       - The architecture handled this naturally: AWS IoT Core accepted the burst and queued messages to Lambda, Lambda scaled up concurrency, and Kafka accepted the increased write rate because it had enough partition capacity
       - On the consumer side, I configured `max.poll.records=500` and `fetch.max.wait.ms=500` so that consumers fetched larger batches during bursts but still maintained low latency during normal periods
    - **If asked more:**
       - If Kafka consumer lag grew beyond 10,000 messages, a health check endpoint in Spring Boot reduced the consumer's `max.poll.records` dynamically to prevent memory exhaustion
41. How did you handle delayed IoT messages?
    - **Answer:**
       - Delayed messages occurred when a gateway lost network connectivity for a few minutes and then reconnected, sending all buffered messages at once with original timestamps
       - I handled this at the InfluxDB write layer: each data point was tagged with `ingestion_time` (when Spring Boot received it) and `sensor_time` (when the gateway recorded it)
       - InfluxDB stored the data by `sensor_time`, so delayed messages were written to their correct time bucket even if they arrived late
    - **If asked more:**
       - Each gateway synced its clock via NTP every hour, and if the clock skew was greater than 5 seconds, the messages carried a `clock_error` flag stored as an InfluxDB tag for data-quality filtering
42. How did you handle duplicate IoT messages?
    - **Answer:**
       - Duplicate messages happened when a gateway retransmitted a message because it didn't receive an MQTT QoS 2 acknowledgment, or when Lambda's at-least-once delivery to Kafka produced duplicates
       - I implemented deduplication in the Spring Boot consumer using a combination of a unique `message_id` (UUID generated by the gateway) and a Redis SET with TTL
       - When a message arrived, the consumer checked if its `message_id` already existed in Redis: if yes, the message was skipped; if no, it was added to Redis with a 24-hour TTL and then written to InfluxDB
    - **If asked more:**
       - A primary-key check on MSSQL would have added 5-10ms latency per message; Redis with in-memory SET operations was orders of magnitude faster at 800 msg/s
43. How did you handle invalid sensor readings?
    - **Answer:**
       - I defined a set of validation rules in the Spring Boot consumer that ran before writing to InfluxDB: temperature had to be between -20°C and 60°C, humidity between 0% and 100%, and the `sensor_time` had to be within 5 minutes of the current time
       - If a message failed validation, it was written to a separate `invalid-readings` Kafka topic, and the Grafana dashboard had a panel showing the count of invalid readings per room over time to help identify failing sensors
    - **If asked more:**
       - A sensor once started reporting -40°C due to a hardware fault; the validation rule caught it, the invalid reading was logged, and the maintenance team replaced the sensor before it caused a false temperature excursion alert
44. Why did you use InfluxDB?
    - **Answer:**
       - We chose InfluxDB because it's purpose-built for time-series data — it handles millions of data points per second with automatic downsampling, retention policies, and continuous queries that would be impractical in a relational database
       - Each sensor reading (temperature, humidity, timestamp, room-id) was stored as a single point, and InfluxDB's columnar storage compressed the data by 10x
       - It also integrated natively with Grafana, so our dashboard queries were simple Flux queries without custom API middleware
    - **If asked more:**
       - Raw data was kept for 7 days at 30-second resolution, then downsampled to 5-minute averages for 30 days, and hourly averages for 1 year, using InfluxDB tasks
45. Why did you use Grafana?
    - **Answer:**
       - Grafana gave us production-ready dashboards with minimal effort — we connected it directly to InfluxDB and built real-time panels for temperature trends, humidity charts, and excursion alerts in a few hours
       - The alerting engine was critical: we configured Grafana to evaluate alert rules every 30 seconds, and when a temperature reading exceeded the threshold for more than 2 consecutive evaluations, it sent notifications via Slack and email
    - **If asked more:**
       - The main dashboard had a top row with real-time temperature gauges per cold room, a middle row with 24-hour trend lines with excursion zones highlighted in red, and a bottom row with alert history
46. What kind of Grafana dashboards did you build?
    - **Answer:**
       - I built three main dashboards
       - The **Operations Dashboard** showed a grid of all 200+ cold rooms, each with a color-coded tile (green=normal, yellow=warning, red=excursion), and clicking a tile opened a detailed 24-hour trend chart
       - The **Alerts Dashboard** listed all active and historical excursions with room name, threshold breached, duration, and resolution time
       - The **Energy Dashboard** displayed power consumption trends per cold room with a temperature overlay — this helped the team identify rooms that were overcooling and wasting energy
    - **If asked more:**
       - Operators could see that Room A consumed 30% more power than Room B for the same temperature range, leading them to discover a faulty door seal that was letting cold air escape, which drove the 18% cost reduction
47. What are excursion alerts?
    - **Answer:**
       - An excursion alert fires when a sensor reading stays outside an acceptable temperature or humidity range for longer than a configured duration
       - For example, if a vaccine cold room had a threshold of 2°C to 8°C, and the sensor reported 9°C for three consecutive readings (90 seconds), the system classified that as a minor excursion
       - If the temperature exceeded 10°C or stayed outside range for more than 5 minutes, it became a major excursion and triggered immediate Slack and SMS alerts to the on-call engineer
    - **If asked more:**
       - Severity levels were: minor (informational, auto-resolved), major (requires acknowledgment within 15 minutes), critical (requires immediate action, escalates to manager if unacknowledged after 5 minutes)
48. How did you reduce temperature excursions by 85%?
    - **Answer:**
       - The 85% reduction came from shifting from reactive to proactive monitoring
       - Before the system, the operations team didn't know about an excursion until someone opened the cold room hours later
       - With real-time monitoring and alerts, the team could respond to a rising temperature trend within 30 seconds — they could check if a door was left open, if the compressor had failed, or if the setpoint had been changed
       - Additionally, historical data helped the team identify patterns, like solar heat gain through a window at 2 PM, and fix the root cause permanently
    - **If asked more:**
       - Measured as (excursions before system - excursions after system) / excursions before system × 100, comparing 6 months before deployment to 6 months after using the same set of cold rooms
49. How did the system reduce energy costs by 18%?
    - **Answer:**
       - The energy savings came from data-driven optimization of cooling setpoints
       - Historical InfluxDB data showed that several rooms were being overcooled — kept at 2°C when the product only required 6°C, which wasted significant energy
       - By analyzing temperature trends alongside power consumption, the operations team identified the optimal setpoint for each room based on its insulation quality, ambient temperature, and product requirements
       - Fixing issues like faulty door seals and insulation gaps directly reduced compressor runtime
    - **If asked more:**
       - The average cold room consumed about ₹15,000/month in electricity before; optimizing setpoints and fixing seals dropped it to ₹12,300/month, saving ₹2,700/month per room across 200 rooms
50. How did you improve load time by 80%?
    - **Answer:**
       - I improved the React dashboard's initial load time by implementing server-side pagination and lazy loading for the cold-room grid — originally, the API was returning all 200+ rooms with full sensor history, which was about 15 MB of JSON and took 12 seconds to parse and render
       - I changed the API to return only the latest reading per room with a summary status (green/yellow/red), which cut the payload to under 50 KB, and loaded room details on demand when the user clicked a tile
       - I also enabled Redis caching for the aggregate dashboard data with a 10-second TTL
    - **If asked more:**
       - API response time went from 3.2 seconds to 180ms, React render time from 8.5 seconds to 400ms, total perceived load time from ~12 seconds to ~2 seconds
51. How did you improve responsiveness by 30%?
    - **Answer:**
       - The 30% responsiveness improvement refers to the time between a sensor reading arriving at the backend and the dashboard reflecting that new value
       - The bottleneck was the polling interval: the React dashboard was polling the REST API every 15 seconds
       - I switched to WebSocket connections — the Spring Boot service pushed new readings to connected clients as soon as they were written to InfluxDB, which reduced update latency to under 500ms
    - **If asked more:**
       - I used Spring Boot's `WebSocketHandler` with a `SimpMessagingTemplate` that broadcast the latest reading to the `/topic/latest-readings` channel whenever a new data point was persisted
52. What was the role of Redis in the cold-chain project?
    - **Answer:**
       - Redis served three purposes
       - First, it was the **deduplication cache**: we stored `message_id` for each incoming message with a 24-hour TTL, so duplicate messages were detected in under 1ms
       - Second, it was the **latest-reading cache**: for each sensor, we stored the most recent valid reading so the dashboard's summary API could return the current state of all 200+ rooms instantly
       - Third, it was the **session store** for Spring Session — user sessions were stored in Redis so that any Spring Boot instance could serve any request without losing session state
    - **If asked more:**
       - The dedup cache stored ~200,000 keys at ~100 bytes each = ~20 MB; the latest-reading cache stored 600 keys at ~200 bytes each = ~120 KB — well within a single t3.micro instance's 500 MB allocation
53. How did you secure APIs with Spring Security?
    - **Answer:**
       - I configured Spring Security with a JWT-based authentication filter that intercepted all API requests except the public health-check endpoint
       - The filter extracted the `Authorization: Bearer <token>` header, validated the token's signature using an RSA public key, checked the token's expiration and issuer claims, and then set the `SecurityContext` with the user's roles and permissions
       - I also applied method-level security using `@PreAuthorize` annotations — admin endpoints were restricted to `ROLE_ADMIN`, dashboard data to `ROLE_VIEWER`
       - For the IoT ingestion API, I used API keys instead of JWT because sensors couldn't manage token refresh
    - **If asked more:**
       - The React app received an access token with 15-minute expiry and a refresh token with 7-day expiry; the frontend used an Axios interceptor to automatically refresh the access token when it expired
54. How did JWT authentication work in your project?
    - **Answer:**
       - When a user logged in through the React login page, the Spring Boot authentication endpoint validated the username/password against the MSSQL user table and returned a signed JWT access token (valid for 15 minutes) and a refresh token (valid for 7 days)
       - The JWT contained the user's ID, roles, and a session ID in its claims, signed with an RSA private key so any service could verify the token using the public key
       - The React app stored the token in `localStorage` and sent it in the `Authorization` header for every API call
    - **If asked more:**
       - We used RSA instead of HMAC because with HMAC every service needs to share the same secret key; with RSA only the authentication service holds the private key
55. Where did OAuth2 fit in your project?
    - **Answer:**
       - We used OAuth2 for Grafana authentication integration — instead of managing separate Grafana user accounts, we configured Grafana to use OAuth2 with our Spring Boot application as the authorization server
       - When a user logged into Grafana, they were redirected to our login page, authenticated with their existing credentials, and Grafana received an access token for API calls
    - **If asked more:**
       - The `oauth2` section in `grafana.ini` had `auth_url`, `token_url`, and `api_url` pointing to our Spring Boot endpoints, with role mapping from JWT claims
56. How did you use Swagger/OpenAPI?
    - **Answer:**
       - I used SpringDoc OpenAPI to auto-generate Swagger documentation for all our REST APIs
       - The documentation was available at `/swagger-ui.html` and `/v3/api-docs` for each microservice
       - The Swagger UI became the primary reference for the frontend team — they could see every endpoint, its schema, authentication requirements, and test calls directly from the browser
       - The frontend team used the OpenAPI spec to generate TypeScript API clients using `openapi-generator`, which eliminated manual type definitions
    - **If asked more:**
       - We used `@Tag` annotations to group endpoints by domain (Sensor Data, Alerts, Configuration, Users), each with its own section in Swagger UI
57. How did Jenkins reduce deployment time by 70%?
    - **Answer:**
       - Before Jenkins, deployments were manual: build the JAR locally, SCP it to EC2, SSH in, stop the service, replace the JAR, restart, verify — about 30 minutes per deployment
       - I set up a Jenkins pipeline that automated the entire process: on every push to `main`, Jenkins checked out code, ran tests, built the Docker image, pushed it to Docker Hub, SSH'd into EC2, pulled the new image, stopped the old container, started the new one, and ran a smoke test — all in under 10 minutes
       - The pipeline also included automatic rollback if the smoke test failed
    - **If asked more:**
       - The Jenkinsfile had stages: Checkout → Test → Build → Dockerize → Deploy → Smoke Test → (on failure) Rollback, with post-build Slack notifications
58. How did Docker help in deployment?
    - **Answer:**
       - Docker eliminated the "it works on my machine" problem by packaging the Spring Boot application, its dependencies, and the JVM version into a single container image that ran identically on the developer's laptop, the Jenkins build server, and the production EC2 instance
       - It also made deployments atomic and reversible — we tagged each image with the Git commit hash, and rolling back was as simple as running with the previous image tag
       - We ran four containers on a single EC2 instance: Spring Boot API, Redis, Grafana, and Nginx — all managed with Docker Compose
    - **If asked more:**
       - We used a custom bridge network `cold-chain-net` so containers could communicate by service name (e.g., `api:8080`, `redis:6379`) instead of environment-specific IP addresses
59. What was the role of AWS S3?
    - **Answer:**
       - In the cold-chain project, S3 served as the long-term archival store for raw sensor data
       - While InfluxDB kept high-resolution data for 7 days and downsampled data for up to a year, all raw sensor messages were archived to S3 in Parquet format using a nightly batch job
       - This served two purposes: compliance (we needed to retain raw data for 3 years) and reprocessing (if we discovered a bug in our data pipeline, we could replay archived data)
       - Each object was keyed by `year/month/day/room-id/hour.parquet` for efficient range queries
    - **If asked more:**
       - A Spring Boot scheduled task ran at midnight, queried the last 24 hours of raw data from a tracking table in MSSQL, serialized it to Parquet, and uploaded to S3 with server-side encryption
60. What was the role of PostgreSQL?
    - **Answer:**
       - We used PostgreSQL as the metadata and configuration database for the cold-chain project — it stored user accounts, role assignments, cold-room metadata, alert configuration rules, and audit logs
       - We chose PostgreSQL over MSSQL for this non-time-series workload because the operational overhead was lower and the `JSONB` column type was convenient for storing flexible alert rule configurations
       - The Spring Boot application used JPA/Hibernate to interact with PostgreSQL, and InfluxDB queries referenced room IDs that were joined against PostgreSQL metadata in Grafana
    - **If asked more:**
       - The `alert_rules` table had columns `room_id`, `metric` (temperature/humidity), `operator` (gt/lt), `threshold`, `duration_seconds`, `severity`, and a `channels` JSONB column for notification targets
61. What was the role of MQTT?
    - **Answer:**
       - MQTT was the communication protocol between the IoT gateways and AWS IoT Core
       - We chose MQTT over HTTP because it's lightweight (the binary header is only 2 bytes), supports persistent connections with minimal overhead, and has built-in QoS levels — we used QoS 1 (at-least-once delivery) for reliable delivery
       - Each gateway published messages to topics like `cold-chain/{location-id}/{room-id}/{sensor-id}`, and IoT Core used topic filtering to route each location's data to the appropriate Lambda function
    - **If asked more:**
       - We had 600 unique topics across 20 locations, 10 rooms per location, and 3 sensors per room; IoT Core wildcard subscription `cold-chain/+/+/+` captured all messages
62. What was the role of Databricks?
    - **Answer:**
       - Databricks was used for offline analytics and reporting on aggregated cold-chain data
       - While Grafana handled real-time operational dashboards, the business team needed weekly and monthly reports on energy consumption trends, excursion patterns by location, and sensor reliability statistics
       - We exported downsampled InfluxDB data to S3 as Parquet files daily, and Databricks notebooks read that data to generate trend analyses and predictive models
    - **If asked more:**
       - One use case was a linear regression on energy consumption vs. ambient temperature for each room, identifying rooms where the slope was abnormally high (indicating poor insulation) and generating a prioritized maintenance work order list
63. What was the role of DLT?
    - **Answer:**
       - DLT (Delta Live Tables) in Databricks was used to build and maintain the ETL pipeline that transformed raw sensor archives into clean, analytics-ready tables
       - Instead of writing manual Spark jobs with fragile scheduling, DLT let us define the pipeline declaratively — we specified the source (S3 Parquet files), the transformations (deduplication, filtering invalid readings, joining with room metadata), and the target Delta tables, and DLT handled incremental processing, data quality checks, and automatic retries
    - **If asked more:**
       - The "bronze" table ingested raw Parquet from S3, the "silver" table applied deduplication and validation, and the "gold" table joined with room metadata for the business reporting layer
64. What was the most challenging project you worked on?
    - **Answer:**
       - The CDMS stored procedure optimization was the most challenging because it wasn't just a technical problem — I had to understand the business meaning of every column, every join, and every temp table in a 400-line procedure that nobody had fully documented
       - The pressure was high because the nightly batch had been failing for weeks, and the business team needed their reports
       - I had to balance speed with safety — I couldn't afford to introduce a calculation error that would affect partner incentives
       - The approach of making one change at a time, testing it against the full dataset, and only then moving to the next bottleneck was what made it work
    - **If asked more:**
       - After three days of analysis, I found that a `WHERE CAST(date_column AS DATE) = '2024-01-01'` was forcing a full table scan because the `CAST` made the predicate non-SARGable; rewriting it to a range condition alone cut 45 minutes from the runtime
65. Which project had the highest business impact?
    - **Answer:**
       - The inventory validation engine had the highest direct business impact because it saved Lenovo's channel finance team an entire week of manual work every month and gave them trustworthy inventory numbers for the first time
       - Before the system, the finance team couldn't close the monthly books on time because inventory discrepancies took days to resolve
       - After the system, reconciliation was a 15-second automated process with a clear audit trail, and the finance team could close the month on the first working day
    - **If asked more:**
       - One quarter before the system, a partner dispute over 2,000 serial numbers took 3 weeks of email exchanges and physical audits to resolve; after the system, the same scenario was resolved in 15 seconds
66. Which project had the most technical complexity?
    - **Answer:**
       - The cold-chain IoT monitoring system was the most technically complex because it involved the widest range of technologies and failure modes — hardware failures (sensors dying, gateways disconnecting), network issues (MQTT disconnections, Kafka broker restarts), data quality problems (duplicates, delayed messages, corrupt payloads), and real-time performance requirements all at once
       - It wasn't enough to make each component work individually — they all had to work together reliably, and a failure in any layer could cascade
    - **If asked more:**
       - One complex failure was a Kafka broker disk filling up because retention wasn't configured correctly after a schema change increased message size by 3x; consumers couldn't commit offsets, which caused rebalancing and duplicate downstream writes
67. What would you improve if you redesigned one of your projects?
    - **Answer:**
       - If I redesigned the cold-chain project, I would use Kafka with Tiered Storage or Confluent Cloud instead of self-managed Kafka on EC2
       - Managing Kafka ourselves meant handling broker restarts, partition rebalancing, disk space monitoring, and OS patching — all of which added operational overhead
       - I would also add a schema registry from day one to prevent silent data loss from message format changes, and implement end-to-end tracing with OpenTelemetry for faster debugging
    - **If asked more:**
       - With Avro and Schema Registry, the producer would have to register the new schema and the consumer would automatically reject incompatible changes, preventing silent data loss
68. What production issue did you face and how did you solve it?
    - **Answer:**
       - One critical production issue was that the Kafka consumer lag started growing unboundedly during peak hours, reaching 500,000 messages after 4 hours, which meant the dashboard was showing data 4 hours old
       - I initially suspected the consumer was too slow, but after analyzing logs, I found that InfluxDB was throwing `partial write` errors because we had exhausted write batch capacity
       - The root cause was that the Spring Boot consumer's `@KafkaListener` was configured with `concurrency=1` — all 6 partitions were assigned to a single thread
       - I fixed it by increasing `concurrency=3`, tuning `max.poll.records` from 500 to 200, and adding retry with exponential backoff for InfluxDB writes
    - **If asked more:**
       - We had a Grafana panel showing Kafka consumer lag per partition, and a PagerDuty alert fired when lag exceeded 10,000 messages for more than 5 minutes
69. How do you explain your project to a non-technical person?
    - **Answer:**
       - For the cold-chain project: "Imagine you have 200 refrigerators spread across India, each storing products that spoil if the temperature goes wrong for more than a few minutes. Before our system, someone had to walk to each refrigerator and check a thermometer twice a day — which meant a broken refrigerator could go unnoticed for 12 hours and spoil everything inside. We installed sensors that send temperature data every 30 seconds, and our system automatically alerts the maintenance team the moment something starts going wrong — often before the product is even affected. This cut spoiled products by 85% and saved 18% on electricity bills."
    - **If asked more:**
       - I follow the "before and after" framework — describe the pain before (manual checking, spoilage, late discovery) and the improvement after (automated, immediate, data-driven)
70. How do you explain your project architecture to a senior engineer?
    - **Answer:**
       - I would describe the cold-chain architecture as a six-stage event-driven pipeline: IoT gateways publish MQTT to AWS IoT Core (QoS 1), which triggers a Python Lambda that produces to a 6-partition Kafka topic on EC2
       - A Spring Boot Kafka consumer group (3 instances, each handling 2 partitions) validates, deduplicates using Redis SETs with 24-hour TTL, writes time-series data to InfluxDB (with 7-day retention and downsampling), and caches latest readings in Redis
       - Grafana queries InfluxDB for real-time dashboards and alerts, while a React dashboard uses WebSocket for live updates
    - **If asked more:**
       - The critical design decisions were: Kafka for decoupling (handles bursts, provides replay), Redis for sub-millisecond dedup and caching (avoids InfluxDB load), and InfluxDB's automatic downsampling for cost-effective long-term storage
