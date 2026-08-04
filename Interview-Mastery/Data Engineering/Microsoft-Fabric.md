# Microsoft Fabric — Complete Interview Notes

Microsoft Fabric is Microsoft's unified Software-as-a-Service (SaaS) data analytics platform. It brings together data movement, data lakes, data engineering, data integration, data science, real-time intelligence, and business intelligence into a single, integrated product. These notes cover the core concepts, workloads, architecture, and common interview scenarios.

---

## What is Microsoft Fabric

### Overview

- **Fabric** is an end-to-end analytics platform announced by Microsoft in **May 2023** (GA in late 2023).
- It is delivered as a **SaaS (Software-as-a-Service)** product — meaning Microsoft manages the infrastructure, scaling, patching, and upgrades for you.
- Fabric unifies **multiple analytics workloads** under one platform so teams do not need to stitch together separate Azure services.
- Everything is built around **OneLake**, a single, unified lakehouse that acts as the "data foundation" for every workload in Fabric.
- Because it is SaaS, there is **no infrastructure to provision** (no clusters, no VMs, no storage accounts to manage manually).
- Fabric is accessible from the **Fabric portal** (app.fabric.microsoft.com) and is tied to a Power BI tenant.

### Built on Azure Technologies

- Fabric is not a brand-new engine for everything — it **reuses proven Azure components**:
  - **Azure Synapse Analytics** — for data warehousing (Spark, SQL)
  - **Power BI** — for semantic models and reporting
  - **Azure Data Factory (ADF)** — for data integration and pipelines
  - **Azure Data Explorer (ADX)** — for real-time/KQL analytics
- This means skills from Synapse, ADF, and Power BI **transfer directly** into Fabric.
- Example: A Fabric **Data Factory pipeline** is essentially the same concept as an **Azure Data Factory pipeline**; a Fabric **Warehouse** is similar to a **Synapse dedicated SQL pool** but serverless and SaaS-managed.

### One Security Model, One Governance Model

- Fabric applies **one security model** across all workloads — define access once at the workspace/OneLake level, and it applies everywhere.
- **One governance model** is enforced through **Microsoft Purview** integration — sensitivity labels, data loss prevention, and lineage all flow through the same control plane.
- This removes the "security sprawl" problem where each separate Azure service needed its own separate security setup.

### Fabric vs. Separate Azure Services

| Aspect | Separate Azure Services | Microsoft Fabric |
| --- | --- | --- |
| Provisioning | Manual per service (storage, compute, workspace) | Single capacity, all workloads enabled |
| Data sharing | Copy/E-export between services | OneLake shared store, no copies |
| Security | Per-service configs | One model, one governance layer |
| Billing | Per-service consumption | One F SKU capacity |
| Integration | Custom glue code required | Native, built-in |

- The main value proposition: **one place to store, one place to secure, one place to pay.**

---

## Fabric Workloads / Experiences

### The Workload Panes

- The Fabric portal shows workloads as **left-hand navigation items**. Each workload is a set of integrated tools within the same SaaS platform.

### Lakehouse

- A **Lakehouse** is an architecture where a data lake (files) and a data warehouse (structured tables) coexist.
- In Fabric, the Lakehouse stores **Delta tables** in OneLake and exposes them via a **SQL endpoint**.
- It is the default choice for **data engineering** work — raw files, bronze/silver/gold medallion layers.

### Warehouse

- The **Fabric Warehouse** is a fully-managed, T-SQL-native data warehouse.
- It is the right place for **SQL-centric workloads**, existing T-SQL code, and governed, dimensional models (star schemas).
- It shares OneLake storage with Lakehouse but has its **own compute** (SQL engine).

### Data Factory (Pipelines)

- Fabric's **Data Factory** provides **pipelines** for data movement and orchestration.
- Includes **Copy activity**, **data flows** (no-code transformation), and orchestration of notebooks, stored procedures, and other activities.

### Data Engineering (Spark)

- The Data Engineering experience provides **Notebooks** powered by **Apache Spark** (PySpark, Scala, Spark SQL).
- Used for **data transformation, ETL/ELT**, and machine learning feature engineering.

### Data Science (Machine Learning)

- Fabric Data Science provides **MLflow**-integrated experiment tracking, model training, and registration.
- Supports **Python**, **AutoML**, and integration with **Azure Machine Learning**.

### Power BI (Reporting)

- Power BI is **natively built into Fabric** — semantic models and reports live side-by-side with your data.
- Because the semantic model can connect **directly to OneLake tables** (Direct Lake mode), there is no need to copy data into an import model.
- **Direct Lake** mode loads Delta parquet files into memory, giving near import-mode performance without duplication.

### Real-Time Intelligence

- The **Real-Time Intelligence** workload handles **streaming data**.
- Includes **Event Streams** (ingestion), **KQL databases** (Azure Data Explorer in Fabric), and **reflexes** (automated responses to conditions).
- Example use case: IoT telemetry, app logs, financial tick data.

### Data Activator

- **Data Activator** is a no-code tool to build **alerts and automated actions** based on data conditions.
- You define **triggers** (e.g., "when order volume drops 20%") and **actions** (send Teams message, start a pipeline, send email).
- It monitors data in real time without needing to write code.
- Example: A retailer monitors inventory levels and triggers a Teams alert + Power Automate flow when stock for a best-selling SKU runs low.

### OneSecurity and Purview Integration

- **OneSecurity** in Fabric enforces a single security model across storage, compute, and reporting.
- Microsoft **Purview** integration provides:
  - Sensitivity labels on columns/tables
  - Data Loss Prevention (DLP) policies
  - **Data lineage** tracking across pipelines, notebooks, and reports
- Governance is therefore applied once and inherited everywhere.

---

## OneLake

### What is OneLake

- **OneLake** is a single, unified, logical data lake that comes with every Fabric tenant.
- Every Fabric workspace automatically gets a **OneLake container** — you never provision storage.
- It provides the **"one copy of data, consumed by all workloads"** model.
- OneLake is the "database" for the platform: Lakehouse, Warehouse, KQL, and Power BI all read from the same underlying storage.
- Under the hood it is built on **Azure Data Lake Storage (ADLS) Gen2** but abstracts it away.

### Universal Data Store

- All Fabric workloads read and write to OneLake **by default**:
  - Spark notebooks write Delta tables to OneLake.
  - Warehouses store tables in OneLake (as Delta files).
  - Power BI semantic models reference OneLake tables directly.
  - KQL databases store their data in OneLake as well.
- This means **no data movement between workloads** — the same table is shared.

### Data Stored Once, Accessed by All

- Because data lives in OneLake once, there is a single source of truth.
- Example: You create a `silver_sales` table in a Lakehouse. The Warehouse can query it, Power BI can build a Direct Lake model on it, and a Spark notebook can transform it further — **all without copying data**.
- This directly attacks the "data silos" and "copy proliferation" problem.

### Shortcuts (No Data Copy)

- **Shortcuts** are logical pointers to data stored **elsewhere** — either external (S3, ADLS Gen2) or in another Fabric workspace.
- Shortcuts make the external data appear **as if it is inside OneLake**, without moving or copying it.
- You can query shortcut data with Spark, SQL, and Power BI just like native tables.
- Example: Point a shortcut at an ADLS Gen2 folder of parquet files and query it with a Warehouse SQL endpoint immediately.

### Mirroring (No ETL Copy for Data Sources)

- **Mirroring** continuously replicates data from source databases (Azure SQL, Snowflake, Cosmos DB, etc.) into OneLake as **Delta tables**, automatically.
- No ETL/ELT code is written — you just point Fabric at the source and it keeps data in sync.
- Mirrored data is queryable and governed like any OneLake data.
- Example: Mirror an Azure SQL database. The tables appear in OneLake and update automatically, then you can join them with other data.

### File Formats: Delta (Default) and Parquet

- **Delta Lake** is the **default** format in OneLake — all tables are Delta tables (transaction log + Parquet data files).
- Delta provides **ACID transactions**, time travel, and schema evolution.
- Underlying files are **Parquet** (columnar, compressed).
- OneLake also stores generic files (CSV, JSON, images) alongside Delta tables for raw landing data.

### Folder Structure

- OneLake paths follow the convention: `https://onelake.dfs.fabric.microsoft.com/<workspace>/<item>/...`
- A typical Lakehouse folder structure inside OneLake:

```
Lakehouse "SalesLH"
├── Tables/                      # managed Delta tables (bronze/silver/gold)
│   ├── bronze_orders/
│   ├── silver_orders/
│   └── gold_customer_orders/
├── Files/                       # raw files (CSV, JSON, parquet, images)
│   ├── raw/orders_2026-01-01.csv
│   └── land/ingest_log.json
└── _delta_log/                  # delta transaction logs (hidden)
```

- The `Tables` area holds managed Delta tables; the `Files` area holds unstructured/raw data.
- This structure is **identical across workspaces**, so paths are predictable and portable.

---

## Fabric Lakehouse

### What is a Fabric Lakehouse

- A Fabric **Lakehouse** combines a **data lake** (files) with **structured Delta tables**, all in OneLake.
- It gives you the flexibility of a data lake and the reliability of a warehouse.
- Analogy: a Lakehouse is "**a lake that acts like a warehouse**" — files for raw data, Delta tables for curated data.

### Tables vs. Files

| Concept | Description |
| --- | --- |
| **Tables** | Managed **Delta tables** stored in the `Tables` area. They have a schema, support ACID, and can be queried via SQL or Spark. |
| **Files** | Raw/unmanaged files stored in the `Files` area. No schema enforcement — used for landing data, images, JSON, etc. |

- You can **convert Files into Tables** by loading them with a notebook or using the "Load to Tables" option in the UI.
- Example: Drop `orders.csv` into Files → run a PySpark notebook → write `bronze_orders` Delta table → now it is queryable as a table.

### Tables: Managed Delta Tables

- Managed tables in a Lakehouse are **Delta tables**:
  - **ACID transactions** (atomic, consistent, isolated, durable)
  - **Time travel** (`VERSION AS OF`)
  - **Schema evolution / enforcement**
  - **Upserts / MERGE** via Delta operations
- These tables can be read by:
  - Spark (PySpark / Spark SQL)
  - The **SQL analytics endpoint** (T-SQL)
  - Power BI **Direct Lake** semantic models
- Example (PySpark):

```python
df = spark.read.csv("Files/raw/orders.csv", header=True)
df.write.mode("overwrite").format("delta").saveAsTable("bronze_orders")
```

### Files: Raw Files in the Lakehouse

- The `Files` area stores raw data without schema management.
- You can upload files directly in the portal, or land them via pipeline Copy activity.
- Files are accessible to Spark as a path: `abfss://.../Files/raw/...` or relative `Files/raw/...`.
- Example: read a raw JSON file in Spark:

```python
df = spark.read.json("Files/raw/events.json")
```

### Creating a Lakehouse in the Fabric Portal

1. Open the **Fabric portal** → switch to **Data Engineering** workload.
2. Click **New → Lakehouse**.
3. Name it (e.g., `SalesLakehouse`) and choose a **workspace**.
4. Click **Create** — a Lakehouse item appears with `Tables` and `Files` areas.
5. Start loading data:
   - Use **Upload files** for small manual loads.
   - Use a **Notebook** or **Pipeline** for automated ingestion.
   - Use **Get Data** for connectors.
6. Once tables exist, you can explore them, query via SQL, or build Power BI reports.

### SQL Endpoint and Semantic Model

- Every Lakehouse automatically exposes:
  - A **SQL analytics endpoint** — a T-SQL endpoint you can query from SSMS, Azure Data Studio, or `sqlcmd`.
  - A **default semantic model** — Power BI can connect to the Lakehouse tables directly.
- The SQL endpoint is **read-only** for the Lakehouse (write via Spark or warehouse).
- With **Direct Lake**, Power BI loads Delta files straight into memory — no import, no copy, near-instant.

---

## Fabric Warehouse

### What is a Fabric Warehouse

- The Fabric **Warehouse** is a fully-managed, distributed SQL data warehouse in SaaS form.
- It is **T-SQL native** — you can run DDL (`CREATE TABLE`), DML (`INSERT/UPDATE/DELETE`), stored procedures, and views.
- Data stored in a Warehouse lives in **OneLake as Delta tables** too, so it is still shared and portable.
- It replaces the old "Synapse dedicated SQL pool" model with a serverless, auto-scaling SaaS experience.

### T-SQL Support (DDL, DML)

- Full **T-SQL** surface for both DDL and DML:

```sql
-- DDL
CREATE TABLE dbo.dim_customer (
    customer_id    INT PRIMARY KEY,
    customer_name  VARCHAR(200),
    region         VARCHAR(50)
);

-- DML
INSERT INTO dbo.dim_customer VALUES (1, 'Acme Corp', 'West');

UPDATE dbo.dim_customer
SET region = 'NorthWest'
WHERE customer_id = 1;

-- Query
SELECT region, COUNT(*)
FROM dbo.dim_customer
GROUP BY region;
```

- Supports **stored procedures**, **views**, and **temporary tables**.
- Great for migrating **existing SQL Server / Synapse code** into Fabric.

### Compute Isolation from Lakehouse

- The Warehouse has its **own compute** (SQL engine), separate from Lakehouse/Spark compute.
- Both read/write the **same OneLake Delta files**, but each workload bills its own compute.
- This means:
  - SQL-heavy teams do not consume Spark CUs.
  - Spark teams do not consume Warehouse CUs.
  - Data is **not duplicated** — just different engines over the same files.
- Example: A Spark notebook writes `gold_sales` to OneLake; a Warehouse SQL query reads the same Delta files without touching Spark compute.

### Warehouse vs. Lakehouse in Fabric

| Feature | Lakehouse | Warehouse |
| --- | --- | --- |
| Primary users | Data engineers (Spark) | Data engineers / analysts (SQL) |
| Language | PySpark, Scala, Spark SQL | T-SQL |
| Write path | Spark / external tools | T-SQL DML |
| Best for | Medallion ETL, raw-to-curated, ML | Governed dimensional models, legacy T-SQL, ad-hoc SQL |
| Files area | Yes (raw files) | No |
| SQL endpoint | Read-only | Full read/write |
| Compute | Spark | SQL (own capacity usage) |

- **Rule of thumb**: use **Lakehouse** when you are building ETL/data engineering pipelines with Spark; use **Warehouse** when you want SQL-centric modeling, governance, and reporting over curated data.

### Data Movement Between Them via Shortcuts

- Lakehouse and Warehouse can reference each other's tables via **shortcuts** — no copy needed.
- Example: In a Warehouse, create a shortcut to a `bronze_orders` table in a Lakehouse, then build a `gold_sales` table with T-SQL that reads the shortcut.
- This gives you the **best of both**: Spark for heavy ETL, T-SQL for modeling, zero duplication.

```sql
-- In the Warehouse, after creating a shortcut to the Lakehouse table:
CREATE VIEW dbo.v_bronze_orders AS
SELECT * FROM bronze_orders;   -- bronze_orders is a shortcut
```

---

## Data Factory Pipelines

### What are Fabric Pipelines

- **Pipelines** in Fabric are the data integration/orchestration tool (inherited from **Azure Data Factory**).
- They **move and transform data** between source and destination — both inside and outside Fabric.
- Used to automate **scheduled, repeatable** data workloads.
- Example pipeline: `Copy raw sales CSV from ADLS → Load into Lakehouse bronze table → Run notebook to build silver → Run stored proc in Warehouse to build gold`.

### Copy Activity

- The **Copy activity** copies data from a source to a sink with minimal code.
- Supports **hundreds of connectors**: SQL Server, Azure SQL, Snowflake, S3, ADLS Gen2, SharePoint, Salesforce, REST, etc.
- Configuration includes source, sink, and mappings (column mapping, schema handling).
- Under the hood it uses the **data movement service** to parallelize and optimize transfers.
- Example: Copy an entire folder of parquet from an S3 bucket into the Lakehouse `Files/raw` area.

### Data Flows (No-Code Transformations)

- **Data flows** give a **no-code / low-code** transformation experience similar to **Power Query**.
- You build transformation steps visually: joins, filters, aggregations, pivot/unpivot, derived columns.
- Data flows use a **Spark-based compute** engine at run time.
- Great for **business users** who want to transform data without writing code.
- Example: a data flow that joins `customers` and `orders`, filters to the last 90 days, and writes the result to a gold table.

### Orchestrating Notebooks and Stored Procedures

- Pipelines are **orchestrators** — they can call other Fabric items:
  - **Notebook activity** — run a Spark notebook and pass parameters.
  - **Stored procedure activity** — execute T-SQL in a Warehouse.
  - **Data flow activity**, **Copy activity**, **KQL activity**, **Delete activity**, etc.
- Activities can be chained with **conditional logic** (If, ForEach, Until) and dependencies.
- Example:

```
[Copy: ADLS -> Lakehouse bronze]
        |
        v
[Notebook: Build silver table]
        |
        v
[Stored Proc: Load gold dimension table]
        |
        v
[Send success email / trigger Data Activator]
```

### Integration with Data Engineering Notebooks

- Pipelines pass **parameters** to notebooks (e.g., `@pipeline().parameters.RunDate`).
- The notebook can read pipeline parameters via widgets, making pipelines reusable across dates.
- This is the classic **parameterized ETL** pattern in Fabric.

### Scheduling

- Pipelines can be triggered:
  - **On a schedule** (cron-like, e.g., hourly, daily at 2 AM)
  - **On data arrival** (event-driven)
  - **Manually**
- Schedule timezone and retry policies are configurable.
- Example: run the nightly ELT at `00:30 UTC` Monday–Friday with 3 retries on failure.

---

## Data Engineering in Fabric (Spark)

### Fabric Notebooks

- **Notebooks** are the primary development interface for Fabric data engineering.
- Built on **Apache Spark** — you write code in **PySpark**, **Scala**, **Spark SQL**, or **R**.
- Cells can mix languages, and you can add **Markdown** cells for documentation.
- Notebooks are stored as Fabric items and can be run **on demand**, from **pipelines**, or **scheduled**.
- Example notebook:

```python
from pyspark.sql import functions as F

orders = spark.read.format("csv") \
    .option("header", True) \
    .load("Files/raw/orders.csv")

result = orders.filter(F.col("status") == "COMPLETED") \
    .groupBy("region") \
    .agg(F.sum("amount").alias("total_amount"))

result.write.format("delta").mode("overwrite").saveAsTable("silver_orders")
```

### Spark Jobs and Sessions

- Each notebook uses a **Spark session** on a **Spark pool**.
- The session stays alive for a configured **time-to-live (idle timeout)** so consecutive cells reuse compute.
- Spark **jobs** are the actual computation units executed by the session.
- Session types:
  - **Starter pool** — quick startup, good for small jobs.
  - **Custom pools** — defined by the admin with specific node sizes, autoscale, and runtime.
- Idle sessions that time out release compute (important for **cost control**).

### Selecting the Spark Runtime

- Fabric lets you pick a **Spark runtime version** (e.g., runtime 1.2 / 1.3 which include Spark 3.5.x, Delta, and ML libraries).
- Runtime choice affects library compatibility and features.
- Administrators set **default runtime** at capacity level; notebooks can override it.
- Example: if you need the latest Delta features, select the newest runtime; if your team standardized on Spark 3.4, pin that version.

### Compute for Notebooks

- Notebook compute is **billed to the Fabric capacity** in **Capacity Units (CUs)** based on vCore-hours used.
- You can configure:
  - Number of **executors / nodes**
  - **Autoscale** on/off
  - Session **idle timeout**
- Larger compute = faster jobs but more CU consumption; right-sizing is key to cost management.
- Example: a nightly job with 500 GB of input might use 8 medium nodes, while a small ad-hoc notebook uses the starter pool.

---

## Mirroring

### What is Fabric Mirroring

- **Mirroring** automatically replicates data from an external database into OneLake as **Delta tables**.
- It is **continuous** (near real-time) and **no-code** — you do not write ETL.
- The mirrored tables in OneLake are **queryable** by all Fabric workloads (Spark, SQL, Power BI).
- You keep the source as the **system of record**; Fabric keeps a governed copy ready for analytics.

### Supported Sources

- **Azure SQL Database**
- **Azure SQL Managed Instance**
- **Azure Cosmos DB**
- **Snowflake**
- **Azure Databricks (Unity Catalog)**
- More sources added over time (e.g., SQL Server on-prem via gateway, Fabric databases).
- Example: Mirror a Snowflake warehouse so all downstream Fabric analytics run on the OneLake copy, freeing Snowflake compute.

### How Mirroring Differs from Pipelines

| Aspect | Pipelines | Mirroring |
| --- | --- | --- |
| Mechanism | Pull data on a schedule (batch) | Continuous, incremental replication |
| Code | Copy activity / notebooks / data flows | None — configure and go |
| Latency | Depends on schedule (minutes–hours) | Near real-time (seconds–minutes) |
| Workload | Any transformation or movement | Pure replication of source tables |
| Compute | Consumes pipeline/Spark CUs on each run | Small continuous sync compute |

- **Use pipelines** when you need transformations, joins, aggregations, or complex mapping.
- **Use mirroring** when you just need source tables available in OneLake, current and continuously updated.

### Cost Savings

- No **transformation code** to write or maintain → lower engineering effort.
- Offloads read workload from the source warehouse (e.g., Snowflake) to the mirrored copy → **reduces source compute cost**.
- Data is stored once in OneLake and shared by all workloads → **no multiple copies**.
- Example: analytics dashboards query the mirrored copy instead of hammering the source Snowflake account with repeated full-table scans.

---

## Shortcuts

### What are Shortcuts

- A **shortcut** is a logical pointer in OneLake that references data stored **elsewhere**.
- The data appears inside OneLake **as if it were local**, but **no data is copied or moved**.
- You can create shortcuts to:
  - **Amazon S3**
  - **Azure Data Lake Storage (ADLS) Gen2**
  - **Another OneLake location** (another workspace/tenant)
  - **Dataverse**
  - **Google Cloud Storage** (with connectors)
- Shortcuts work for both **tables** and **files/folders**.

### Why Shortcuts Avoid Data Duplication

- Because a shortcut is a pointer, the data stays in its original location.
- Benefits:
  - **No storage duplication** → saves OneLake storage costs.
  - **No copy latency** → data is always current.
  - **Single source of truth** → changes upstream are immediately visible.
- Example: your org already stores customer data in ADLS Gen2. Create a shortcut instead of copying — teams get governed access via OneLake without a second copy.

### Shortcut Types

| Shortcut type | Source | Use case |
| --- | --- | --- |
| **Amazon S3** | S3 bucket (path + access key/ARN) | Bring AWS data into Fabric without moving it |
| **ADLS Gen2** | Azure storage container/path | Connect existing enterprise data lake |
| **OneLake** | Another workspace/tenant | Share tables across teams/tenants without copy |
| **Dataverse** | Power Platform data | Expose Dataverse tables for analytics |
| **GCS** | Google Cloud Storage | Multi-cloud data access |

### OneLake Shortcuts Across Workspaces

- You can shortcut a table from **Workspace A** into **Workspace B**.
- This enables **data sharing** without copy and without giving direct access to the source workspace.
- Permissions are still enforced (the shortcut inherits/requires access to the underlying item).
- Example: the Finance team publishes `gold_financials` in their workspace; the Sales team creates a OneLake shortcut to it and reports on it — no data was moved.

---

## Fabric vs Databricks vs Snowflake

### Positioning

| Platform | Positioning |
| --- | --- |
| **Microsoft Fabric** | Unified **SaaS** analytics platform: lake, warehouse, integration, BI, real-time all in one |
| **Databricks** | **Lakehouse** platform focused on data engineering, data science, and ML on open formats (Delta) |
| **Snowflake** | **Data cloud / warehouse** optimized for SQL analytics and secure data sharing |

- Fabric = "everything together, SaaS, Microsoft-centric".
- Databricks = "lakehouse + AI/ML first, best-in-class Spark performance".
- Snowflake = "SQL-first cloud warehouse with separation of storage and compute, great sharing".

### Key Differences

- **Compute vs. storage**:
  - Snowflake famously separates storage and compute (scale each independently).
  - Fabric unifies storage (OneLake) and provides multiple engines over it.
  - Databricks uses its own optimized Spark engine over cloud storage.
- **Format**:
  - Databricks invented **Delta Lake**; Fabric adopted **Delta** as its native format — so they are interoperable at the file level.
  - Snowflake stores data in its own **proprietary micro-partition format** (unless using Iceberg/open catalog).
- **Language**:
  - Snowflake: SQL-centric (also Python via Snowpark).
  - Databricks: Python/Scala/SQL on Spark.
  - Fabric: PySpark, SQL, Power Query, and no-code dataflows.

### Fabric's Integration with Power BI and Microsoft 365

- **Power BI is built in** — no separate licensing/model connection; Direct Lake mode eliminates duplicate data.
- Deep integration with **Microsoft 365**: Microsoft Teams (alerts, sharing), Excel, Outlook, SharePoint, and **Copilot** across the platform.
- Example: a Power BI report in Fabric can push an alert through **Data Activator** into Teams, all within one product.
- This end-to-end Microsoft story is Fabric's strongest differentiator.

### When to Choose Fabric

Choose **Fabric** when:
- You are already a **Microsoft shop** (Entra ID, Power BI, M365).
- You want **one platform** for ingestion, lakehouse, warehouse, and BI (no stitching).
- You need **SaaS simplicity** (no infrastructure management).
- You value **one security/governance model** and native Purview integration.

Choose **Databricks** when:
- Your workloads are **ML/AI-heavy** and need best-in-class Spark performance.
- You need fine-grained control over clusters and libraries.
- You prefer the **open lakehouse** ecosystem with multi-cloud neutrality.

Choose **Snowflake** when:
- You want the best **SQL warehouse performance and data sharing**.
- You have mature SQL analysts and little desire for Spark/ML in the platform.
- You need strict **separation of compute and storage**.

---

## Real-Time Intelligence

### What is Real-Time Intelligence in Fabric

- The **Real-Time Intelligence** workload handles **streaming and time-based** data analysis.
- It is Fabric's version of **Azure Data Explorer (ADX)**.
- Includes:
  - **Event Streams** — ingest data from sources like Event Hubs, IoT Hub, Kafka, and custom apps.
  - **KQL databases** — high-performance columnar stores optimized for time-series queries.
  - **Reflexes** — auto-respond to data conditions with alerts/actions.
  - **Real-Time dashboards** — KQL-based visualizations.

### Event Streams (Kafka-like)

- **Event Streams** is Fabric's ingestion service for real-time data (managed Kafka/Event Hubs concept).
- Sources: **Azure Event Hubs**, **Azure IoT Hub**, **Kafka**, **custom applications** (REST), **Azure Blob/ADLS** (as a source), and more.
- Data flows into a **stream** and can be routed to:
  - A **KQL database**
  - A **Lakehouse** (for batch/history)
  - Power BI (for live dashboards)
  - **Reflexes** for alerting
- Example: IoT temperature sensors publish to Event Hubs → Event Stream routes to a KQL database for real-time anomaly detection.

### KQL (Kusto Query Language) Databases

- **KQL databases** are columnar, append-only data stores optimized for **time-series and log analytics**.
- Queried with **KQL** (not T-SQL).
- Example KQL query:

```kql
TempReadings
| where TimeStamp > ago(1h)
| summarize avg(Temperature) by DeviceID, bin(TimeStamp, 5m)
| render timechart
```

- KQL databases can also **export to OneLake**, so results/raw data can be shared with Lakehouse workloads.
- Great for: IoT telemetry, application logs, security events, financial tick data.

### Querying Streaming Data

- Real-time data can be queried **as it arrives** using KQL.
- **Reflexes** can evaluate conditions on the stream and trigger **Data Activator** actions (Teams alert, Power Automate flow, start a pipeline).
- Example: a reflex fires "page on-call engineer" when error rate exceeds 5% over a 5-minute window — no batch job needed.

---

## Security and Governance

### One Security Model (Workspace-Level)

- Fabric applies security at multiple levels:
  - **Tenant** (global admin policies)
  - **Capacity** (admin, contributors)
  - **Workspace** (roles: Admin, Member, Contributor, Viewer)
  - **Item/table/row/column** granularity
- Roles example:
  - **Admin**: full control including security settings.
  - **Member**: read/write, share, but no admin settings.
  - **Contributor**: create/edit items, cannot share.
  - **Viewer**: read-only.
- The same workspace roles govern **all workloads** — no per-service security setup.

### Row-Level Security

- **Row-level security (RLS)** restricts which rows a user can see in a table.
- In a Warehouse you define RLS with **T-SQL security predicates**:

```sql
CREATE SECURITY POLICY SalesSecurity
ADD FILTER PREDICATE dbo.fn_RegionFilter(Region)
ON dbo.sales
WITH (STATE = ON);
```

- In Power BI, RLS uses **roles** and DAX filters.
- RLS in the warehouse is enforced at the **SQL query layer** — even direct SQL queries respect it.
- Example: a national sales manager only sees rows where `Region = 'West'`.

### Purview Integration for Governance

- **Microsoft Purview** provides the governance backbone:
  - **Data catalog** — searchable inventory of Fabric data assets.
  - **Sensitivity labels** (e.g., Confidential) inherited down to columns.
  - **Data Loss Prevention (DLP)** policies that block risky exports.
  - **Data lineage** — see where data came from and where it flows.
- Labels created in Purview show up automatically on Fabric items and query results.

### Data Lineage in Fabric

- Fabric shows **lineage views** per item: which pipelines feed a table, which reports consume it, etc.
- Lineage helps:
  - Impact analysis (what breaks if I change this column?)
  - Compliance audits
  - Understanding data provenance
- Example: opening a Lakehouse's lineage view shows `ADLS → Copy activity → bronze_orders → silver_orders → Power BI report`.

### Microsoft Entra ID Integration

- **Microsoft Entra ID** (formerly Azure AD) is the identity provider for Fabric.
- Access via **Single Sign-On (SSO)** — users sign in once.
- Fabric supports **service principals** and **managed identities** for automated pipeline auth.
- Example: a pipeline authenticates to ADLS Gen2 using the capacity's **managed identity**, avoiding hard-coded secrets.

---

## Capacity and Pricing

### Fabric Capacity (F SKUs)

- Fabric runs on **Fabric capacity**, bought as **F SKUs** (F2, F4, F8, ..., F64, F128, F512, F1024, F2048).
- **F64+** unlocks the full Fabric experiences (including **free Power BI** for report consumers via capacity).
- Smaller SKUs (F2–F32) are for development; larger for production.
- Capacity can be purchased via **Azure portal** (pay-as-you-go) or **Microsoft 365** (reserved).

| SKU | Capacity Units (CUs) | Typical use |
| --- | --- | --- |
| F2 | 2 CU | Dev/test, single user |
| F8 | 8 CU | Small team workloads |
| F32 | 32 CU | Production data pipelines |
| F64 | 64 CU | Full production; enables Power BI in capacity |
| F1024+ | 1024 CU | Enterprise scale |

### How Capacity is Consumed (CU = Capacity Units)

- Every operation (Spark job, SQL query, pipeline run, Power BI query) consumes **Capacity Units (CUs)**.
- **1 CU ≈ 1 vCore** (plus memory) for a second.
- Capacity is a **pool of CUs** shared across all workloads in that capacity — one huge Spark job can consume most of the pool momentarily.
- **Throttling**: if demand exceeds the pool, Fabric queues/throttles jobs until CUs free up.
- The **Capacity Metrics app** (built-in Power BI report) shows CU consumption by item and identifies hotspots.

### Workspace Assigned to Capacity

- Each workspace runs **on a capacity** (default: the "Shared capacity" tier with limited features).
- Assigning a workspace to a paid capacity enables the **full Fabric experience**.
- You can assign different workspaces to different capacities to isolate production from dev.
- Example: `Prod-Workspace` → F128 capacity; `Dev-Workspace` → F8 capacity.

### Pause/Resume Capacity

- Fabric supports **pausing a capacity** in the Azure portal — stops billing and all workloads in that capacity.
- Resuming brings everything back.
- **Limitations**: not available on all SKUs/purchase types (e.g., reserved capacity via M365 has limits); paused capacity can affect scheduled pipelines and Power BI.
- Best practice: pause dev/test capacities overnight or on weekends to **save cost**.

---

## Common Mistakes

### Not Understanding OneLake and Creating Duplicates

- Teams often copy the same data into Lakehouse, Warehouse, and Power BI **because they assume separate storage**.
- In Fabric they all share **OneLake** — one copy serves all.
- Mistake: building multiple copies "for safety" — you pay storage twice and risk divergence.
- Fix: keep one canonical table in OneLake and use **shortcuts** and **Direct Lake** to reference it.

### Using Warehouse for Everything Instead of Lakehouse

- The Warehouse is convenient for SQL people, but it is **not** the best place for raw ingestion, files, or Spark ETL.
- Mistake: trying to land raw files / run data engineering in a Warehouse.
- Fix: use a **Lakehouse** for the medallion ETL layers; use the **Warehouse** for the governed, SQL-centric final layer (or for teams that only do SQL).

### Ignoring Shortcuts (Copying Data Unnecessarily)

- Teams sometimes copy data from ADLS/S3/other workspaces when a **shortcut** would do.
- Mistake: scheduled copy jobs that duplicate terabytes needlessly → storage and compute waste, stale copies.
- Fix: prefer shortcuts for external/existing data; use Copy only when you truly need a snapshot or transformation.

### Over-Allocating Capacity (Costs)

- Buying a huge F SKU "to be safe" wastes money if your real peak is far lower.
- Mistake: F512 capacity running a few small notebooks.
- Fix: **right-size the SKU**, monitor the **Capacity Metrics app**, use autoscale/autopilot where available, and pause dev capacities.

### Mixing Real-Time and Batch Workloads Incorrectly

- Real-time streaming workloads (Event Streams + KQL) and batch ETL (pipelines + Spark) have **very different patterns**.
- Mistake: trying to process true streaming data with scheduled batch jobs (adds latency, misses real-time alerting) or using streaming tools for large batch backfills (wastes compute).
- Fix: use **Real-Time Intelligence** for streaming and alerting; use **pipelines + Lakehouse** for batch; combine them cleanly (stream → KQL → export to OneLake for batch analytics).

### Other Frequent Mistakes

- **Not using parameterized pipelines** — hard-coding dates in notebooks instead of passing pipeline parameters.
- **Ignoring Delta features** — not enabling MERGE/time travel where they would simplify upserts.
- **No medallion structure** — a single table dump with no bronze/silver/gold layering.
- **Forgetting lineage/governance** — shipping tables with no sensitivity labels or documentation.
- **Letting idle Spark sessions run** — forgetting idle timeouts → wasted CU spend.

---

## Interview Questions (15+)

### Fabric vs Databricks vs Snowflake

**Q1. What is Microsoft Fabric, and how does it compare to Databricks and Snowflake?**
Fabric is a unified **SaaS** analytics platform covering ingestion (Data Factory), lakehouse/warehouse (OneLake), data engineering (Spark notebooks), data science, real-time intelligence (KQL), and Power BI — all with one security and governance model. Databricks is a lakehouse platform focused on Spark performance and ML. Snowflake is a SQL-first cloud warehouse famous for separation of storage/compute and data sharing. Fabric differentiates by being all-in-one, Microsoft-centric (Entra, Power BI, M365), and SaaS-managed.

**Q2. When would you choose Fabric over Databricks or Snowflake?**
Choose Fabric when you are a Microsoft shop (Power BI + M365 + Entra), want one platform and one bill, need SaaS simplicity, and value the built-in Power BI Direct Lake integration. Choose Databricks for heavy ML/Spark workloads with fine cluster control; Snowflake for mature SQL-only analytics and best-in-class data sharing.

**Q3. How does Fabric's file format interoperate with Databricks?**
Both use **Delta Lake** tables over Parquet. A Delta table written in Fabric's OneLake can be read by Databricks (and vice versa) because Delta is an open format. This file-level interoperability is a big practical advantage of Fabric's Delta-native design.

### OneLake

**Q4. What is OneLake, and why is it important?**
OneLake is the single, logical data lake that ships with every Fabric tenant. Every workspace gets storage automatically. All workloads (Spark, SQL, KQL, Power BI) read/write the same Delta tables there, so data is stored once and consumed everywhere — eliminating copy proliferation and giving a single source of truth.

**Q5. How does OneLake store data, and what is its folder structure?**
OneLake stores **Delta tables** (transaction log + Parquet files) by default, plus raw files. Paths look like `https://onelake.dfs.fabric.microsoft.com/<workspace>/<item>/...`. A Lakehouse has a `Tables` area (managed Delta tables) and a `Files` area (raw CSV/JSON/parquet). This structure is consistent across all workspaces.

### Lakehouse vs Warehouse

**Q6. What is the difference between a Fabric Lakehouse and a Fabric Warehouse?**
A **Lakehouse** combines files + Delta tables, is driven by **Spark**, and is the home of ETL/medallion work; its SQL endpoint is read-only. A **Warehouse** is **T-SQL-native** (full DDL/DML, stored procedures) with its own SQL compute, best for governed dimensional models and migrating existing T-SQL code. They share OneLake, so they work on the same Delta files.

**Q7. Can a Lakehouse and a Warehouse access the same data?**
Yes — through **OneLake** and **shortcuts**. A Warehouse can create a shortcut to a Lakehouse table and query it, and vice versa, without copying data.

**Q8. What is the medallion architecture, and how is it implemented in Fabric?**
Medallion is a layered design: **bronze** (raw as-landed data), **silver** (cleaned/validated, conformed), and **gold** (business-ready, aggregated for reporting). In Fabric, use a **Lakehouse** with Spark notebooks/pipelines to build each layer as Delta tables, then expose **gold** tables to Power BI via Direct Lake or to a **Warehouse** for SQL modeling.

### Shortcuts

**Q9. What is a Fabric shortcut? Give an example.**
A shortcut is a **logical pointer** to data stored elsewhere (S3, ADLS Gen2, another OneLake workspace, Dataverse) that appears inside OneLake as if local — **no data is copied**. Example: instead of copying a 10 TB ADLS folder into Fabric, create an ADLS shortcut; Spark and SQL can query it immediately and see live updates.

**Q10. Why would you use a shortcut instead of a Copy activity?**
When you only need to *access* external data without a snapshot or transformation. Shortcuts avoid storage duplication, avoid copy latency, and always reflect the source. Use Copy activity when you need to transform, change formats, or take a stable snapshot.

### Mirroring

**Q11. What is Fabric mirroring, and how is it different from a pipeline?**
Mirroring **continuously replicates** source database tables (Azure SQL, Snowflake, Cosmos DB, etc.) into OneLake as Delta tables with **no ETL code**. Pipelines are batch pulls that run on a schedule and can do arbitrary transformations. Mirroring is near real-time, continuous, and code-free; pipelines are scheduled and transformation-oriented.

**Q12. What are the benefits of mirroring?**
No transformation code to write/maintain, near real-time availability of source data, and reduced read load on the source system (analytics run on the OneLake copy). You also get governed, shared access to the mirrored data across all Fabric workloads.

### Pipelines

**Q13. What can a Fabric Data Factory pipeline do?**
Move data (Copy activity), transform without code (data flows, Power Query-like), and orchestrate other items — notebooks, stored procedures, data flows, KQL activities — with conditional logic and scheduling. Example: nightly pipeline that copies from ADLS to bronze, runs a notebook to build silver, then calls a stored proc to build gold.

**Q14. How do you pass parameters to a Spark notebook from a pipeline?**
The pipeline's **Notebook activity** passes parameters (e.g., `@pipeline().parameters.RunDate`) into the notebook, which reads them via **widgets** (`dbutils.widgets.get("RunDate")`). This makes the same notebook reusable across dates/tenants.

### Fabric Notebook

**Q15. What is a Fabric notebook, and how do you use it?**
It's an interactive **Apache Spark** development environment supporting **PySpark, Scala, Spark SQL, R**, with Markdown cells. It runs on a Spark pool/session billed to the Fabric capacity. Notebooks can run ad hoc, be scheduled, or be orchestrated by pipelines. Example: a PySpark notebook reads raw CSV and writes a Delta silver table.

**Q16. How is notebook compute billed?**
By **Capacity Units (CU)** consumed (vCore-seconds). The session's node size, autoscale settings, and idle timeout determine consumption. Right-sizing nodes and setting idle timeouts control cost.

### Capacity

**Q17. What is a Fabric capacity (F SKU), and how does consumption work?**
Fabric runs on capacity purchased as **F SKUs** (F2 → F2048). Each SKU provides a pool of **Capacity Units (CUs)**. All workloads (Spark, SQL, pipelines, Power BI queries) draw from this shared pool; if demand exceeds the pool, jobs are throttled/queued. Workspaces are assigned to a capacity; pausing a dev capacity stops billing and workloads.

### Security Model

**Q18. How does security work in Fabric?**
One security model across all workloads. Levels: tenant policy → capacity → **workspace roles** (Admin/Member/Contributor/Viewer) → item → table → row/column. **Row-level security** can be enforced via T-SQL security predicates in the Warehouse or Power BI roles. **Microsoft Entra ID** provides SSO and service-principal auth, and **Purview** enforces sensitivity labels, DLP, and lineage.

**Q19. What is Direct Lake, and why does it matter?**
Direct Lake lets Power BI semantic models load **Delta Parquet files from OneLake directly into memory**, giving import-mode performance without importing/copying data. It means reports are always current and storage is not duplicated — a key Fabric advantage.

**Q20. What are Data Activator and Real-Time Intelligence used for?**
**Real-Time Intelligence** handles streaming data via **Event Streams** (ingestion from Event Hubs/IoT/Kafka), **KQL databases** (fast time-series queries), and reflex-based alerting. **Data Activator** is a no-code layer that detects conditions (e.g., "stock below threshold") and triggers actions like Teams alerts or Power Automate flows.

---

## Quick Revision Cheat Sheet

- **Fabric** = unified SaaS analytics platform from Microsoft (2023).
- **OneLake** = single logical lake, Delta-first, shared by all workloads.
- **Lakehouse** = files + Delta tables, Spark-driven ETL.
- **Warehouse** = T-SQL-native, full DDL/DML, governed modeling.
- **Pipelines** = ADF-style orchestration: copy, data flows, notebooks, stored procs.
- **Notebooks** = Apache Spark (PySpark/Scala/SQL) interactive + scheduled.
- **Mirroring** = continuous no-code replication of source DBs into OneLake.
- **Shortcuts** = logical pointers (S3/ADLS/OneLake) — no data copy.
- **Real-Time** = Event Streams + KQL databases + reflexes.
- **Security** = one model, workspace roles, RLS, Entra ID, Purview.
- **Capacity** = F SKUs → CUs → throttling; assign workspaces, pause to save cost.
- **Power BI** = built in; **Direct Lake** reads OneLake Delta files in-memory.
