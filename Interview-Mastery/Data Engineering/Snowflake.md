# Snowflake Interview Notes

Comprehensive interview preparation notes for Snowflake covering architecture, compute, storage, data loading, security, performance, and cost management.

---

## What is Snowflake

### Overview
- Snowflake is a **cloud-native data warehouse** delivered as a fully managed **SaaS (Software as a Service)** platform.
- It is **not** built on top of an existing database engine (like Redshift is on PostgreSQL). Snowflake wrote its own storage, compute, and query engine from scratch.
- Runs natively on all three major clouds:
  - **Amazon Web Services (AWS)**
  - **Microsoft Azure**
  - **Google Cloud Platform (GCP)**
- You never install, patch, or manage infrastructure. Snowflake handles servers, storage, networking, upgrades, and maintenance for you.
- Querying is done with **standard SQL**, and all metadata, configuration, and administration happens through a web UI (Snowsight), CLI (SnowSQL), REST API, or connectors.

### Core Value Propositions
- **Elastic scaling** — warehouses scale up or down (and even pause) independently of storage.
- **Multi-cluster / multi-user concurrency** — many workloads run at once without interference.
- **Near-zero administration** — no indexing, no vacuuming, no manual tuning of physical storage.
- **Pay-per-use pricing** — you pay for the compute you actually consume (per-second billing with a 60-second minimum) plus storage for data stored.
- **Instant, secure data sharing** — share live data between accounts without copying it.

### Who Uses Snowflake
- Data engineers (ELT pipelines, transformations, data modeling).
- Data analysts / BI teams (running dashboards and ad-hoc analytics).
- Data scientists (exploring large datasets, feeding ML feature stores).
- Used as a central **data warehouse / data cloud** layer in modern data stacks, often combined with dbt, Airflow, Fivetran, and Looker.

### ELT vs ETL
- Snowflake encourages **ELT (Extract, Load, Transform)**:
  - Load raw data first (COPY INTO or Snowpipe).
  - Transform inside Snowflake using SQL (often orchestrated with dbt).
- No need for a separate transformation server.

---

## Snowflake Architecture

### The Three-Layer Architecture
Snowflake's architecture is famously divided into three independent layers:

1. **Storage Layer (Storage / Data Layer)**
   - Where all data lives, in cloud object storage (S3, Azure Blob, GCS).
   - Data is stored in **columnar format** within **micro-partitions**.
   - Snowflake manages the layout entirely — no indexes, no manual partitioning required.
   - Only Snowflake computes against this storage; it is not directly accessible to users.

2. **Compute Layer (Query Processing / Virtual Warehouses)**
   - Made up of **virtual warehouses** — independent clusters of compute (VMs).
   - Each warehouse is a separate MPP (Massively Parallel Processing) cluster.
   - Executes queries, runs DML (INSERT, UPDATE, DELETE, MERGE), and processes transformations.
   - Warehouses can be scaled up (bigger machines) or scaled out (more machines) and can be suspended when idle.

3. **Cloud Services Layer (Services Layer)**
   - The "brain" of Snowflake: a shared, stateless set of services that coordinate everything.
   - Responsibilities include:
     - **Metadata management** (schemas, tables, privileges, usage stats).
     - **Query parsing and optimization** — builds the execution plan.
     - **Security and authentication** (login, role checks, encryption keys).
     - **Transaction management** — ACID guarantees across distributed compute.
     - **Concurrency control** — lock and multi-version coordination.
   - Because this layer is stateless, it scales automatically and does not slow down with more warehouses or users.

### Separation of Compute and Storage
- Storage and compute are **decoupled**: they scale independently.
- A warehouse can be paused to **zero** while your data still sits safely in storage — you pay nothing for compute during that time.
- You can spin up a huge warehouse to process a massive query, then shut it down. No data is lost or migrated.
- Multiple warehouses can all query the **same** underlying storage concurrently with **zero contention** — each warehouse reads the shared data independently.

### Why This Architecture Matters
- **Independent scaling**: scale compute without touching storage, and vice versa.
- **Cost control**: don't pay for idle compute — suspend warehouses, resume on demand.
- **Concurrency**: multiple workloads (ingestion, BI, ML) each get their own warehouse with no performance interference.
- **Elasticity**: add warehouses instantly; no re-provisioning or downtime.
- **No data movement**: cloning, sharing, and time travel work on metadata because there is no physical data copy required for these operations.

---

## Virtual Warehouses

### What is a Virtual Warehouse
- A virtual warehouse (VW) is an **independent, elastic cluster of compute resources** used to execute queries and DML.
- Each warehouse contains multiple **nodes (VMs)**, and each node has CPU, memory, and local SSD cache.
- Warehouses are fully isolated from each other: a long-running heavy query on Warehouse A does not slow down queries on Warehouse B.
- You can have many warehouses per account, sized differently for different workloads (e.g., a small warehouse for dashboards, a big one for nightly ELT).

### Warehouse Sizing
- Sizes range from **X-Small to 6X-Large** (and beyond in some editions).
- Doubling the size roughly **doubles the compute power** (more CPU, more memory, more local cache, more parallelism).
- Size examples:
  - **X-Small**: 1 node
  - **Small**: 2 nodes
  - **Medium**: 4 nodes
  - **Large**: 8 nodes
  - **X-Large**: 16 nodes
  - **2X-Large**: 32 nodes
  - ... up to **6X-Large** for very heavy workloads.
- Larger warehouses also have **more local SSD cache**, so repeated scans of hot data can be served from cache instead of cloud storage.

### Scaling Up vs Scaling Out
- **Scaling up (vertical)**: change warehouse size to a larger/smaller T-shirt size. Good for single large queries that need more parallelism and memory.
- **Scaling out (horizontal / multi-cluster)**: add more clusters of the same size. Good for **concurrency** — many simultaneous queries from many users.
- A **multi-cluster warehouse** has:
  - A `min_cluster_count` (minimum clusters running).
  - A `max_cluster_count` (maximum clusters that can be spun up).
  - `scaling_policy = STANDARD` (adds clusters when a queue forms) or `ECONOMY` (less aggressive about adding clusters).
- Multi-cluster warehouses are ideal for BI workloads with unpredictable user spikes (e.g., Monday morning dashboard traffic).

### Auto-Suspend and Auto-Resume
- **Auto-suspend**: warehouse automatically shuts down after a configurable period of inactivity (default 10 minutes, can be 5 minutes or less, minimum 1 minute in some cases).
- **Auto-resume**: when a query arrives while the warehouse is suspended, the warehouse automatically starts up again (adds ~2-5 seconds startup latency).
- Example SQL:

```sql
CREATE OR REPLACE WAREHOUSE analytics_wh
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 300          -- suspend after 5 minutes of inactivity
  AUTO_RESUME = TRUE          -- auto-start when a query comes in
  INITIALLY_SUSPENDED = TRUE; -- start in suspended state to save credits
```

- Best practice: set `AUTO_SUSPEND` aggressively (300 seconds or less) to avoid paying for idle compute.

### Cost Considerations
- Virtual warehouses are the **primary source of credit consumption** in Snowflake.
- Credits are consumed only while a warehouse is **running**, including while it sits idle (unless suspended).
- Per-second billing (with a 60-second minimum) for queries executed on a warehouse.
- Larger warehouses consume **2x credits per second** for each size doubling.
- Multi-cluster warehouses multiply credit burn by the number of active clusters.
- Key cost levers:
  - Choose the right size (start small, scale only when needed).
  - Set aggressive auto-suspend.
  - Suspend warehouses manually during off-hours.
  - Use multi-cluster only for real concurrency needs.

---

## Micro-partitions

### What are Micro-partitions
- Micro-partitions are the **fundamental unit of storage** in Snowflake — small, contiguous blocks of data.
- Each micro-partition contains between **50 MB and 500 MB** (compressed) of data.
- Snowflake automatically splits a table's rows into micro-partitions as data is loaded or modified.
- No user configuration is required — micro-partitions are created and managed automatically.

### Columnar Storage Within Micro-partitions
- Inside each micro-partition, data is stored **column by column** (columnar / column-oriented storage), not row by row.
- Each column is **compressed independently** (different compression applied per column based on data type and value distribution).
- Benefits of columnar storage:
  - Queries that read only a few columns read **only those columns**, skipping the rest.
  - Better compression → less storage cost and less data to scan.
  - Ideal for analytical (aggregation-heavy) workloads.

### Columnar Metadata
- For every column in every micro-partition, Snowflake stores **metadata**:
  - Min and max values.
  - Number of distinct values.
  - Null count.
  - Average and total size.
- This metadata is what powers **pruning** — Snowflake can skip entire micro-partitions without even reading them.

### Data Clustering
- **Clustering** refers to how well rows with similar values are grouped together within micro-partitions.
- Clustering is **automatic and continuous** in Snowflake:
  - As data is loaded, Snowflake tracks how well columns are clustered.
  - If clustering degrades, Snowflake may automatically re-cluster the table in the background (this consumes credits).
- **Clustering depth** measures clustering quality: lower is better.
- You can improve clustering on specific columns by defining **clustering keys** (see the Tables section).

### Pruning (Micro-partition Elimination)
- **Pruning** is the process of skipping micro-partitions that do not contain rows relevant to a query.
- When a query has a predicate like `WHERE date = '2024-01-15'`, Snowflake:
  1. Reads the min/max metadata for each micro-partition.
  2. Drops any micro-partition whose min/max range does not overlap the predicate.
  3. Only scans the remaining (few) micro-partitions.
- Because micro-partitions are small and numerous, pruning is extremely fine-grained.

### Why Queries Are Fast
- Queries are fast because Snowflake **never scans entire tables** when it can avoid it:
  - **Pruning** eliminates irrelevant micro-partitions using column metadata.
  - **Columnar format** limits reads to the columns actually referenced.
  - **Local caching** on warehouse nodes serves recently queried data.
  - Queries are parallelized across all nodes in the warehouse, each handling a slice of the micro-partitions.

### Example
```sql
-- A table with 1 year of data (365 micro-partitions worth).
-- This query prunes down to only the partitions containing January data:
SELECT region, SUM(sales)
FROM fact_sales
WHERE sale_date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY region;
```

---

## Snowflake Tables

### Table Types
- **Permanent tables**: default; full protection including time travel (up to 90 days) and fail-safe (7 days).
- **Transient tables**: survive until explicitly dropped but have **no fail-safe** and a reduced time-travel window (1 day default, up to 1 day; can be 0-1 days depending on edition... actually 0 to 1 day for transient, or up to 90 in some configurations). Used for temporary/intermediate data where recovery is not needed.
- **Temporary tables**: exist only within a session and vanish when the session ends. No fail-safe, minimal time travel. Good for staging within a session.
- Summary table:

| Type | Persistence | Fail-safe | Time Travel | Use case |
|------|------------|-----------|-------------|----------|
| Permanent | Until dropped | Yes (7 days) | Up to 90 days | Production tables |
| Transient | Until dropped | No | 0-1 days (configurable up to 90 in some editions) | Intermediate/staging |
| Temporary | Current session only | No | Within session | Session-local work |

### Create Table Syntax

```sql
CREATE OR REPLACE TABLE customers (
  customer_id    NUMBER(10,0)   NOT NULL,
  first_name     VARCHAR(50),
  last_name      VARCHAR(50),
  email          VARCHAR(200),
  signup_date    DATE,
  is_active      BOOLEAN DEFAULT TRUE
)
CLUSTER BY (signup_date); -- optional clustering key
```

- Other common DDL:
```sql
-- Create table from a query
CREATE TABLE top_customers AS
SELECT customer_id, SUM(amount) AS total
FROM orders
GROUP BY customer_id
ORDER BY total DESC
LIMIT 1000;

-- Drop table (recoverable via UNDROP within time travel window)
DROP TABLE customers;

-- Truncate (faster than DELETE for removing all rows)
TRUNCATE TABLE customers;

-- Alter
ALTER TABLE customers ADD COLUMN loyalty_tier VARCHAR(20);
ALTER TABLE customers SET CLUSTER KEY (signup_date);
```

### Clustering Keys
- A clustering key is a column (or set of columns) that Snowflake uses to improve micro-partition pruning.
- Use clustering keys on:
  - **Large tables** (billions of rows).
  - Columns frequently used in **WHERE filters**.
  - Columns where natural clustering is poor (e.g., monotonically increasing IDs loaded out of order).
- Example:
```sql
CREATE TABLE orders (
  order_id NUMBER,
  customer_id NUMBER,
  order_date DATE,
  total NUMBER
) CLUSTER BY (order_date, customer_id);
```
- Clustering keys trade off some load-time/credit cost (background re-clustering) for much faster queries.

### Clone and Time Travel
- **Time Travel** lets you query or recreate data as it existed at a past point in time.
- **Clone (CLONE)** makes a snapshot copy of a table (or schema, or database) at the current time or at a past time via time travel.
- Cloning is a **metadata-only operation** — no physical data is copied (see Zero-Copy Cloning below).

```sql
-- Clone current state
CREATE TABLE customers_backup CLONE customers;

-- Clone state as of 2 hours ago
CREATE TABLE customers_backup CLONE customers AT (OFFSET => -120 * 60);

-- Query a table as it was yesterday
SELECT * FROM customers AT (TIMESTAMP => '2024-01-15 12:00:00'::timestamp);

-- Recreate a dropped table from time travel
CREATE TABLE customers_restored CLONE customers AT (BEFORE => (SELECT time FROM sales_history WHERE name = 'customers'));
```

### Zero-Copy Cloning
- `CLONE` does **not** copy data — it creates a **new metadata pointer** referencing the same micro-partitions.
- This makes cloning:
  - **Instant** (seconds, regardless of table size).
  - **Storage-free** (no additional storage cost at creation time).
- Storage is only charged when the clone or the original **diverges** (i.e., when you write new data to one of them), because Snowflake uses copy-on-write for changed micro-partitions.
- Zero-copy cloning works for **tables, schemas, and databases**, making environment setup (dev/staging/test) fast and cheap.
- Great for: creating dev environments, taking pre-release snapshots, testing DML before a risky change, and rollback scenarios.

### Table Storage Considerations
- Storage cost is based on **compressed** size of data + time travel + fail-safe copies.
- Avoid dropping/creating tables repeatedly (creates churn); prefer `CREATE OR REPLACE` where sensible.
- Use **transient** tables for staging/intermediate data to avoid fail-safe and reduce time-travel storage overhead.

---

## Time Travel

### What is Time Travel
- Time Travel is a feature that lets you **access historical data** — data as it existed at any point within the retention window.
- Enabled by default for all permanent tables (retention configurable from **0 to 90 days** depending on edition).
- Uses include:
  - Recovering accidentally **dropped tables** or deleted rows.
  - **Duplicating/backing up** data from a past point in time.
  - **Analyzing how data changed** over time (audit, debugging).
  - Comparing current data against a past snapshot.

### AT vs BEFORE
- `AT` — query the data **as it existed at** a specific point in time.
- `BEFORE` — query the data **as it existed before** a specific point in time (used to recover data as it was before a destructive statement).
- Example:
```sql
-- Data as of a specific timestamp
SELECT * FROM orders AT (TIMESTAMP => '2024-01-15 12:00:00'::timestamp_tz);

-- Data as of 30 minutes ago
SELECT * FROM orders AT (OFFSET => -1800); -- offset in seconds

-- Data as it existed BEFORE a particular query/statement
SELECT * FROM orders BEFORE (STATEMENT => '0191a1f0-...'); -- query ID

-- Using a query ID to get data as of right before that query ran
SELECT * FROM orders BEFORE (STATEMENT => (SELECT query_id FROM my_session_queries));
```

### Data Retention Periods
- Time travel retention is set per object (table/schema/database):
  - Standard Edition: **0 or 1 day**.
  - Enterprise Edition and above: **0 to 90 days**.
- Set retention at creation or via ALTER:
```sql
CREATE TABLE orders (order_id NUMBER, ...) DATA_RETENTION_TIME_IN_DAYS = 30;

ALTER TABLE orders SET DATA_RETENTION_TIME_IN_DAYS = 90;
```
- **Fail-safe** (7 extra days) applies after time travel for permanent tables only.

### How to Recover Dropped Tables Using UNDROP
- When a table is dropped, it is not immediately destroyed — it moves into time travel and can be recovered with **UNDROP**.
```sql
DROP TABLE customers;

-- Oops — recover it (restores the most recent version with that name)
UNDROP TABLE customers;
```
- UNDROP works for tables, schemas, and databases.
- If the name is in use by a new object, you can rename the new object away and then UNDROP.

### Cloning Past Data
- Combine time travel with zero-copy cloning to restore a historical snapshot as a new table:
```sql
-- Restore orders as of 2 days ago into a fresh table
CREATE TABLE orders_snapshot CLONE orders AT (OFFSET => -2 * 86400);

-- Restore a whole database from a past time
CREATE DATABASE prod_restore CLONE prod_db AT (TIMESTAMP => '2024-01-10 00:00:00'::timestamp);
```
- Because cloning is metadata-only, restoring huge datasets from time travel is fast and cheap.

### Example: Fix a Bad UPDATE
```sql
-- Someone ran: UPDATE employees SET salary = salary * 1000;

-- Option 1: recreate the table as it was before the bad update
CREATE TABLE employees_fixed CLONE employees BEFORE (STATEMENT => '<bad_query_id>');

-- Option 2: rewrite the bad rows from time travel
CREATE TABLE employees_good CLONE employees AT (OFFSET => -3600);
DELETE FROM employees; -- or a more surgical delete
INSERT INTO employees SELECT * FROM employees_good;
DROP TABLE employees_good;
```

---

## Fail-safe

### What is Fail-safe
- **Fail-safe** is an additional, non-configurable **7-day recovery period** that applies **after** the time-travel window ends.
- Its purpose: **disaster recovery** — recovering data that Snowflake itself would otherwise be unable to restore (e.g., catastrophic storage failure).
- Available only for **permanent** tables (not transient, not temporary).

### Key Characteristics
- **Cannot be configured or disabled** — it is always 7 days, always on for permanent objects.
- **Cannot be queried directly** — you cannot run SELECT against fail-safe data, and you cannot clone from it.
- To recover from fail-safe, you must **contact Snowflake Support**, who will restore the data on your behalf (typically through a clone).
- **Additional storage cost** — fail-safe consumes 1 full copy of the table's storage (on top of time-travel copies), roughly doubling storage footprint for changed data.

### Typical Timeline for a Dropped Table
```
Event (DROP TABLE) -> Time Travel (0-90 days, queryable/cloneable) -> Fail-safe (7 days, Support only) -> Data gone forever
```

### Use Cases
- Recovering data after a **disaster** (region failure, accidental mass deletion that exceeded time travel).
- Compliance/audit scenarios where you need an extra safety net.
- **Not** for normal operational recovery — use time travel for that.

### Best Practices
- Do not rely on fail-safe as your backup strategy; it is last-resort disaster recovery.
- Use time travel + zero-copy clones + regular exports for real backups.
- Keep sensitive/unnecessary transient data in transient tables to avoid paying fail-safe storage on data you do not need to protect.

---

## Stages

### What is a Stage
- A **stage** is a location where data files live before being loaded into Snowflake tables.
- Data is first **uploaded to a stage**, then loaded into a table with **COPY INTO**.
- Stages are tied to a database/schema and are used by the loading pipeline.

### Internal vs External Stages
- **Internal stages** — storage managed by Snowflake (inside Snowflake's own cloud storage):
  - **User stage** (per-user, referenced as `@~`).
  - **Table stage** (per-table, referenced as `@%tablename`).
  - **Named internal stage** (`CREATE STAGE`), recommended for controlled pipelines.
- **External stages** — reference cloud object storage you own:
  - AWS S3 (`s3://bucket/path/`).
  - Azure Blob / ADLS (`azure://account.blob.core.windows.net/container/path`).
  - Google Cloud Storage (`gcs://bucket/path`).
  - External stages use **storage integrations** or explicit credentials to authenticate.

```sql
-- Named internal stage
CREATE OR REPLACE STAGE my_stage
  FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"');

-- External stage on S3
CREATE OR REPLACE STAGE s3_stage
  STORAGE_INTEGRATION = my_s3_integration
  URL = 's3://my-bucket/landing/'
  FILE_FORMAT = (TYPE = 'PARQUET');
```

### File Formats
- Snowflake natively loads **semi-structured and structured** files:
  - **CSV** (also TSV, delimited text).
  - **JSON** (loading into `VARIANT` columns).
  - **Parquet** (columnar, very popular for analytics).
  - **Avro**, **ORC**, and others (XML).
- File formats are defined as objects and reused across stages and COPY statements:

```sql
CREATE OR REPLACE FILE FORMAT my_json_fmt
  TYPE = 'JSON'
  STRIP_OUTER_ARRAY = TRUE;

CREATE OR REPLACE FILE FORMAT my_csv_fmt
  TYPE = 'CSV'
  COMPRESSION = 'GZIP'
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

### COPY INTO — Loading Data
- `COPY INTO` is the command that **loads data from a stage into a table**.
- It supports both **initial load** and **incremental load** (tracks already-loaded files via metadata so files are not re-loaded).

```sql
-- Load CSV files
COPY INTO sales_raw
FROM @my_stage
FILE_FORMAT = (FORMAT_NAME = my_csv_fmt);

-- Load with error handling (continue on error, log bad rows)
COPY INTO sales_raw
FROM @my_stage
FILE_FORMAT = (FORMAT_NAME = my_csv_fmt)
ON_ERROR = 'CONTINUE'
PURGE = TRUE; -- delete files from stage after successful load
```

- Loading **JSON** into a VARIANT column:

```sql
COPY INTO events
FROM @my_stage
FILE_FORMAT = (FORMAT_NAME = my_json_fmt);
-- events has a column: payload VARIANT
```

- Transformations during load (ELT-lite, using the `$1` file stream and SELECT):

```sql
COPY INTO sales (order_id, customer_id, order_date, amount)
FROM (
  SELECT $1:order_id,
         $1:customer_id,
         TO_DATE($1:order_date),
         $1:amount::NUMBER(10,2)
  FROM @my_stage
)
FILE_FORMAT = (FORMAT_NAME = my_json_fmt);
```

### PUT Command — Uploading Files
- **PUT** uploads local files to an internal stage (it does **not** work with external stages — those are uploaded via cloud CLIs like `aws s3 cp`).
- PUT is run via SnowSQL (the command-line client), not through the web UI.

```sql
-- Upload a local CSV to a named stage (SnowSQL syntax)
PUT file://C:\data\sales_2024_01.csv @my_stage AUTO_COMPRESS = TRUE;

-- List files in a stage
LIST @my_stage;

-- Remove files from a stage
REMOVE @my_stage PATTERN = '.*\.csv\.gz';
```

- `AUTO_COMPRESS = TRUE` gzip-compresses files before upload, reducing storage and load time.

### Snowpipe (Continuous / Auto-Ingestion)
- **Snowpipe** is Snowflake's **continuous, serverless ingestion** service (covered in depth in the next section).
- Automatically loads files **as soon as they appear** in a stage, with no scheduled COPY job.

### Manual vs Continuous Loading Summary
| Approach | When to use |
|----------|------------|
| `COPY INTO` (scheduled) | Batch loads (daily/hourly), large historical backfills |
| `Snowpipe` | Streaming/near-real-time, frequent small files |
| `PUT` + COPY | One-off local uploads, small data |

---

## Snowpipe

### What is Snowpipe
- Snowpipe is a **serverless, event-driven data ingestion** service.
- It watches a stage (internal or external) and **automatically loads new files into a table** as soon as they appear.
- No warehouse is needed for Snowpipe loads — it runs on Snowflake-managed compute, and you pay per **ingested file** (by data volume), not per credit of a warehouse.

### How It Works
1. Files land in a stage (e.g., an S3 bucket).
2. A **notification** is fired (S3 event notification / SNS, Azure Event Grid / Blob storage events, or GCS Pub/Sub).
3. A **pipe** (defined in Snowflake) receives the notification and triggers COPY of the new files.
4. Files are loaded into the target table within **~1-2 minutes** (often faster).

### Setting Up Snowpipe
```sql
-- 1. Define a pipe referencing a stage and file format
CREATE OR REPLACE PIPE my_pipe
  AUTO_INGEST = TRUE
  AS
    COPY INTO my_table
    FROM @my_stage
    FILE_FORMAT = (FORMAT_NAME = my_csv_fmt);

-- 2. Get the SQS queue / notification endpoint for the external stage
SHOW PIPES;
-- Take the "notification_channel" / ARN from the output and
-- configure it as an event notification on your S3 bucket (or Azure/GCP equivalent).

-- Manage pipes
ALTER PIPE my_pipe SET PIPE_EXECUTION_PAUSED = TRUE;  -- pause
ALTER PIPE my_pipe SET PIPE_EXECUTION_PAUSED = FALSE; -- resume
DROP PIPE my_pipe;
```

- For internal stages, you can also load manually with `ALTER PIPE ... REFRESH` or the Snowpipe REST API.

### Event-Driven Ingestion
- Snowpipe is **push-based**: cloud storage events (S3 `s3:ObjectCreated:*`, Azure Blob created, GCS object finalize) trigger loading.
- Alternative, also possible: **polling-based** via the Snowpipe REST API (`INSERT FILES` endpoint).
- Because it is event-driven, data arrives continuously with minimal latency — no scheduler needed.

### Difference Between COPY INTO and Snowpipe
| Aspect | COPY INTO | Snowpipe |
|--------|-----------|----------|
| Execution | Needs a running warehouse | Serverless (no warehouse) |
| Trigger | Manual or scheduled | Event-driven, automatic |
| Latency | Batch (hourly/daily) | Near real-time (minutes) |
| Cost | Warehouse credits | Pay per file/data volume |
| Best for | Big batches, backfills | Continuous, frequent small files |

### Latency and Cost
- **Latency**: typically 1-2 minutes from file arrival to queryable data (near real-time, not true streaming like Kafka).
- **Cost**:
  - Charged per **1 MB of loaded data** (per file, with a small rounding up).
  - Cheaper than running a warehouse 24/7 for the same volume of small loads.
  - No minimum warehouse cost; ideal for frequent small ingests.
- **Note**: Snowpipe loads use a different cost line than warehouse credits and appear under **Data loading / Snowpipe** in usage views.

### Snowpipe vs Streaming
- Snowpipe is **batch-ish / micro-batch** (file based).
- For true row-level streaming, Snowflake also offers **Snowpipe Streaming** (Kafka-style) and the Kafka connector.

---

## Security

### Snowflake Security Model
- Security is built into the **cloud services layer**.
- Key pillars:
  - **Identity & access control** (authentication + RBAC).
  - **Encryption** at rest and in transit.
  - **Network security** (network policies, private connectivity).
  - **Data protection** (masking, row-level security, audit logs).
- Snowflake holds certifications such as **SOC 1/SOC 2**, **ISO 27001**, **HIPAA**, **PCI DSS**, and **GDPR** compliance support.

### Access Control (Roles, Grants)
- Snowflake uses **RBAC (Role-Based Access Control)**:
  - **Privileges are granted to roles**, and **roles are granted to users**.
  - Users never hold privileges directly — they inherit them through roles.
- Key statements:
```sql
-- Create a role
CREATE ROLE analyst;

-- Grant privileges to the role
GRANT SELECT ON DATABASE analytics TO ROLE analyst;
GRANT USAGE ON WAREHOUSE reporting_wh TO ROLE analyst;

-- Grant the role to a user
GRANT ROLE analyst TO USER jane_doe;

-- Revoke
REVOKE SELECT ON DATABASE analytics FROM ROLE analyst;
```

### RBAC in Snowflake — Key Roles
- **ACCOUNTADMIN**: top-level role; can manage everything including other admins, billing, and account settings.
- **SYSADMIN**: creates/owns warehouses, databases, and other objects used by the organization.
- **SECURITYADMIN**: manages users and grants roles; a child of USERADMIN.
- **USERADMIN**: manages users and roles.
- **PUBLIC**: the role every user gets by default; used for default object access (e.g., granting USAGE on a database to PUBLIC).
- Custom roles can be created for teams (analyst, engineer, auditor) and nested (role hierarchy), so privileges propagate.
- Hierarchy note: `ACCOUNTADMIN > SECURITYADMIN/USERADMIN > SYSADMIN > PUBLIC` in terms of management powers.

### Privilege Inheritance
- Roles can be granted to other roles:
```sql
GRANT ROLE analyst TO ROLE engineering; -- engineering inherits analyst's privileges
```
- Ownership: the owner of an object can grant privileges on it (with the OWNERSHIP privilege).

### Encryption
- **Data at rest**: fully encrypted automatically using **AES-256** (hardware-enforced via KMS in each cloud).
  - Snowflake manages encryption keys by default; you can bring your own keys (**customer-managed keys** / tri-secret secure).
- **Data in transit**: encrypted with **TLS 1.2+** (HTTPS) for all communication between clients, warehouse nodes, and storage.
- File-level encryption uses envelope encryption (each file encrypted with a unique key, keys encrypted by master keys).

### Authentication Methods
- **Username + password** (standard).
- **Multi-factor authentication (MFA)** — recommended, enforced via DUO.
- **Key-pair authentication** — RSA key pairs for programmatic/service accounts (SnowSQL, connectors):
```sql
-- Generate a private key (OpenSSL), then register its public key:
ALTER USER svc_account SET RSA_PUBLIC_KEY = 'MIIBIjANBgkqh...';
```
- **SSO / SAML 2.0** — federate with Okta, Azure AD, ADFS, Google Workspace, etc.:
  - Users sign in through the IdP; Snowflake trusts the IdP assertion.
- **OAuth** for applications and clients.

### Network Policies
- **Network policies** restrict which IP addresses / ranges can connect to the account.
```sql
CREATE NETWORK POLICY corporate_net
  ALLOWED_IP_LIST = ('10.0.0.0/8', '203.0.113.5')
  BLOCKED_IP_LIST = ('192.0.2.0/24');

ALTER ACCOUNT SET NETWORK_POLICY = corporate_net;
```
- Combine with **PrivateLink / Private Connectivity** for completely private network access (no public internet).

### Other Security Features
- **Dynamic Data Masking** — mask sensitive columns at query time based on the role:
```sql
CREATE OR REPLACE MASKING POLICY email_mask AS (val STRING) RETURNS STRING ->
  CASE WHEN CURRENT_ROLE() IN ('ANALYST') THEN val ELSE '***masked***' END;
```
- **Row-level security** using row access policies.
- **Auditing**: `ACCOUNT_USAGE.QUERY_HISTORY`, `LOGIN_HISTORY`, `ACCESS_HISTORY` for governance and security monitoring.

---

## Snowflake Data Sharing

### Secure Data Sharing (Without Copying Data)
- Snowflake lets you **share data between accounts without copying or transferring the data**.
- The consumer reads the provider's data **live**, from the provider's own storage, via Snowflake's services layer.
- No ETL, no file drops, no FTP, no S3 buckets, no duplication.
- Because it is metadata/pointer-based, sharing is **instantaneous** and the consumer always sees the **latest data**.

### How Sharing Works
- The **provider** creates a **share** (a container of databases/schemas/tables/views) and grants it to a consumer account.
- The **consumer** sees the shared objects read-only (they can query, join, and copy from them, but cannot modify).
- Shares require both accounts to exist in Snowflake (different clouds/regions are possible via replication in some setups).

```sql
-- Provider side
CREATE SHARE my_sales_share;
GRANT USAGE ON DATABASE sales_db TO SHARE my_sales_share;
GRANT USAGE ON SCHEMA sales_db.public TO SHARE my_sales_share;
GRANT SELECT ON TABLE sales_db.public.daily_sales TO SHARE my_sales_share;

-- Add the consumer account (e.g., ABC12345) with the share
ALTER SHARE my_sales_share ADD ACCOUNTS = ABC12345;

-- Consumer side: create a database from the share
CREATE DATABASE shared_sales FROM SHARE ABC12345.my_sales_share;
-- Now query it like any local database:
SELECT * FROM shared_sales.public.daily_sales;
```

### Provider vs Consumer
- **Provider**: the account that owns and shares data.
  - Controls which objects are shared and with whom.
  - Can revoke or update shares at any time.
  - Can share **views** (including secure views) to expose only a subset or transformed data.
- **Consumer**: the account receiving the share.
  - Read-only access to the shared objects.
  - Can create local views/materialized views on top of shared data.
  - Can copy data out of the share into local tables (which then costs storage).

### Data Marketplace
- The **Snowflake Marketplace** is a public data marketplace where providers publish shares that any Snowflake customer can discover and consume.
- Includes third-party datasets (demographics, weather, financial data, geospatial, etc.).
- Some listings are free, some paid — but consumption still does **not** copy data to your account.

### Why Data Sharing Is Instant
- Sharing is a **metadata-only** operation:
  - The share simply references the provider's micro-partitions.
  - No physical data movement, no ingestion, no load jobs.
  - The consumer's warehouse reads directly from the provider's storage (with proper credentials negotiated by the services layer).
- Result: a customer can start querying a dataset within **seconds to minutes** of a share being granted, regardless of dataset size.

### Use Cases
- Sharing clean, governed data across business units within the same account/organization.
- Sharing with **external partners/vendors** without exposing underlying infrastructure.
- Publishing datasets via the marketplace.
- Building a **data mesh** where each team owns and shares its domain data.

---

## Tasks and Scheduling

### What are Snowflake Tasks
- A **task** is a scheduled SQL statement (or stored procedure call) that runs automatically on a schedule or in response to a trigger.
- Tasks are executed on a warehouse (or serverless).
- Example:
```sql
CREATE TASK load_sales_daily
  WAREHOUSE = analytics_wh
  SCHEDULE = '60 MINUTE'
AS
  COPY INTO sales_raw FROM @my_stage FILE_FORMAT = (FORMAT_NAME = my_csv_fmt);
```

### Scheduling with CRON
- Tasks can be scheduled with either an interval or a **cron expression**:
```sql
CREATE TASK hourly_rollup
  WAREHOUSE = analytics_wh
  SCHEDULE = 'USING CRON 0 * * * * America/New_York'
AS
  INSERT INTO daily_rollup
  SELECT order_date, region, SUM(amount)
  FROM sales_raw
  GROUP BY order_date, region;
```
- Cron format: `minute hour day-of-month month day-of-week`, with optional timezone.
- Common examples:
  - Every 5 minutes: `USING CRON */5 * * * *`
  - Every day at 2 AM UTC: `USING CRON 0 2 * * * UTC`
  - Every Monday 9 AM: `USING CRON 0 9 * * MON`

### Task DAGs (Dependent Tasks)
- Tasks can depend on other tasks, forming a **task DAG (Directed Acyclic Graph)**.
- A child task only runs **after its parent(s) complete successfully**.
- DAGs enable multi-step pipelines (extract → transform → load → aggregate) orchestrated natively in Snowflake.

```sql
CREATE TASK load_raw      WAREHOUSE = analytics_wh SCHEDULE = '5 MINUTE' AS COPY INTO raw_tbl FROM @stage;
CREATE TASK transform     WAREHOUSE = analytics_wh AFTER load_raw AS INSERT INTO clean_tbl SELECT ... FROM raw_tbl;
CREATE TASK aggregate     WAREHOUSE = analytics_wh AFTER transform AS INSERT INTO agg_tbl SELECT ... FROM clean_tbl;

ALTER TASK aggregate ADD AFTER transform;
-- Start the whole DAG (only the root task needs to be scheduled)
ALTER TASK load_raw RESUME;
```

### Task Management Commands
```sql
ALTER TASK task_name RESUME;   -- start scheduling
ALTER TASK task_name SUSPEND;  -- stop scheduling
SHOW TASKS;
DROP TASK task_name;
CREATE OR REPLACE TASK ...;
```

### Serverless Tasks
- If you omit `WAREHOUSE`, tasks can run **serverless** (Snowflake provisions compute on demand). More convenient, but you lose control over warehouse sizing and it can be costlier for heavy jobs.

### Difference From dbt Scheduling
- **Snowflake Tasks**:
  - Native database scheduler inside Snowflake.
  - Great for simple SQL pipelines, COPY jobs, and lightweight transforms.
  - Limited orchestration logic (no complex branching, retries, or external dependencies out of the box).
- **dbt**:
  - A **transformation framework** (SQL models with refs, tests, docs, lineage).
  - Runs models via `dbt run`, scheduled by external orchestrators (Airflow, dbt Cloud, cron).
  - Not a scheduler itself; it is a build tool. You schedule dbt runs on top of Snowflake.
- Common pattern: **dbt + Airflow/dbt Cloud** for transformation, while Snowflake tasks handle database-level ingestion and housekeeping.
- Tasks can be used to trigger dbt runs too (e.g., task calls a stored procedure that invokes dbt via Snowflake's external functions or just kicks off a step).

### When to Use Tasks
- Scheduled COPY loads (hourly/daily).
- Database housekeeping (ARCHIVE, DELETE old partitions, run maintenance).
- Simple dependencies without an external orchestrator.
- Triggering pipes / refreshing materialized views fallback.

---

## Views and Materialized Views

### Regular Views
- A view is a **saved query** — a virtual table with no physical storage.
- Data is always current: querying the view executes the underlying query against base tables.
- Views are logical and do not improve performance (no pre-computation, no index).
- Useful for: security (limit columns/rows), simplifying complex joins, consistent definitions.

```sql
CREATE OR REPLACE VIEW v_sales_summary AS
SELECT region, order_date, SUM(amount) AS total_sales
FROM sales
GROUP BY region, order_date;

SELECT * FROM v_sales_summary WHERE region = 'EMEA';
```

- **Secure views**: hide the definition (the underlying query text) from non-owners, and prevent conversion to a regular view. Used when sharing data.

### Materialized Views in Snowflake
- A materialized view **stores the result of the query** as actual data (pre-computed).
- Snowflake automatically maintains it: as base tables change, the materialized view is **incrementally refreshed in the background** (no manual REFRESH needed).
- Refreshes use **serverless compute** automatically (no warehouse needed to maintain it).
- Querying a materialized view can be **much faster** because Snowflake scans only the (small, pre-aggregated) materialized data instead of the huge base table.

```sql
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT order_date, region, SUM(amount) AS total_sales
FROM sales
GROUP BY order_date, region;
```

### Important Limitations of Materialized Views
- Supported only for a subset of queries (aggregates with GROUP BY, some joins, and certain functions).
- Do **not** support: non-deterministic functions (CURRENT_TIMESTAMP), HAVING in some cases, subqueries in certain positions, OUTER joins in some versions, etc.
- There is a **maximum limit** on the number of materialized views per table in some editions.
- They consume **storage** (the materialized result set) and background **compute** for refreshes.

### When to Use Materialized Views
- Base table is **large** and the same aggregation is queried repeatedly.
- Queries need **fast reads** on a small pre-aggregated result.
- Reporting/dashboards over slowly changing but large fact tables.
- Base table changes are **incremental** (appends), which makes auto-refresh cheap and efficient.

### When NOT to Use Them
- If base tables are updated heavily (constant rewrites), refresh cost can exceed the benefit.
- If you need full SQL flexibility — use a regular view or a transformed table.
- For one-off or ad-hoc queries — just query the base table.

### Views vs Materialized Views Summary
| Aspect | View | Materialized View |
|--------|------|-------------------|
| Storage | None | Stores pre-computed result |
| Performance | Same as base query | Fast (no base scan) |
| Freshness | Always current | Auto-maintained, near current |
| Maintenance cost | None | Storage + background refresh credits |
| Use case | Logic/security reuse | Repeated heavy aggregations |

---

## Performance

### Query Profiling (Query Profile Tab)
- Every query in Snowsight has a **Query Profile** — a visual, per-operator breakdown of how the query executed.
- Key things to inspect:
  - **Operators and timing**: which steps took longest (scan, join, aggregation, sort).
  - **Rows/bytes scanned** vs produced: watch for full-table scans that should have pruned.
  - **Pruned partitions / scanned partitions** ratio.
  - **Table scan details**: bytes scanned vs bytes eligible for pruning (if `scanned > eligible`, clustering is poor).
  - **Spilled bytes / local vs remote I/O**: heavy spilling to disk or remote storage indicates insufficient warehouse memory.
- Use the profile to find bottlenecks: joins without proper keys, per-row processing, filters applied too late, large sorts.

### Result Cache
- Snowflake caches the **results** of queries in the services layer (not per-warehouse) for 24 hours by default.
- If the **same query** (byte-for-byte identical SQL, same data) is re-run within the cache window and the underlying data is unchanged, results are returned from cache almost instantly — **without consuming warehouse compute**.
- Conditions for cache hit:
  - Exact same query text.
  - No change to the underlying tables' data (Snowflake tracks this).
  - Same role privileges.
- Best practice: parameterize/reuse exact SQL for repeated reports; avoid `SELECT *` plus tiny whitespace changes that break cache hits.
- Cache is shared across warehouses (server-side), so different users can benefit from each other's cached results.

### Warehouse Scaling for Performance
- If a query is I/O or CPU bound and reads a huge amount of data, **scale up** the warehouse (larger size = more parallelism + more local cache).
- If you have **many concurrent queries**, scale out (multi-cluster) to avoid queuing.
- Watch `QUERY_HISTORY` for **queued time** — if queries wait in the queue, the warehouse is too small or needs multi-cluster.
- Resize at runtime:
```sql
ALTER WAREHOUSE analytics_wh SET WAREHOUSE_SIZE = 'X-LARGE';
ALTER WAREHOUSE analytics_wh RESUME;
```

### Pruning and Clustering Keys to Speed Queries
- The single biggest performance lever is **pruning**.
- Ensure WHERE predicates filter on **clustered columns** (or columns with good natural clustering) so micro-partitions are skipped.
- Add clustering keys to large tables on the columns used most in filters.
- Check the query profile's prune ratio: if scans are near 100% of table, improve clustering.

### Using EXPLAIN
- `EXPLAIN` shows the **logical and physical execution plan** without running the query — useful for understanding joins, scans, pruning, and ordering.

```sql
EXPLAIN USING TEXT
SELECT region, SUM(amount)
FROM sales
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY region;

EXPLAIN USING TABULAR
SELECT * FROM sales s JOIN customers c ON s.customer_id = c.customer_id;
```

- What to look for in plans:
  - **TableScan**: how many partitions scanned; check filter pushdown.
  - **JoinStrategy**: Broadcast vs Hash vs Nested Loop; ensure joins are on appropriate keys.
  - **Filter placement**: filters should be pushed down as early as possible.
  - **Aggregation/Window** operations: sorting/partitioning steps.

### General Performance Tips
- Avoid `SELECT *` in production — read only needed columns (columnar benefit).
- Push filters and aggregations as early as possible.
- Use **micro-batch / incremental** loads rather than frequent full reloads.
- Partition large loads by time (e.g., daily partitions) so pruning is natural.
- Use **streams/tasks** for incremental transforms instead of recomputing full tables.
- For joins, ensure the smaller table is on the correct side or add appropriate keys to let the optimizer pick the best strategy.
- Monitor `WAREHOUSE_LOAD_HISTORY` to see if warehouses are oversized/underutilized.

---

## Cost Management

### Credit Consumption
- **Credits** are Snowflake's billing unit for compute.
- Consumed by:
  - **Virtual warehouses** (running time, including idle time until auto-suspend).
  - **Serverless features** (Snowpipe, materialized view refresh, automatic clustering, serverless tasks).
  - **Cloud services** (but only if cloud services usage exceeds 10% of a warehouse's credits; the first 10% is free).
- Storage is billed separately, per **TB-month** of compressed data (including time travel and fail-safe).

### Why Separation Matters for Cost
- Because compute and storage are separate, you can:
  - **Suspend** warehouses when not in use → zero compute cost while data persists.
  - **Share storage** across many small workloads without paying for idle capacity.
  - **Pay only for the compute you use** per query — no minimum infrastructure.
- This is fundamentally different from old on-prem warehouses or single-monolith cloud VMs where idle hardware still costs money.

### Warehouse Auto-Suspend to Save Cost
- Always configure `AUTO_SUSPEND` on warehouses (default 10 min is often too long for idle dev/test).
```sql
CREATE WAREHOUSE dev_wh
  WAREHOUSE_SIZE = 'X-SMALL'
  AUTO_SUSPEND = 60   -- suspend after 1 minute idle
  AUTO_RESUME = TRUE
  INITIALLY_SUSPENDED = TRUE;
```
- Suspend warehouses manually during off hours:
```sql
ALTER WAREHOUSE analytics_wh SUSPEND;
```
- Right-size warehouses: prefer X-SMALL/SMALL for light workloads; scale up only for heavy queries.

### Monitoring Usage with ACCOUNT_USAGE Views
- The `SNOWFLAKE.ACCOUNT_USAGE` schema holds historical usage/observability data:
  - `WAREHOUSE_METERING_HISTORY` — credits per warehouse per hour.
  - `QUERY_HISTORY` — every query, duration, scanned bytes, warehouse used.
  - `QUERY_ACCELERATION_HISTORY` — credits for query acceleration.
  - `AUTOMATIC_CLUSTERING_HISTORY`, `MATERIALIZED_VIEW_REFRESH_HISTORY`, `PIPE_USAGE_HISTORY` — serverless costs.
  - `STORAGE_USAGE` — table-level storage bytes (including time travel/fail-safe).

```sql
-- Credit burn per warehouse over the last 7 days
SELECT warehouse_name,
       SUM(credits_used) AS credits_used,
       SUM(credits_used_compute) AS compute_credits
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY 1 ORDER BY 2 DESC;

-- Longest-running / most expensive queries
SELECT query_id, query_text, warehouse_name, execution_time
FROM snowflake.account_usage.query_history
WHERE execution_time > 60000 -- ms
ORDER BY execution_time DESC LIMIT 20;

-- Storage breakdown by table
SELECT table_name, bytes / (1024*1024*1024) AS size_gb
FROM snowflake.account_usage.table_storage_metrics
ORDER BY size_gb DESC;
```

### Cost Optimization Checklist
- Set **auto-suspend** aggressively (60-300s) on all warehouses.
- **Right-size** warehouses; start small and scale only on evidence.
- Use **multi-cluster** only for genuine concurrency spikes; set `scaling_policy = ECONOMY`.
- Use **Snowpipe** for frequent small loads (serverless, per-byte pricing).
- Use **transient tables** for staging data (no fail-safe, less time-travel storage).
- Avoid unnecessary **clones** of large tables (though zero-copy clones are cheap, they still accrue time-travel storage on writes).
- **Query efficiently**: prune, cache, avoid SELECT *, use incremental transforms.
- Monitor via `ACCOUNT_USAGE` dashboards and set resource monitors/budgets.

### Resource Monitors (Budgets)
```sql
-- Cap monthly credits
CREATE RESOURCE MONITOR monthly_budget
  WITH CREDIT_QUOTA = 5000
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY;

ALTER WAREHOUSE analytics_wh SET RESOURCE_MONITOR = monthly_budget;
-- Notifications via email at thresholds; optionally SUSPEND_TASK on exceed.
```

---

## Common Mistakes

### Over-Sizing Warehouses
- Using X-LARGE for a query that would finish fine on MEDIUM wastes credits (each doubling = 2x cost).
- **Fix**: start small, scale up only when query profile shows the query is resource-bound; use `ALTER WAREHOUSE ... SET WAREHOUSE_SIZE` dynamically if needed.

### Not Using Clustering Keys for Large Tables
- Huge tables with filters on poorly-clustered columns cause full-table scans and slow queries.
- **Fix**: add clustering keys on frequently-filtered columns of large tables; monitor pruning in query profiles.

### Ignoring Result Caching
- Re-running slightly-different SQL or leaving off deterministic parameters defeats the 24h result cache.
- **Fix**: reuse exact query text for repeated reports; leverage cache hits to reduce compute cost to zero for common dashboards.

### Loading Data Inefficiently
- Doing many tiny single-file COPY loads, or loading during peak BI hours on the same warehouse, causes contention and waste.
- **Fix**: batch files, load incrementally with Snowpipe for continuous flows, and use a dedicated loading warehouse.

### Not Setting Auto-Suspend
- Warehouses left running 24/7 burn credits even when idle (a MEDIUM warehouse idle all month is very expensive).
- **Fix**: always set `AUTO_SUSPEND` (60-300s) and `AUTO_RESUME = TRUE`; suspend dev/test warehouses overnight.

### Unnecessary Cloning of Large Tables
- While zero-copy clones are cheap at creation, cloning big tables and then writing to both copies doubles storage via time travel, and frequent clones create metadata churn.
- **Fix**: clone only when needed (dev environments, pre-deploy snapshots); drop clones promptly; prefer transient tables for scratch work.

### Other Common Pitfalls
- **SELECT * on wide tables** — reads far more data than needed (hurts cost and speed).
- **Ignoring query profiles** — shipping slow queries instead of diagnosing prune ratios and joins.
- **Loading JSON into string columns** — use VARIANT + FLATTEN for semi-structured data.
- **Not using ON_ERROR in COPY** — loads fail entirely on one bad file; use `ON_ERROR = CONTINUE` + `REJECTED_RECORD` tables.
- **Running transformations on the load warehouse** — mix heavy analytic queries with ingestion, causing slowdowns.
- **Forgetting resource monitors** — no budget guardrails, so surprise credit bills.
- **Not using time travel** — dropping tables without knowing `UNDROP` and retention windows.
- **Treating Snowpipe like a streaming service** — it is near-real-time, not sub-second; use proper streaming for true real-time needs.
- **Sharing data via CSV/S3 instead of secure shares** — reinventing the wheel with ETL when secure sharing is instant and free of copies.

---

## Interview Questions

### Architecture
1. **What is Snowflake and how is it different from a traditional data warehouse?**
   - Snowflake is a fully managed, multi-cloud SaaS data warehouse. Unlike traditional warehouses (and cloud ones built on existing engines), Snowflake is built from the ground up as three decoupled layers: storage, compute, and cloud services, so it scales elastically and requires no manual administration.

2. **Explain the three-layer architecture of Snowflake.**
   - Storage layer: columnar micro-partitions in cloud object storage. Compute layer: virtual warehouses (MPP clusters) that execute queries. Cloud services layer: stateless services handling metadata, security, transactions, query parsing/optimization, and concurrency coordination.

3. **Why is the separation of compute and storage important?**
   - It allows independent scaling of each resource, lets you pause compute to zero while data persists (saving cost), supports multiple warehouses sharing the same storage without contention, and enables metadata-only features like zero-copy cloning and secure sharing.

4. **How does Snowflake handle concurrency?**
   - Via multiple virtual warehouses (each isolated) and multi-cluster warehouses. Additionally, cloud services coordinate locking and multi-version concurrency (ACID transactions) so many workloads run simultaneously without interfering.

### Virtual Warehouses
5. **What is a virtual warehouse and how do you size it?**
   - It is an independent cluster of compute nodes that executes queries and DML. Sizes go from X-Small to 6X-Large; each doubling roughly doubles compute power (CPU/memory/cache/parallelism). Choose size based on data volume and query complexity.

6. **Explain the difference between scaling up and scaling out a warehouse.**
   - Scaling up changes the T-shirt size (more per-node power) for heavy individual queries; scaling out adds more clusters (multi-cluster) to handle many concurrent users. Scaling out is best for BI spikes; scaling up for single large analytics jobs.

7. **What is auto-suspend and auto-resume?**
   - Auto-suspend stops the warehouse after a configured idle period; auto-resume restarts it on the next query. They prevent paying for idle compute while keeping warehouses convenient for ad-hoc use.

### Micro-partitions
8. **What are micro-partitions and how do they speed up queries?**
   - Micro-partitions are 50-500MB contiguous columnar storage blocks holding metadata (min/max, distinct counts) per column. The optimizer uses this metadata to prune (skip) irrelevant partitions, so only a small fraction of a table is ever scanned.

9. **What is pruning and how does clustering affect it?**
   - Pruning is skipping micro-partitions whose min/max ranges don't match query predicates. Clustering is how well rows with similar values group together; good clustering (or explicit clustering keys) makes pruning far more effective and queries dramatically faster.

### Time Travel & Cloning
10. **What is time travel and how do AT and BEFORE differ?**
    - Time travel lets you query or recover historical data within a retention window (0-90 days). `AT` views data as of a specific time; `BEFORE` views data as it was before a specific statement/time (useful for undoing a bad DML).

11. **How do you recover a dropped table?**
    - Use `UNDROP TABLE table_name;` which restores the most recent dropped object from time travel. If the name is taken, rename the new object first, then UNDROP.

12. **What is zero-copy cloning?**
    - `CREATE TABLE x CLONE y` creates a metadata-only pointer to the same micro-partitions — instant and storage-free until data diverges (copy-on-write). Works on tables, schemas, and databases, ideal for dev environments and snapshots.

13. **What is fail-safe and how is it different from time travel?**
    - Fail-safe is a mandatory 7-day disaster-recovery period after time travel ends, for permanent tables only. It cannot be configured or queried directly — you must contact Snowflake Support. It is not a replacement for backups.

### Stages, Snowpipe & Loading
14. **What is a stage and what's the difference between internal and external stages?**
    - A stage is a location holding files before loading. Internal stages are managed by Snowflake (user `@~`, table `@%table`, or named); external stages point to your own S3/Azure/GCS storage and use storage integrations/credentials.

15. **How does COPY INTO work and how is it different from Snowpipe?**
    - `COPY INTO` loads files from a stage into a table using a running warehouse (batch, manual/scheduled). Snowpipe is serverless, event-driven loading that loads files automatically as they arrive (near real-time), billing per file/volume rather than warehouse credits.

16. **How do you load semi-structured data like JSON?**
    - Load into a VARIANT column via COPY INTO with a JSON file format, then use `FLATTEN` and `:` path notation to extract fields. Optionally transform during COPY using a SELECT on the `$1` file stream.

### Security & RBAC
17. **Explain Snowflake's RBAC model.**
    - Privileges are granted to roles and roles are granted to users (and roles to roles). Key system roles: ACCOUNTADMIN (top), SYSADMIN (object owner), SECURITYADMIN/USERADMIN (users/roles), PUBLIC (default). Users inherit privileges only through roles.

18. **How does Snowflake encrypt data?**
    - AES-256 encryption at rest (managed or customer-managed keys via KMS) and TLS 1.2+ in transit. You can also enforce MFA, key-pair auth, SSO/SAML federation, OAuth, and network policies.

### Tasks & Scheduling
19. **What are Snowflake tasks and task DAGs?**
    - Tasks are scheduled SQL statements (`SCHEDULE = '60 MINUTE'` or cron). DAGs chain tasks with `AFTER`, so children run after parents succeed, enabling multi-step pipelines natively in the database.

20. **How do Snowflake tasks differ from dbt scheduling?**
    - Tasks are an in-database scheduler for SQL; dbt is a transformation framework (models/tests/docs) that is orchestrated externally (Airflow, dbt Cloud). Common pattern: dbt for transforms, tasks for ingestion/housekeeping.

### Data Sharing
21. **What is secure data sharing and why is it instant?**
    - Providers grant a share (objects) to consumer accounts; consumers query provider data live without copying. It is metadata-only — the share points at the provider's micro-partitions, so data never moves, making sharing instantaneous and always current.

22. **What's the difference between a provider and consumer in a share, and what is the Marketplace?**
    - The provider owns and controls the shared objects; the consumer gets read-only access. The Snowflake Marketplace lets providers publish shares publicly for any customer to consume — again without data copy.

### Performance & Caching
23. **What is the result cache and when does it apply?**
    - The result cache stores exact query results server-side for up to 24 hours (until underlying data changes). Identical queries (same SQL, same data, same privileges) return instantly from cache with zero warehouse compute.

24. **When should you use a materialized view?**
    - When the same heavy aggregation over a large base table is queried repeatedly and base data changes incrementally. Snowflake auto-refreshes them in the background (serverless) so queries are fast; but they have SQL limitations and storage/refresh costs.

25. **How would you investigate a slow query?**
    - Check the Query Profile for prune ratio, bytes scanned vs eligible, spill/remote I/O, join strategy and filter placement; use `EXPLAIN` to review the plan; verify clustering keys on large filtered columns; and confirm the warehouse isn't the bottleneck (queued time, scaling).

### Cost
26. **What consumes credits in Snowflake and how do you control cost?**
    - Warehouses (running/idle until auto-suspend), serverless features (Snowpipe, MVs, auto-clustering), and cloud services beyond 10%. Control via auto-suspend, right-sizing, multi-cluster only when needed, transient tables, Snowpipe for small loads, and resource monitors.

27. **How do you monitor cost and usage?**
    - Query `SNOWFLAKE.ACCOUNT_USAGE` views: `WAREHOUSE_METERING_HISTORY`, `QUERY_HISTORY`, `STORAGE_USAGE`, `PIPE_USAGE_HISTORY`, and `AUTOMATIC_CLUSTERING_HISTORY`; set up resource monitors with credit quotas and notifications.

28. **Why does the separation of compute and storage reduce cost?**
    - You only pay for compute while it runs, and you can suspend it entirely while your data remains stored. No idle hardware, no re-provisioning, and shared storage across workloads means no over-provisioning for peak usage.

---

*End of Snowflake interview notes. Good luck!*
