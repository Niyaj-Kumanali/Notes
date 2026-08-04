# Databricks

## Overview

- **Definition** — Databricks is a unified data analytics platform built on top of Apache Spark that provides a **lakehouse** architecture combining data lake and data warehouse capabilities
- **Why It Exists** — Traditional architectures force a choice: cheap, flexible data lakes (with poor reliability and no ACID) OR expensive, governed data warehouses (with rigid schemas); Databricks bridges the gap with a single platform
- **Founded By** — The original creators of Apache Spark (Matei Zaharia and the Berkeley AMPLab team), founded in 2013
- **Positioning** — "The lakehouse company" — enables data engineering, data science, machine learning, and BI on a single platform
- **Deployment** — Runs as a managed service on all three major clouds: **AWS, Azure, and GCP** (plus Databricks-to-Databricks, and the Databricks Foundation Models)
- **Core Pieces** — Workspaces, Clusters, Notebooks, Jobs, Delta Lake, Unity Catalog, Databricks SQL, MLflow, and Delta Live Tables

## What is Databricks

### Databricks as a Unified Data Analytics Platform

- One platform covers the full data lifecycle:
  - **Ingestion** — Auto Loader, Spark Streaming, cloud-native connectors
  - **Storage** — Delta Lake sits on cloud object storage (S3, ADLS, GCS)
  - **Transformation** — Spark SQL, DataFrames, Delta Live Tables
  - **BI / SQL** — Databricks SQL warehouses querying the lakehouse directly
  - **Machine Learning** — MLflow, ML endpoints, feature stores
- Unified means data teams share one source of truth instead of duplicating data across separate ETL, warehouse, and ML pipelines
- Compute and storage are decoupled, so you scale each independently and only pay for compute while it runs

### Founded by Creators of Apache Spark

- Spark's creators launched Databricks in 2013 to commercialize Spark and build a managed platform around it
- Because Databricks wrote Spark, its runtime is tuned and optimized deeply for Spark workloads
- The Databricks Runtime (DBR) bundles Spark with performance enhancements, optimizations, and cloud integrations not found in open-source Spark
- Deep Spark lineage means Databricks can commit performance fixes back to open-source Spark

### Runs on AWS, Azure, GCP

- Databricks runs on top of the customer's cloud account, not Databricks' own servers
- The data plane (clusters, storage) lives in your cloud VPC/subscription, giving you data control
- Cloud-specific details handled for you:
  - AWS: IAM roles, S3 storage, VPC
  - Azure: managed identities, ADLS Gen2, Azure Private Link
  - GCP: GCS, service accounts, VPC
- Cross-cloud portability means the same notebooks and pipelines run on any cloud

### Delta Lake: The Storage Layer

- Delta Lake is the open-source transactional storage layer that sits on top of Parquet
- It adds a **transaction log** (`_delta_log` directory of JSON checkpoints) to plain Parquet files
- This gives you ACID transactions, time travel, schema enforcement, and scalable metadata
- Delta is the foundation of the lakehouse — without it, the lake would just be "a bunch of Parquet files"

### Lakehouse Architecture

- A **lakehouse** is a data architecture that stores data in open formats (Parquet/Delta) on cheap object storage but offers warehouse features (ACID, governance, SQL) on top
- The lakehouse removes the need to copy data into a separate proprietary warehouse
- One copy of data serves both BI and machine learning workloads
- Delta Lake + Spark (or Photon) is the mechanism that delivers warehouse-grade performance on lake storage

## Databricks Architecture

### Control Plane vs Data Plane

- **Control Plane** — the Databricks-owned layer that manages the platform
  - Hosts the web application, notebook UI, job scheduler, cluster manager, and metadata store (Hive metastore / Unity Catalog metadata)
  - Runs in Databricks' cloud account in the region you choose
  - Does not touch your raw data — it only holds metadata and serves the UI/API
- **Data Plane** — the layer that runs inside your cloud account
  - Contains the actual compute: **clusters**, **SQL warehouses**, and **Delta Live Tables** runners
  - Stores your data in your own cloud object storage (S3/ADLS/GCS)
  - Compute here runs your Spark jobs and queries
- Communication between planes is over secure channels (HTTPS, secure TLS tunnels)
- This split is why Databricks can be "serverless" — Databricks can spin up and tear down the data-plane compute on demand
- **Why it matters** — Data never leaves your cloud boundary, which satisfies security/compliance, and Databricks can patch and manage the control plane centrally

### Workspaces, Clusters, Notebooks, Jobs

- **Workspace** — the collaboration environment: notebooks, folders, libraries, and files; the UI surface of the control plane
- **Cluster** — a set of VMs (driver + workers) running Spark that executes notebooks and jobs
- **Notebook** — a document of executable cells in Python, Scala, SQL, or R with rich output rendering
- **Job** — a scheduled, production execution of notebooks, JARs, Python files, or SQL queries with retry, alerting, and dependency handling

### Unity Catalog

- Unity Catalog is Databricks' unified governance solution for managing and securing all data assets
- Provides a single place for fine-grained access control across workspaces, catalogs, schemas, and tables
- Built on a **metastore** that tracks objects and lineage
- Covers data discovery (searchable catalog), governance (grants, audit logging), and lineage (how data flows)
- See the dedicated **Unity Catalog** section below

### Databricks Runtime

- The Databricks Runtime (DBR) is the pre-configured stack that runs on clusters
- Bundles: Apache Spark, Delta Lake, optimized I/O, cloud connectors, GPU libraries, and built-in ML libraries
- Regular releases track Spark versions with Databricks-specific performance patches
- **Photon** is a high-performance native vectorized query engine that is part of recent DBRs (see Clusters section)
- DBR also includes delta-rs / Pandas / Koalas (now Polars) support for non-Spark workloads

## Workspaces and Notebooks

### What is a Databricks Workspace

- A workspace is the central hub where users collaborate on data projects
- Contains:
  - **Folders** to organize notebooks and files (Git-backed repos supported)
  - **Notebooks** for interactive development
  - **Libraries** (Python wheels, JARs) shared across clusters
  - **Dashboards** for SQL query visualization
  - **Files** (uploaded data, configs)
- Supports Git integration so notebooks can be versioned and reviewed like code
- Access to workspaces is controlled by user roles (Admin, Developer, Analyst)

### Notebooks (Python, Scala, SQL, R)

- A notebook is a sequence of cells, each containing code in a supported language
- Supported languages: **Python, Scala, SQL, R** (plus Markdown cells for documentation)
- Notebooks can mix languages: e.g., a SQL cell reading from a temp view created by a Python cell
- `%python`, `%scala`, `%sql`, `%r` magic commands switch the language of a cell
- Variables pass between cells within the same SparkSession when compatible (e.g., Python→SQL via temp views, not raw variables)

### Cells and Interactive Development

- Each cell runs independently; output renders inline (tables, charts, text, images)
- You can run all cells, a single cell, or up to a chosen cell (`Run All Above`)
- Interactive development loop: write a cell → run → inspect output → iterate
- Notebooks attach to a running cluster; multiple users can share a cluster and notebook
- Rich UI widgets allow parameterization without editing code (dropdowns, text boxes)

### Notebook Workflow for Data Exploration

- Typical exploration workflow:
  1. Create a notebook and attach it to a cluster
  2. Read a data source (e.g., `spark.read.table(...)` or Auto Loader)
  3. Explore schema, row counts, and samples with `printSchema()`, `count()`, `display()`
  4. Write transformation cells (filter, aggregate, join) iteratively
  5. Persist the result as a Delta table when the exploration is stable
  6. Convert the final cells into a scheduled job for production
- Use `display(df)` to get interactive, chart-able output rather than `df.show()`
- Example cell sequence:

```python
# Cell 1: read data
sales = spark.read.table("silver.sales")

# Cell 2: inspect
sales.printSchema()
display(sales.limit(10))

# Cell 3: aggregate
from pyspark.sql import functions as F
daily = (sales
         .groupBy("order_date")
         .agg(F.sum("amount").alias("revenue")))
display(daily.orderBy("order_date"))
```

```sql
-- Cell 4: SQL alternative against the same temp view
CREATE OR REPLACE TEMP VIEW daily_sql AS
SELECT order_date, SUM(amount) AS revenue
FROM silver.sales
GROUP BY order_date;
```

## Clusters

### What is a Databricks Cluster

- A cluster is a group of cloud VMs running the Databricks Runtime with a Spark driver and worker nodes
- The **driver** hosts the SparkSession, runs your notebooks/application, and coordinates work
- **Workers** execute the tasks; they cache data and run the computation
- Clusters are ephemeral by design — spin up when needed, terminate when done

### Cluster Types

- **Interactive clusters** — shared, long-lived clusters for development and ad-hoc exploration
  - Users attach notebooks and share the compute
  - Stay running until terminated (manual or auto-termination)
- **Job clusters** — created automatically when a job runs, then destroyed
  - Ephemeral, isolated, cost-efficient for production schedules
  - No idle cost — you only pay while the job runs
- **Serverless compute** — Databricks-managed compute that needs no cluster configuration
  - You never provision VMs; Databricks scales instantly
  - Used for serverless SQL warehouses and serverless notebooks
  - No infrastructure to manage and no cold-start tuning

### Cluster Configuration

- **Databricks Runtime version** — pick the Spark/DBR version (and whether Photon is enabled)
- **Node types** — the VM types for the driver and workers (e.g., `m5.xlarge` on AWS)
  - Choose CPU vs memory optimized based on the workload; use GPU types for ML training
- **Autoscaling** — workers scale between a minimum and maximum range based on load
  - Spark looks at pending tasks and backlog to decide scaling
  - Reduces cost during quiet periods, adds capacity during bursts
- **Spark config** — custom `spark.*` settings (memory, shuffle partitions, dynamic allocation)
- **Environment variables and libraries** — install Python wheels, Maven packages, init scripts
- **Access mode** — single-user vs shared (affects isolation and how credentials are scoped)
- **Auto-termination** — set an idle timeout (e.g., 30–120 min) so dev clusters don't run forever

### Cluster Modes

- **Single-node mode** — cluster with only a driver, no workers
  - Cheap for development, small data, and non-Spark libraries (pandas, MLlib on one node)
  - All work runs locally on the driver
- **Standard (multi-node) mode** — one driver plus one or more workers
  - True distributed processing across the cluster
  - Used for any real-scale data workload

### Auto-Termination

- Clusters can be configured to automatically terminate after a period of inactivity
- Prevents runaway costs from developers leaving clusters running overnight
- Jobs that fail or finish terminate their job clusters automatically
- The auto-termination timeout is set at cluster creation (commonly 30–120 minutes for dev)

### Photon Engine

- **Photon** is Databricks' native, vectorized query engine written in C++ that accelerates SQL and DataFrame workloads
- It replaces parts of the Spark execution plan with highly optimized operators
- Achieves order-of-magnitude speedups on SQL analytics, especially scans, aggregations, and joins
- Enabled per-cluster; works on Delta tables automatically; no code changes needed
- Not available on all node types (needs supported hardware) and is a paid add-on for some plans
- Best for BI-style analytical SQL; complex UDF-heavy Spark pipelines may not benefit as much

### Why Clusters Matter

- **Compute isolation** — each cluster is its own set of VMs; a runaway job on one cluster does not affect another
- **Scaling** — clusters scale horizontally by adding/removing workers, so you can handle growing data volumes
- **Cost control** — decoupling compute from storage means you can terminate compute when idle and keep data in cheap object storage
- **Workload separation** — keep dev, prod, BI, and ML workloads on separate clusters so they don't contend

## Delta Lake

### What is Delta Lake

- Delta Lake is an open-source **transactional storage layer** on top of Parquet
- It turns plain Parquet files into a managed, versioned table with database-grade guarantees
- Adds: ACID transactions, schema enforcement/evolution, time travel, UPSERTs, and scalable metadata
- Fully compatible with the Spark ecosystem and increasingly with other engines (Trino, Flink, DuckDB via delta-rs)
- Storage format is open (Parquet) — your data is never locked into a proprietary format

### Delta Table Structure

- A Delta table is a directory of **Parquet data files** plus a hidden `_delta_log` directory
- `_delta_log` contains JSON files (`00000000000000000000.json`, `...00001.json`, etc.) recording every transaction
- Each commit (JSON entry) lists which files were added and removed by that operation
- Periodically, checkpoints (Parquet files of the log) compact the log for faster startup reads
- Example layout:

```
mytable/
  _delta_log/
    00000000000000000000.json
    00000000000000000001.json
    00000000000000000002.json
    _last_checkpoint
  part-00000-xxx-c000.snappy.parquet
  part-00001-xxx-c000.snappy.parquet
```

### ACID Transactions in Delta Lake

- **Atomicity** — every write either fully succeeds or is not applied; no partial table states
- **Consistency** — the transaction log ensures readers see a consistent snapshot
- **Isolation** — concurrent writers do not see each other's uncommitted changes
- **Durability** — committed changes are written to storage and survive failures
- Implemented via **optimistic concurrency control**: writers attempt commits and retry on conflict
- This is why concurrent Spark jobs and BI queries can safely read/write the same table

### Time Travel

- Delta keeps a version history of the table, so you can query any past version
- **VERSION AS OF** — query by commit version number

```sql
SELECT * FROM sales VERSION AS OF 42;

-- Or in Python:
df = spark.read.option("versionAsOf", 42).table("sales")
```

- **TIMESTAMP AS OF** — query by timestamp

```sql
SELECT * FROM sales TIMESTAMP AS OF '2026-01-01T00:00:00';

-- Python:
df = spark.read.option("timestampAsOf", "2026-01-01T00:00:00").table("sales")
```

- Common uses: debugging bad data, reproducing a report "as of" a date, rolling back mistakes
- History is retained until `VACUUM` removes old files — set retention carefully
- `DESCRIBE HISTORY sales;` shows the full commit history

### Schema Enforcement and Schema Evolution

- **Schema enforcement** — Delta rejects writes whose schema does not match the table
  - Prevents accidentally writing wrong-typed or extra columns that silently corrupt data
- **Schema evolution** — explicitly allow compatible changes (add columns, widen types)
- Enable with a merge schema option or SQL DDL:

```sql
-- Add a column
ALTER TABLE sales ADD COLUMNS (new_col STRING);
```

```python
(df
 .write
 .option("mergeSchema", "true")
 .mode("append")
 .saveAsTable("sales"))
```

### Merge, Update, Delete (UPSERT)

- Delta supports full DML on tables: `MERGE`, `UPDATE`, `DELETE`
- **MERGE** is the UPSERT operation — insert new rows, update or delete matched ones

```sql
MERGE INTO sales AS target
USING incoming_orders AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET
  amount = source.amount,
  status = source.status
WHEN NOT MATCHED THEN INSERT (order_id, customer_id, amount, status)
VALUES (source.order_id, source.customer_id, source.amount, source.status);
```

```python
# Python DataFrame equivalent
from delta.tables import DeltaTable
target = DeltaTable.forName(spark, "sales")
target.alias("t").merge(
    source=incoming.alias("s"),
    condition="t.order_id = s.order_id"
).whenMatchedUpdate(set={
    "amount": "s.amount", "status": "s.status"
}).whenNotMatchedInsert(values={
    "order_id": "s.order_id",
    "customer_id": "s.customer_id",
    "amount": "s.amount",
    "status": "s.status"
}).execute()
```

- `UPDATE` and `DELETE` with `WHERE` clauses for targeted corrections

```sql
DELETE FROM sales WHERE status = 'cancelled';
UPDATE sales SET discount = 0.1 WHERE region = 'EU';
```

### Z-Order Optimization

- **Z-Ordering** co-locates data with similar values in the same files, dramatically improving filter and join performance on the Z-ordered columns
- Unlike partitioning, it does not create new directories — it clusters within files
- Use on columns frequently used in `WHERE` clauses with high cardinality

```sql
OPTIMIZE sales ZORDER BY (customer_id, order_date);
```

```python
from delta.tables import DeltaTable
DeltaTable.forName(spark, "sales").optimize().executeZOrderBy("customer_id", "order_date")
```

- Best practice: Z-order on the 1–2 columns you filter/join most, and re-run periodically as data changes

### Delta Lake vs Apache Iceberg vs Hudi

- **Delta Lake** — Databricks-backed; tight Spark integration, strongest within Databricks; open source (Linux Foundation)
- **Apache Iceberg** — Netflix-originated; strong multi-engine support (Spark, Flink, Trino, Dremio); table format is engine-agnostic by design
- **Apache Hudi** — Uber-originated; strong on incremental upsert-heavy ingestion and streaming writes
- All three are "table formats" providing ACID + time travel on lake storage
- All are now open source and support similar core features (MERGE, time travel, CDC)
- Choosing factors: ecosystem integration, engine support, operational maturity in your org, and managed-service support (Delta is native to Databricks; Iceberg is native to Snowflake/AWS Athena)

### Streaming with Delta

- Delta tables support **streaming reads and writes** via Spark Structured Streaming
- Streaming writes: a stream continuously appends to a Delta table

```python
(spark.readStream
 .format("cloudFiles")          # Auto Loader
 .option("cloudFiles.format", "json")
 .load("/path/to/landing")
 .writeStream
 .format("delta")
 .option("checkpointLocation", "/path/to/checkpoint")
 .outputMode("append")
 .toTable("bronze.events"))
```

- Streaming reads: consume a Delta table as a stream to process new rows as they appear

```python
(spark.readStream
 .table("silver.sales")
 .writeStream
 .format("delta")
 .option("checkpointLocation", "/path/to/ckpt2")
 .toTable("gold.daily_sales"))
```

- **Change Data Feed (CDF)** — captures row-level changes (inserts, updates, deletes) on a Delta table
  - Enabled via table property: `ALTER TABLE t SET TBLPROPERTIES (delta.enableChangeDataFeed = true)`
  - Downstream systems can consume only the changed rows rather than re-reading everything
  - Powers incremental pipelines, CDC to other stores, and streaming joins
- Star schemas and incremental aggregates are often built with streaming Delta reads instead of full reloads

## Spark SQL and DataFrames

### DataFrame vs RDD vs SQL

- **RDD (Resilient Distributed Dataset)** — low-level primitive: raw JVM objects, no schema
  - Most control, least optimization; the Catalyst optimizer cannot optimize arbitrary RDD code
  - Rarely needed in practice; used for custom low-level operations
- **DataFrame** — schema'd distributed collection of rows backed by the Catalyst optimizer
  - The primary API in Python (`pyspark.sql.DataFrame`), Scala, and R
  - Optimizes via Catalyst and executes on Tungsten — preferred for most work
- **SQL** — declarative queries against tables and views, compiled into the same execution plan as DataFrames
  - Best for analysts, for expressing relational logic, and for interacting with Delta tables/catalog
- Rule of thumb: prefer DataFrame/SQL for almost everything; use RDDs only for bespoke low-level code
- DataFrames and SQL share the same underlying plan, so performance is similar; pick by readability

### SparkSession and Spark Commands

- `SparkSession` is the unified entry point introduced in Spark 2.0 (replaced `SparkContext`/`SQLContext`/`HiveContext`)
- In notebooks it is pre-created as `spark`

```python
spark.version            # Spark version
spark.sparkContext       # underlying low-level context
spark.catalog.listTables()   # list tables in catalog
spark.conf.set("spark.sql.shuffle.partitions", "200")
```

- Common commands: `spark.read`, `spark.readStream`, `spark.table()`, `spark.sql()`, `spark.createDataFrame()`

### Reading and Writing Data

- Reading CSV, JSON, Parquet:

```python
# CSV
df = (spark.read
      .option("header", "true")
      .option("inferSchema", "true")
      .csv("/data/raw/sales"))

# JSON
df = spark.read.json("/data/raw/events")

# Parquet
df = spark.read.parquet("/data/raw/snapshots")

# From a table
df = spark.table("silver.sales")
```

- Writing:

```python
df.write.mode("overwrite").parquet("/data/out/sales_parquet")
df.write.mode("append").saveAsTable("silver.sales")

# Partitioned write
df.write.mode("overwrite").partitionBy("year", "month").saveAsTable("silver.sales")
```

- JDBC (with appropriate driver on the cluster):

```python
df = (spark.read
      .format("jdbc")
      .option("url", "jdbc:postgresql://dbhost:5432/appdb")
      .option("dbtable", "orders")
      .option("user", "app")
      .option("password", "secret")
      .load())
```

- Note: for production, prefer Delta tables and connectors (JDBC pushdown, Azure Synapse connector, etc.)

### Transformations

- Transformations build a logical plan; they are **lazy** — nothing executes until an action triggers it

```python
from pyspark.sql import functions as F

filtered = sales.filter(F.col("amount") > 100)
selected = sales.select("order_id", "customer_id", "amount")
renamed = sales.withColumnRenamed("amt", "amount")
new_col = sales.withColumn("revenue_usd", F.col("amount") * F.col("fx_rate"))

grouped = (sales
           .groupBy("customer_id")
           .agg(F.sum("amount").alias("total_spend"),
                F.countDistinct("order_id").alias("order_count")))

joined = (orders
          .join(customers, on="customer_id", how="inner")
          .select("order_id", "customer_name", "amount"))
```

- Common functions: `F.col`, `F.lit`, `F.when/otherwise`, `F.sum`, `F.count`, `F.avg`, `F.min`, `F.max`, `F.coalesce`, `F.date_trunc`, `F.window` (streaming tumbling windows)

### Actions

- Actions trigger actual computation and return results

```python
df.collect()        # return all rows to the driver as a list (DANGEROUS on large data)
df.count()          # count rows
df.show(10)         # print the first rows
df.first()          # first row
df.take(5)          # first n rows
df.write.saveAsTable(...)   # writing is an action too
```

- `collect()` pulls everything to the driver and can OOM the driver — always limit, aggregate, or write to disk instead

### Lazy Evaluation

- Spark builds a **directed acyclic graph (DAG)** of transformations before executing anything
- Transformations are recorded as a plan; execution is deferred until an action (like `count()` or `write`) is called
- Benefits:
  - Spark can **reorder and combine** operations (filter pushdown, predicate pruning)
  - Unused transformations are never computed
  - Multiple reads of the same source can be shared in one pass
- Example: if you `select` a column and never use it downstream, Spark may skip computing it

### Catalyst Optimizer

- Catalyst is Spark's query optimizer; it turns logical plans into efficient physical plans
- Optimization phases:
  1. **Analysis** — resolve column names and types against the catalog
  2. **Logical optimization** — constant folding, predicate pushdown, projection pruning
  3. **Physical planning** — choose join strategies (broadcast vs shuffle), add exchange nodes
  4. **Code generation** — generate JVM bytecode for hot paths
- `df.explain(True)` shows the analyzed, optimized, and physical plans — useful for debugging performance
- Example optimizations:
  - **Predicate pushdown** — filters applied at the storage layer (e.g., Delta/Parquet stats) so fewer rows are read
  - **Projection pruning** — only read the columns you actually use
  - **Broadcast joins** — small tables are shipped to every worker instead of shuffled

### Tungsten Execution Engine

- Tungsten is Spark's runtime engine that optimizes memory and CPU usage
- Uses **off-heap binary memory** with cache-friendly layout instead of JVM objects
- **Whole-stage code generation** — fuses operators into a single optimized function, reducing virtual calls and overhead
- Vectorized columnar processing for common operations
- Result: dramatically lower CPU cost and more predictable memory vs classic Spark/MapReduce execution

## Databricks SQL

### SQL Warehouses (Compute for BI/SQL)

- A **SQL warehouse** is a compute cluster purpose-built for SQL queries and BI (like an "analytics engine")
- Replaces classic warehouse VMs — it's compute over your lakehouse data
- Three classes:
  - **Serverless** — fully managed, instant startup, scales to zero; easiest to operate
  - **Pro** — optimized for concurrency and performance at a higher price point
  - **Classic** — general-purpose SQL compute, lower cost
- You set size (Small/Medium/Large etc., or serverless auto), scaling limits, and auto-stop behavior
- Multiple users and BI tools can query concurrently without sharing notebooks or clusters

### Querying Delta Tables with SQL

- Databricks SQL queries Delta tables directly via Unity Catalog three-level names: `catalog.schema.table`

```sql
SELECT customer_id,
       SUM(amount) AS total_spend
FROM prod_sales.silver.sales
WHERE order_date >= '2026-01-01'
GROUP BY customer_id
ORDER BY total_spend DESC;
```

- Full SQL support: CTEs, window functions, DML, `OPTIMIZE`/`VACUUM`, and `DESCRIBE HISTORY`

```sql
WITH monthly AS (
  SELECT DATE_TRUNC('month', order_date) AS month,
         SUM(amount) AS revenue
  FROM prod_sales.silver.sales
  GROUP BY 1
)
SELECT month, revenue,
       LAG(revenue) OVER (ORDER BY month) AS prev_month,
       revenue - LAG(revenue) OVER (ORDER BY month) AS delta
FROM monthly;
```

- BI integration: connect Tableau, Power BI, Looker, Superset via JDBC/ODBC endpoints
- Dashboards in Databricks SQL let analysts build and schedule visualizations from queries

### Delta Live Tables (DLT) — Declarative Pipelines

- DLT is a framework for building **declarative, reliable data pipelines** as code
- You write what the output should be (Python or SQL) and DLT handles execution, orchestration, and quality
- Tables are declared with `@dlt.table` (Python) or `CREATE OR REPLACE LIVE TABLE` (SQL)
- DLT manages dependencies between tables, computes incrementally, and handles retries/failure recovery
- Built-in **data quality constraints** (expectations) that can drop, quarantine, or fail on bad rows

```python
import dlt
from pyspark.sql import functions as F

@dlt.table
def silver_sales():
    return (spark.readStream
            .table("bronze.sales")
            .withColumn("clean_amount", F.col("amount").cast("decimal(12,2)")))

@dlt.table
@dlt.expect("positive_amount", "amount > 0")
def gold_daily_revenue():
    return (spark.readStream
            .table("LIVE.silver_sales")
            .groupBy("order_date")
            .agg(F.sum("clean_amount").alias("revenue")))
```

```sql
CREATE OR REPLACE LIVE TABLE silver_sales AS
SELECT *, CAST(amount AS DECIMAL(12,2)) AS clean_amount
FROM LIVE.bronze_sales;

CREATE OR REPLACE LIVE TABLE gold_daily_revenue (
  CONSTRAINT positive_amount EXPECT (amount > 0)
) AS
SELECT order_date, SUM(clean_amount) AS revenue
FROM LIVE.silver_sales
GROUP BY order_date;
```

- Great for medallion pipelines (bronze → silver → gold) with automated quality checks and incremental processing

### Databricks SQL vs PySpark Notebooks

- **Databricks SQL** — declarative, for analysts and BI; runs on SQL warehouses; simpler, governed, dashboard-ready
- **PySpark notebooks** — programmatic, for engineers and data scientists; run on clusters; handle complex ETL logic, ML, and streaming
- Use SQL for reporting and simple transformations; use notebooks for complex pipelines, ML, and custom logic
- Both read the same Delta tables, so teams can mix approaches on one lakehouse

## Unity Catalog

### What is Unity Catalog

- Unity Catalog is Databricks' unified data governance layer for all data assets across the lakehouse
- Provides one place to discover, manage, and secure data for the whole organization
- Replaces scattered per-cluster metastores with a central, fine-grained governance model
- Features: centralized access control, data discovery, automated lineage, audit logging, and data search

### Metastore, Catalog, Schema, Tables Hierarchy

- The hierarchy is: **Metastore → Catalog → Schema → Table/View** (and volumes/functions/models)

```
my_metastore
  └── catalog: prod_sales
        ├── schema: bronze
        │     └── table: raw_orders
        ├── schema: silver
        │     └── table: sales
        └── schema: gold
              └── table: daily_revenue
```

- A **metastore** is the top-level container of metadata and access policies; it can span multiple workspaces
- A **catalog** groups schemas (analogous to a database); used to separate environments (dev/prod) or domains
- A **schema** contains tables, views, and functions
- A **table** is the actual data (managed or external Delta table)
- Fully qualified reference in SQL: `catalog.schema.table`

```sql
SHOW CATALOGS;
SHOW SCHEMAS IN prod_sales;
SHOW TABLES IN prod_sales.silver;
SELECT * FROM prod_sales.silver.sales;
```

### Data Discovery and Access Control (Grants)

- Users can search the catalog for data assets, view metadata, and inspect lineage
- Access control is **role-based** via grants; admin grants privileges to users/groups/service principals

```sql
GRANT SELECT ON TABLE prod_sales.silver.sales TO analyst_group;
GRANT SELECT, MODIFY, CREATE ON SCHEMA prod_sales.silver TO engineering_group;

-- Revoke
REVOKE SELECT ON TABLE prod_sales.silver.sales FROM analyst_group;
```

- Privileges: `SELECT`, `MODIFY` (insert/update/delete), `CREATE`, `USAGE`, `OWNERSHIP`, etc.
- Fine-grained options: column-level masking, row-level filters, and dynamic views for complex rules
- **Lineage** tracks how tables were produced (which source tables and notebooks/jobs), enabling impact analysis and compliance
- **Audit logging** records every access and governance event to a system table

### Why Unity Catalog Matters

- **Centralized security** — one place to govern access instead of per-cluster Hive metastores with scattered permissions
- **Consistent policy** — the same policies apply across all workspaces, engines (SQL, Python, ML), and clouds
- **Compliance** — audit logs and lineage make it possible to prove data provenance for regulations (GDPR, SOX, HIPAA)
- **No lock-in for metadata** — you can query the same catalog from any cluster or SQL warehouse
- **Self-service** — users discover what data exists, who owns it, and how to access it

## Databricks Jobs and Orchestration

### What are Databricks Jobs

- Jobs are the production scheduling mechanism in Databricks
- They run **notebooks, Python scripts, JAR files, or SQL queries** on a schedule
- Each run gets an isolated job cluster (or uses a pre-existing cluster) and captures logs, metrics, and history
- Key features: schedules, retries, timeouts, alerting, and email/webhook notifications
- Jobs are the standard way to move work from interactive notebooks into reliable production pipelines

### Job Tasks

- A job can have **one task or many tasks** that run as a graph
- **Notebook task** — run a notebook; can pass parameters to it
- **Python task** — run a `.py` script or a wheel entry point
- **JAR task** — run a Scala/Java Spark application
- **SQL task** — run a SQL file (created via Databricks SQL), a warehouse query, or a DLT pipeline
- **dbt task** — run dbt models natively (integration built in)
- **DLT task** — trigger a Delta Live Tables pipeline
- Example notebook task with parameters:

```python
# In the notebook, read parameters from the job
import json
params = json.loads(dbutils.widgets.text("parameters", "{}"))
run_date = params.get("run_date", "")
```

- Or use notebook widgets:

```python
dbutils.widgets.text("run_date", "2026-01-01")
run_date = dbutils.widgets.get("run_date")
```

### Workflows with Dependencies

- Multi-task jobs support **dependency graphs**: tasks run in parallel where independent, and only after their dependencies succeed

```
        ┌──────────────┐
        │ ingest (bronze)│
        └──────┬───────┘
        ┌──────┴───────┐
        │   clean      │
        └──────┬───────┘
        ┌──────┴───────┐      ┌───────────────┐
        │   aggregate   │──────│ publish to BI │
        └──────────────┘      └───────────────┘
```

- Configure via UI, CLI, or the `jobs` API / Terraform
- Conditional execution: use `on_failure`/`on_success` behaviors and `if/else` or conditional tasks
- Tasks support retry policies (e.g., retry 3 times with 10-minute backoff), timeouts, and email alerts

### Databricks vs Airflow for Orchestration

- **Databricks Jobs** — native, no extra infrastructure, deeply integrated with notebooks/SQL/DLT, easiest for pure-Databricks pipelines
  - Limited for complex external dependencies, non-Databricks systems, or fine-grained DAG control
- **Apache Airflow** — general-purpose orchestrator that can coordinate Databricks runs alongside Kafka, databases, APIs, and services
  - Rich DAG features (sensors, XComs, custom operators), but requires you to run and maintain Airflow
  - The `DatabricksSubmitRunOperator` / `DatabricksRunNowOperator` let Airflow trigger Databricks jobs
- Common pattern: **Airflow orchestrates; Databricks executes heavy Spark workloads**
- Small orgs/all-Databricks: use Jobs; complex multi-system workflows: use Airflow (or Dagster/Prefect) calling Databricks

### Parameterized Job Runs

- Pass parameters to jobs so the same job can run for different dates, regions, or configs
- **Notebook parameters** — via widgets or a `parameters` JSON blob in the task definition
- **Python task parameters** — passed as command-line arguments:

```python
# Job task: python script with args
my_script.py --run-date 2026-01-01 --env prod

# Inside the script
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("--run-date", required=True)
args = parser.parse_args()
```

- **Run-now** lets you trigger the job with ad-hoc parameters for backfills or testing
- Use parameters to make pipelines idempotent: `run_date` determines which partitions to overwrite
- Jobs UI/API show run history with parameters, logs, and durations for each run

## Lakehouse Architecture

### What is a Lakehouse

- A lakehouse merges the best of **data lakes** (cheap, flexible, open formats) and **data warehouses** (ACID, governance, SQL performance)
- Data lives once in open, columnar files (Delta on S3/ADLS/GCS); a transaction layer adds reliability
- One system serves BI dashboards, ad-hoc SQL, data engineering, and ML — no need to copy data into a separate warehouse
- Lakehouse vs warehouse: warehouse imposes proprietary storage and compute coupling; lakehouse decouples them

### Bronze, Silver, Gold Layers (Medallion Architecture)

- The **medallion architecture** organizes a lakehouse into progressively refined layers
- Each layer is usually a set of Delta tables in separate schemas (`bronze`, `silver`, `gold`)
- Data flows through with increasing structure, quality, and business value

### Bronze: Raw Ingested Data

- **Purpose** — ingest everything exactly as it arrives; a faithful copy of source data
- Store raw JSON/Parquet/AVRO payloads as-is (schema not strictly enforced; keep full fidelity)
- Preserve the original fields and timestamps; you may add ingestion metadata (e.g., `_load_timestamp`, source filename)
- Never delete or overwrite bronze data — it is your immutable source of truth for re-processing
- Loaded via Auto Loader, JDBC ingestion, or streaming
- Example:

```sql
CREATE OR REPLACE TABLE bronze.orders (
  order_id BIGINT,
  payload STRING,          -- raw JSON body kept verbatim
  source_file STRING,
  _load_timestamp TIMESTAMP
) USING DELTA;
```

### Silver: Cleansed, Conformed Data

- **Purpose** — clean, deduplicate, validate, and shape data into a conformed, query-friendly model
- Typical steps: type casting, null/duplicate handling, joins to reference data, standardizing naming and formats
- Silver tables are still granular (one row per order/event), not aggregated
- This is the layer where most data quality checks live (row counts, nulls, referential integrity)
- Example:

```python
from pyspark.sql import functions as F

cleaned = (spark.readStream
           .table("bronze.orders")
           .select(
               F.col("order_id"),
               F.from_json(F.col("payload"), schema).alias("data"),
               F.col("_load_timestamp"))
           .filter(F.col("data").isNotNull())
           .dropDuplicates(["order_id"]))

cleaned.writeStream.format("delta").toTable("silver.orders")
```

### Gold: Business-Ready Aggregates

- **Purpose** — provide curated, aggregated, denormalized data optimized for dashboards, ML, and reporting
- Heavy aggregations (daily revenue, cohort metrics, KPIs), star-schema fact/dimension tables, or feature tables
- Usually small and fast to query; the layer BI tools and data scientists consume directly
- Example:

```sql
CREATE OR REPLACE TABLE gold.daily_revenue
USING DELTA
AS
SELECT DATE_TRUNC('day', order_ts) AS order_date,
       region,
       SUM(amount) AS revenue,
       COUNT(DISTINCT customer_id) AS active_customers
FROM silver.orders o
JOIN silver.customers c ON o.customer_id = c.customer_id
GROUP BY 1, 2;
```

### Why Medallion Architecture Works

- **Isolation of concerns** — raw data is preserved (bronze), logic is centralized (silver), and consumers get clean outputs (gold)
- **Reprocessability** — because bronze is immutable, you can rebuild silver/gold from scratch if logic changes
- **Quality gates** — each layer can enforce checks before promoting data to the next
- **Performance** — gold tables are small and aggregated, so dashboards are fast without scanning everything
- **Team alignment** — data engineers own bronze/silver; analysts own gold; everyone has a clear contract

### Delta Lake as the Foundation of the Lakehouse

- Without ACID and reliable metadata, a lake is just files — the lakehouse depends on Delta's guarantees
- Delta provides the reliability (atomic commits, schema enforcement) that lets you run "warehouse" workloads on lake storage
- Features like time travel, CDF, and Z-Ordering make the lakehouse practical for production BI and ML
- Because Delta is an open format, the lakehouse avoids vendor lock-in that proprietary warehouses impose

## Auto Loader

### What is Auto Loader

- Auto Loader is Databricks' incremental data ingestion tool for loading files from cloud object storage into Delta tables
- It monitors a directory for **new files** and loads only those, incrementally and reliably
- Supports JSON, CSV, Parquet, AVRO, ORC, XML, and text
- Declared via `cloudFiles` source in Spark Structured Streaming
- Example:

```python
(spark.readStream
 .format("cloudFiles")
 .option("cloudFiles.format", "csv")
 .option("header", "true")
 .option("cloudFiles.schemaLocation", "/data/_schema")   # schema registry
 .load("s3://my-bucket/landing/orders/")
 .writeStream
 .format("delta")
 .option("checkpointLocation", "/data/_checkpoints/orders")
 .outputMode("append")
 .toTable("bronze.orders"))
```

### How It Tracks Files (Checkpointing)

- Auto Loader uses **checkpoints** to remember which files have already been processed
- Two modes:
  - **Directory listing** — lists the directory and tracks processed file paths in the checkpoint; no permissions needed beyond list/read
  - **File notification (recommended for high volume)** — uses cloud-native event notifications (SQS on AWS, Event Grid on Azure, Pub/Sub on GCP) so new files trigger streaming immediately and scale to millions of files
- The checkpoint (a Delta-like log) stores the file queue and processing state, making ingestion **exactly-once** and resumable after failures
- Because it tracks raw file names, it is idempotent — restarting does not re-ingest already-seen files
- Schema inference can be cached (via `schemaLocation`) so schema changes don't break the pipeline

### When to Use Auto Loader

- **Best when:** continuously arriving files in cloud storage (CSV/JSON dumps, logs, IoT data, partner feeds)
- **When not to use:** data already in a database (use JDBC), or a few large one-time historical files (a plain batch read is fine)
- Also supports a **batch mode** (`cloudFiles` used with a normal `read`) to backfill historical files before switching to streaming
- Ideal first step of a bronze layer in the medallion architecture
- Combine with `inferColumnTypes` and schema evolution for loosely-structured inputs

## Databricks and Snowflake Comparison

### Compute / Storage Models

- **Databricks** — **decoupled** compute and storage; compute (clusters/SQL warehouses) spins up on demand over your own object storage (S3/ADLS/GCS); you pay for compute time plus cloud storage
- **Snowflake** — also separates compute and storage, but storage is managed by Snowflake in its own account (multi-cluster shared data architecture); you pay for virtual warehouses (compute) and storage separately
- Both scale compute up/down independently of storage; both separate billing for compute vs storage
- Databricks compute is Spark-based (plus Photon for vectorized execution); Snowflake uses its own proprietary columnar engine

### Lakehouse vs Warehouse

- **Databricks = lakehouse** — open formats (Parquet/Delta), runs on your cloud storage, best for data engineering/ETL, streaming, ML, and BI together; code-first (Python/Scala/SQL)
- **Snowflake = cloud warehouse** — proprietary storage format, best-in-class SQL/BI performance and concurrency; simple to operate; less natural for ML and code-heavy pipelines (though Snowpark adds Python/Scala)
- Databricks excels at complex transformations, ML/AI workloads, and streaming; Snowflake excels at governed, high-concurrency SQL analytics and zero-maintenance warehousing

### When to Choose Which

- Choose **Databricks** when: heavy ETL/ELT, streaming, ML/AI, open formats, data engineering teams using Python/Spark, or you need a lakehouse across many data types
- Choose **Snowflake** when: primarily BI/SQL analytics, need extreme concurrency with minimal ops, want a managed proprietary warehouse, or the org is SQL-first and values simplicity
- Real world: many orgs run both — Databricks for engineering/ML and Snowflake for warehouse analytics, syncing data between them (a pattern worth mentioning in interviews)
- Other competitors in the space: AWS Redshift, Google BigQuery, Azure Synapse/Fabric

## Common Mistakes

### Running Expensive Transformations on Large Clusters

- **Problem** — paying for 20 nodes when the job is small; or running a tiny job on a huge interactive cluster
- **Why it looks correct** — more nodes feel like a safe default; the cluster was already running
- Use the smallest cluster that meets the job's needs; use job clusters with right-sizing; enable autoscaling so idle nodes scale down

### Not Using Delta Lake (Plain Parquet, No ACID)

- **Problem** — writing plain Parquet gives no atomicity, no time travel, no schema enforcement, and corrupts on partial writes
- **Why it looks correct** — Parquet is fast and familiar
- Always write to Delta tables for anything that will be reused or updated; reserve plain Parquet for export-only data

### Not Partitioning / Z-Ordering Large Tables

- **Problem** — scanning billions of rows on every query because the table is one giant unsorted heap
- **Why it looks correct** — queries work, just slowly
- Partition by low-cardinality, frequently-filtered columns (date); Z-order on high-cardinality filter/join columns; keep partitions under ~50 GB and avoid over-partitioning (thousands of tiny files)

### Using collect() on Huge DataFrames

- **Problem** — `df.collect()` ships every row to the driver, blowing up memory and freezing the notebook
- **Why it looks correct** — you just wanted to "see" the data
- Use `display()`, `show(n)`, `take(n)`, `limit(n)`, or write results to Delta and sample from the table; `collect()` only for genuinely small results

### Over-Complicating the Bronze Layer

- **Problem** — filtering, cleaning, joining, and aggregating during ingestion into bronze
- **Why it looks correct** — it seems efficient to clean data "on the way in"
- Bronze should be an immutable, faithful copy of the source; do cleansing/conforming in silver so you never lose raw data and can re-run logic

### Ignoring Table Maintenance (VACUUM, OPTIMIZE)

- **Problem** — tables accumulate thousands of small files and old versions; queries slow down and storage bloats
- **Why it looks correct** — the table still returns correct results
- Run `OPTIMIZE` to compact small files (and optionally Z-order), run `VACUUM` with a safe retention window (respecting time-travel needs) to clean up old files; automate with a scheduled job:

```sql
OPTIMIZE prod_sales.silver.sales ZORDER BY (customer_id);
VACUUM prod_sales.silver.sales RETAIN 168 HOURS;
```

- Also monitor file counts and sizes and add auto-compaction for streaming writes

## Interview Questions

### Foundational Concepts

1. **What is Databricks and how does it differ from a traditional data warehouse?**
   - Databricks is a unified lakehouse platform built on Apache Spark. It stores data in open formats (Delta/Parquet) on object storage and layers ACID, SQL, and governance on top, so it serves data engineering, ML, and BI on one platform. A traditional warehouse (e.g., Redshift, older Snowflake) uses proprietary storage and is optimized purely for SQL/BI.

2. **Explain the lakehouse concept.**
   - A lakehouse combines the low-cost, open, flexible storage of a data lake with the reliability, governance, and SQL performance of a warehouse. It stores data once in an open format (Delta Lake) and supports ACID transactions, time travel, schema enforcement, and BI queries without copying data into a proprietary warehouse.

3. **What is Delta Lake and why is it important?**
   - Delta Lake is an open-source transactional storage layer on top of Parquet. It adds a transaction log (`_delta_log`) that gives ACID transactions, time travel, schema enforcement/evolution, and scalable metadata handling. It is important because it makes a data lake reliable enough to serve as a production lakehouse.

4. **What is the control plane vs data plane in Databricks?**
   - The control plane is Databricks-managed and hosts the UI, notebooks, scheduler, and metadata. The data plane runs in the customer's cloud account and contains the actual compute (clusters, SQL warehouses) and data in the customer's object storage. This separation keeps data in the customer's cloud while Databricks manages the platform.

### Delta Lake Deep Dive

5. **How does Delta Lake provide ACID transactions?**
   - Delta Lake uses a transaction log of ordered commits. Every write appends a JSON commit to `_delta_log` listing added/removed files. Writers use optimistic concurrency control — they attempt a commit and retry if the version has moved. Readers read a consistent snapshot from the log, and committed files are durably written to storage.

6. **What is time travel in Delta Lake and when would you use it?**
   - Time travel lets you query a table as of a past version using `VERSION AS OF` or `TIMESTAMP AS OF`, or read a specific version via Spark options. Use it to debug bad data, reproduce historical reports, audit changes, or roll back accidental writes. Old files must be retained (not yet VACUUMed) for time travel to work.

7. **Explain MERGE (UPSERT) in Delta Lake and give an example.**
   - `MERGE` performs an upsert: rows matching a key are updated and non-matching rows are inserted (and optionally deletes). It is used for CDC, SCD Type 2, and correcting tables. Example: `MERGE INTO sales t USING new_sales s ON t.id = s.id WHEN MATCHED THEN UPDATE SET amount = s.amount WHEN NOT MATCHED THEN INSERT *`.

8. **What are Z-Order and partition pruning, and when do you use each?**
   - Partitioning groups data into directories by a low-cardinality column (e.g., date), letting queries skip entire directories. Z-Ordering clusters values within files by high-cardinality filter/join columns, speeding up range filters without creating too many files. Partition for coarse date-based pruning; Z-order for high-cardinality lookup columns.

9. **How is Delta Lake different from Apache Iceberg and Hudi?**
   - All three are open table formats providing ACID and time travel on lake storage. Delta has the tightest integration with Spark/Databricks and broad tooling. Iceberg has strong multi-engine support (Spark, Flink, Trino) and is engine-agnostic. Hudi is strong on incremental upsert-heavy ingestion. Choice depends on engine ecosystem, streaming needs, and operational fit.

10. **What is the Change Data Feed (CDF) in Delta Lake?**
    - CDF captures row-level inserts, updates, and deletes on a Delta table when enabled (`delta.enableChangeDataFeed = true`). Downstream consumers can read only the changed rows — enabling incremental pipelines, streaming joins, and CDC to external systems without full table re-reads.

### Clusters and Compute

11. **Describe the cluster types in Databricks and when you would use each.**
    - Interactive clusters for shared development and ad-hoc exploration; job clusters that are created per-job-run for production ETL (ephemeral and isolated); serverless compute that Databricks manages end-to-end for zero infra overhead. Choose job/serverless for production, interactive for development.

12. **What is the Photon engine?**
    - Photon is Databricks' native C++ vectorized query engine that accelerates SQL and DataFrame workloads on clusters where it is enabled. It executes many Spark operations natively for order-of-magnitude speedups on analytics queries, with no code changes — you just enable it on the cluster.

13. **Why does cluster configuration matter (node types, autoscaling, auto-termination)?**
    - Node type determines CPU/memory per worker; autoscaling matches workers to load and saves money during quiet periods; auto-termination stops idle interactive clusters from running up bills. Right-sizing these controls both performance and cost.

### Spark and DataFrames

14. **What is lazy evaluation in Spark?**
    - Spark builds a DAG of transformations but does not execute until an action (e.g., `count()`, `write`) is called. This lets Catalyst reorder and combine operations, skip unused work, and read data once. It is why you can chain many transformations cheaply and only pay when you act.

15. **What is the difference between DataFrame, RDD, and SQL in Spark?**
    - RDDs are low-level distributed collections without schema; DataFrames are schema'd and benefit from Catalyst/Tungsten; SQL is the declarative interface compiled into the same optimized plan. Use DataFrames/SQL for nearly everything; use RDDs only for bespoke low-level logic.

16. **Explain the Catalyst optimizer and Tungsten execution engine.**
    - Catalyst is Spark's optimizer: it analyzes, logically optimizes (predicate pushdown, projection pruning), chooses physical plans (join strategies, shuffles), and generates code. Tungsten is the runtime: off-heap binary memory, vectorized processing, and whole-stage code generation to minimize CPU overhead.

### Databricks SQL and DLT

17. **What is a SQL warehouse in Databricks and how is it used?**
    - A SQL warehouse is managed compute purpose-built for SQL and BI. It lets analysts run queries and dashboards against Delta tables concurrently without sharing clusters or notebooks. Classes include Serverless, Pro, and Classic, each balancing cost vs performance/concurrency.

18. **What are Delta Live Tables (DLT)?**
    - DLT is a declarative pipeline framework where you define tables with `@dlt.table` or `CREATE OR REPLACE LIVE TABLE` and DLT handles execution, incremental processing, dependency orchestration, and data quality expectations. It is ideal for building reliable bronze→silver→gold pipelines with built-in validation.

19. **Databricks SQL vs PySpark notebooks — when would you use each?**
    - Use Databricks SQL for declarative reporting, BI dashboards, and simple transformations by analysts. Use PySpark notebooks for complex ETL logic, streaming, ML, and custom code by engineers/data scientists. They read and write the same Delta tables, so teams can mix approaches.

### Governance and Orchestration

20. **What is Unity Catalog and why does it matter?**
    - Unity Catalog is Databricks' centralized governance layer organizing data as metastore → catalog → schema → table. It provides fine-grained grants, data discovery, automated lineage, and audit logging across all workspaces. It matters because it replaces scattered per-cluster metastores with one consistent, compliant security model.

21. **How do you secure data with Unity Catalog? Give an example.**
    - Via SQL grants: `GRANT SELECT ON TABLE sales TO analyst_group`, or column masking / row filters for finer control. Access is enforced regardless of which engine or tool queries the data, and every access is audited.

22. **What are Databricks Jobs and how do you build a workflow with dependencies?**
    - Jobs schedule notebook/Python/JAR/SQL/DLT/dbt tasks. A job can define multiple tasks with a dependency graph so tasks run in parallel or after prerequisites complete, with retries, timeouts, and alerts. Parameters let one job serve many runs (dates, regions, configs).

23. **Databricks Jobs vs Apache Airflow — how would you decide?**
    - Use Databricks Jobs for pure Databricks pipelines with minimal setup and deep integration. Use Airflow when orchestration must span many external systems (databases, Kafka, APIs) and you need rich DAG features — Airflow can trigger Databricks jobs via the `DatabricksSubmitRunOperator` while Databricks handles the heavy compute.

### Architecture and Ingestion

24. **Explain the medallion (bronze/silver/gold) architecture.**
    - Bronze stores raw, immutable ingested data; silver is cleansed, deduplicated, conformed granular data; gold is aggregated, business-ready tables for dashboards/ML. This gives a source of truth (bronze), reliable processing (silver), and fast consumption (gold), plus full reprocessability.

25. **What is Auto Loader and how does it track files?**
    - Auto Loader incrementally ingests files from cloud storage using the `cloudFiles` format. It tracks processed files via checkpoints — either directory listing or cloud file-notification queues — giving exactly-once, resumable ingestion that only loads new files.

26. **When would you use Auto Loader vs a scheduled batch read?**
    - Use Auto Loader for continuously arriving files (log dumps, partner feeds, IoT data) that need incremental, resumable loading into a bronze Delta table. Use a plain batch read for a small number of one-time historical files or when files arrive in bulk on a known schedule.

27. **Describe the compute and storage model of Databricks vs Snowflake.**
    - Both separate compute and storage. Databricks runs compute over your own cloud storage with Spark/Photon and open Delta formats; Snowflake manages its own storage with a proprietary engine optimized for SQL concurrency. Databricks is code-first and strong for ETL/streaming/ML; Snowflake is SQL-first and strong for managed BI warehousing.

28. **What are common mistakes people make with Databricks, and how would you avoid them?**
    - Right-sizing clusters (don't overpay, use autoscaling/job clusters), always using Delta not plain Parquet, partitioning/Z-ordering large tables, avoiding `collect()` on large data, keeping bronze raw, and running regular `OPTIMIZE`/`VACUUM` maintenance.

### Scenario Questions

29. **A dashboard query on a Delta table keeps timing out. What do you check?**
    - Look at table size and file counts (`OPTIMIZE` to compact), ensure filters are on partitioned or Z-ordered columns, verify predicate pushdown is happening, right-size the SQL warehouse (or use Photon), and move heavy aggregations into a gold table rather than scanning silver on every query.

30. **How would you build a production-grade incremental pipeline from cloud storage files to a dashboard?**
    - Use Auto Loader streaming into a bronze Delta table (immutable raw with checkpoints), a silver layer with cleansing, dedup, and schema enforcement, a gold layer with aggregated facts, run it as a scheduled Databricks job or DLT pipeline with data-quality constraints, and expose gold to BI via a SQL warehouse — all governed under Unity Catalog.
