# PySpark

## What is PySpark

### PySpark as the Python API for Apache Spark

- PySpark is the **Python binding for Apache Spark** — it exposes Spark's distributed computing engine to Python developers through a familiar, Pandas-like API.
- You write Python, but the heavy lifting (planning, scheduling, executing) happens in the **JVM**. Python is the driver/worker interface; the actual data crunching runs as JVM bytecode via Py4J bridges and serialized Python UDFs.
- It ships with four major sub-modules:
  - `pyspark.sql` — structured data: DataFrames, Spark SQL, streaming (the 95% of daily work)
  - `pyspark.core` — low-level RDD API and context plumbing
  - `pyspark.ml` — distributed machine learning (pipelines, feature transformers, estimators)
  - `pyspark.streaming` — the older DStream API (mostly superseded by Structured Streaming)

### Spark as a Unified Analytics Engine

- Apache Spark is a **unified analytics engine for large-scale data processing** — "unified" because one engine covers four workload families that used to require four different tools:
  - **Batch processing** — ETL, large-scale transformations on stored data
  - **Streaming** — near-real-time processing of continuously arriving data (Structured Streaming)
  - **SQL** — interactive/adhoc querying via Spark SQL with full SQL dialect support
  - **Machine Learning** — distributed training/inference via MLlib (plus GraphX for graphs)
- Before Spark, you stitched together MapReduce (batch), Storm/Flink (streaming), Impala/Presto (SQL), and Mahout (ML). Spark collapses all of these onto a single runtime and a single codebase.
- **The unifying insight** — all four workloads are expressed as operations on distributed collections (RDDs → DataFrames), so the same engine, optimizer, and memory model serve every case. That is why Spark, not Hadoop MapReduce, became the default processing engine.

### Why Spark (distributed, in-memory, fast on huge datasets)

- **Distributed** — a job is split across a cluster of machines; 10 TB no longer fits on one box, so you need horizontal scaling. Spark scales linearly: double the workers, roughly double the throughput.
- **In-memory computation** — intermediate data is cached in RAM across nodes instead of being written to disk between every stage (the MapReduce failure mode). This is what makes iterative algorithms and interactive queries orders of magnitude faster.
- **Fast on huge datasets** — a well-optimized Spark job does in minutes what single-machine Python/Pandas cannot do at all (Pandas needs the data in one machine's RAM) and what MapReduce took hours to do.
- **Fault tolerance** — via lineage (the DAG) rather than replication. If an executor dies, Spark recomputes only the lost partitions by replaying the recorded transformations, instead of re-running the whole job or restoring from backups.
- **One engine for all data shapes** — structured (SQL, Parquet, Delta), semi-structured (JSON), and unstructured (text, images as binary) — all handled by the same runtime.

### PySpark vs Pandas (single machine vs distributed)

| | Pandas | PySpark |
|---|---|---|
| Scope | Single machine, data must fit in RAM | Cluster, data can be TB/PB scale |
| Execution | Eager, in-process | Lazy, distributed, JVM-backed |
| API style | Python-native, dict/ndarray-heavy | Column expressions, Catalyst-optimized |
| Iteration speed | Fast on small/medium data | Slow to start (cluster + JVM spin-up) |
| Shuffle support | None — all in memory | Built-in but expensive |
| Best for | Cleaning, exploration, ML feature work on < RAM-sized data | Production ETL at scale, huge joins, streaming |

- The rule of thumb: **if the data fits in Pandas on one machine and processing is a one-off, use Pandas.** You get interactivity, rich functions, and zero cluster overhead. PySpark shines when data stops fitting, when you need a repeatable production pipeline, or when the workload is genuinely distributed.
- A common hybrid pattern: use PySpark to aggregate/bronze-clean at scale, then `.toPandas()` on the *small* result for exploration or plotting. Never `.toPandas()` a big DataFrame.

### Spark in the Data Engineering Stack

- Spark sits in the **processing/transformation layer** of the modern data stack:

```
Source systems (Kafka, RDBMS, SaaS, object storage)
        │  ingest
        ▼
Data lake / warehouse (S3/ADLS + Delta/Iceberg, Snowflake)
        │  read + transform
        ▼
Spark / PySpark  ◄── the transformation engine (batch + streaming)
        │  write
        ▼
Curated / aggregated layers  →  BI, ML, dashboards
```

- In practice: Spark reads from object storage or Kafka, transforms with DataFrame/SQL pipelines, and writes back to a lakehouse format (Delta, Iceberg, Hudi) or a warehouse. It is the compute engine, not the storage or governance layer.
- Spark also doubles as the **query engine** for interactive analytics (via Thrift server / SQL warehouses) and the **compute engine** under managed platforms like Databricks, AWS EMR, and Azure Synapse.
- **Interview follow-up:** Your pipeline processes 10 TB daily with a strict 4-hour SLA. Walk through where Spark would be the right choice, where you'd reach for something else (a warehouse, a stream processor, or plain SQL), and why.

## Spark Architecture

### Driver Node (main program, DAG planning)

- The **driver** is the process that runs your `main` function / your `spark-submit` script. It holds the `SparkSession`, your variables, and the orchestration state.
- Its responsibilities:
  - **Convert the program into a DAG** of stages and tasks — it analyzes your transformations and produces an execution plan.
  - **Schedule tasks** onto executors and track their status.
  - **Coordinate execution** — decide when stages start, when shuffle data is ready, when the job is done.
  - **Serve as the shell** — in `pyspark`/`spark-submit`, the Python process talks to the driver JVM.
- The driver is a **single point of failure** — if the driver dies, the job dies. In cluster deploy mode the cluster manager restarts it; in client mode you lose it.
- **Key failure mode:** the driver holds the results of `collect()` and metadata of the whole job. A job that calls `collect()` on a huge result OOMs the driver's memory even though the cluster has plenty of RAM. The driver's heap is a real constraint, not an afterthought.

### Executor Nodes (run tasks, store data)

- **Executors** are worker processes (one or more per worker machine) that actually execute tasks.
- Each executor:
  - Runs **tasks** (one thread per task slot, default 1 core per task) handed to it by the driver.
  - **Caches data** in its memory for reuse (persisted RDDs/DataFrames).
  - Stores **shuffle outputs** that later stages pull from.
  - Reports status and results back to the driver.
- More executors = more parallelism, but each executor carries JVM overhead (~1–2 GB baseline). Tiny executors waste memory on JVM overhead; huge executors hurt fault tolerance (one failure = a lot of recomputation). Tuning targets a middle ground (e.g., 4–8 GB with 2–4 cores each).
- **Scale inflection point:** everything works with a handful of executors. At hundreds of executors, scheduler overhead, shuffle congestion, and skew in one hot partition become the dominant problems — the cluster's throughput is bounded by its worst partition, not its average.

### Cluster Managers: Standalone, YARN, Kubernetes

- The **cluster manager** allocates resources (CPU, memory) and starts/stopped executors when the driver requests them. Spark doesn't care how machines are provisioned — it only needs a manager to lease resources.

| Manager | Best for | Notes |
|---|---|---|
| Standalone | Local dev, small self-managed clusters | Spark's built-in manager; simplest to set up |
| YARN | Existing Hadoop environments | Co-schedules Spark with MapReduce/Tez on shared clusters; secure, production-proven |
| Kubernetes | Containerized, cloud-native deployments | Dynamic pod scaling, better isolation, the modern default in the cloud |
| Local[*] | Dev/testing on one laptop | `master = "local[*]"` runs threads in one JVM |

- Cloud-managed platforms (EMR, Databricks, Synapse) hide this layer behind a UI/API — you pick cluster shape, they manage the manager.

### SparkContext and SparkSession

- **SparkContext (`sc`)** is the entry point to the low-level engine: it establishes the connection to the cluster manager, creates RDDs, broadcasts variables, accumulators. One SparkContext per JVM.
- **SparkSession** is the higher-level, unified entry point introduced in Spark 2.0. It wraps `SparkContext` plus the SQL, Hive, and Streaming contexts under one roof.
- **PySpark timeline** — before Spark 2.0 you juggled three objects:

```python
# Spark 1.x — three separate entry points
from pyspark import SparkContext
from pyspark.sql import SQLContext, HiveContext

sc = SparkContext("local[*]", "app")
sqlContext = SQLContext(sc)          # only for DataFrames/SQL
hiveContext = HiveContext(sc)        # only if you needed Hive
```

- Spark 2.0+ consolidated all of that into one object — `SparkSession` is now the only entry point you need, and `spark` is already created for you in notebooks.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("my_app") \
    .getOrCreate()

sc = spark.sparkContext          # the underlying SparkContext, still accessible
```

- **Where it fits:** SparkSession = user-facing API + SQL catalog; SparkContext = engine + cluster connection; HiveContext/streaming context = specialized pieces now folded into the session.

### Master/Worker Concept

- **Master** = the driver (the "boss" that plans and directs). **Workers** = the machines (executors) that store data and do the computing.
- This is a classic **master-worker** topology: the master never does the heavy data work, the workers never make scheduling decisions.
- Contrast with Hadoop MapReduce, where the JobTracker + TaskTrackers played a similar role but with coarser-grained scheduling and disk-bound execution. Spark's master schedules *fine-grained tasks within stages* and keeps intermediates in memory.

### Where PySpark Fits (Py4J between Python and JVM)

- PySpark is not "Spark rewritten in Python." It's a thin Python layer over the JVM engine:

```
┌─────────────────────────────────────────────────────────────┐
│ Python process                                              │
│   your code → pyspark.sql (Python objects)                  │
│                │                                            │
│          Py4J bridge (socket + JVM objects)                 │
│                │                                            │
│ JVM process                                                 │
│   SparkContext / SQLContext → Catalyst plan → Tasks         │
└─────────────────────────────────────────────────────────────┘
```

- **Py4J** lets the Python process call Java objects over a local socket — Python never sees the actual data rows during normal DataFrame operations; it sends the *plan* to the JVM and the JVM executes it.
- Consequences that matter:
  - **Column expressions are Python descriptors, not logic.** `df.select("a")` doesn't compute anything; it builds a logical plan that the JVM optimizes and executes. This is why DataFrame code is fast even though it's "Python."
  - **Python UDFs break the speed.** A `udf(lambda x: ...)` forces each row to cross the JVM↔Python boundary, serialize to a Python object, run Python, and serialize back. This is tens to hundreds of times slower than built-in expressions.
  - **Python 3.8+ still has the GIL** — pure Python worker code doesn't get multi-threaded speedups; parallelism comes from multiple Python processes (one per task), not threads.

### Lazy Evaluation

- Transformations are **lazy**: Spark records them as a recipe (the logical plan) and does nothing until an **action** demands a result.
- Building a plan: `read → filter → select → groupBy` costs nothing. The first time you call `count()`, `show()`, `collect()`, or `write`, Spark compiles and executes the whole plan.
- **Why it matters (the deep reason):**
  - Spark can **reorder and combine** steps — push filters down to the read, eliminate unused columns, combine adjacent operations — before any data is touched.
  - **Fail fast on schema/logic errors**: errors in transformations surface at the first action, not at the line where the code is written.
  - **Only read what you need** — a `select` of 3 columns can skip reading the other 20 from Parquet.
- The classic confusion: a notebook where "nothing runs" until you hit an action, and bugs look like they're in the wrong place because the stack trace points at the action, not the transformation.

### DAG (Directed Acyclic Graph) of Transformations

- Every Spark job is a **DAG** — nodes are RDD/DataFrame operations, edges are dependencies between them. "Acyclic" = no cycles = no recursion; the flow always moves forward.
- Spark's scheduler:
  - Splits the DAG at **shuffle boundaries** into **stages** (each stage = tasks that can run without moving data).
  - Runs stages sequentially; within a stage, tasks run in parallel across partitions.
  - Each stage is a "wave" of tasks; the slowest task in a stage defines that stage's wall-clock time (the straggler problem).
- The DAG is the single most useful artifact for debugging performance: **the Spark UI's DAG visualization** shows every stage, every shuffle (shown as a join/repartition arrow), and every sku.
- **Interview follow-up:** Given this transformation chain — `read → filter → join → groupBy → orderBy → write` — how many shuffle stages do you expect, where are they, and what determines whether the number goes up or down?

## SparkSession

### Creating a SparkSession

- The standard entry point — one builder, one call to create or reuse:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("CustomerETL") \
    .master("yarn") \                    # or "local[*]", "k8s://https://...", "spark://..."
    .config("spark.executor.memory", "8g") \
    .config("spark.sql.shuffle.partitions", "200") \
    .getOrCreate()
```

- `getOrCreate()` is the idiom: if a session already exists in this JVM, return it; otherwise build a new one. Use it everywhere (notebooks, jobs) so code is idempotent.
- `master` and most `config` values can also come from `spark-submit` flags or config files — in managed environments (EMR/Databricks) the cluster defines them, so the session builder only sets app-level options.

### Configurations (executor memory, shuffle partitions)

- Configuration is set via `.config(key, value)`, `spark-submit --conf`, or a `spark-defaults.conf` file. Precedence: command-line/`spark-submit` overrides files; runtime `spark.conf.set()` overrides for the session.

| Key | Default (roughly) | What it controls |
|---|---|---|
| `spark.executor.memory` | 1g | Heap per executor |
| `spark.executor.cores` | 1 | Task slots per executor |
| `spark.sql.shuffle.partitions` | 200 | Partition count after a shuffle (join/groupBy) |
| `spark.default.parallelism` | depends on cores | Default task parallelism when not specified |
| `spark.driver.memory` | 1g | Driver heap — where `collect()` results go |
| `spark.sql.adaptive.enabled` | true in Spark 3.2+ | Turns on AQE (auto-coalescing, skew join) |
| `spark.broadcast.threshold` | 10m | Table size under which a join is auto-broadcast |

- **The two numbers to tune first** are almost always `executor.memory/cores` (right-size the worker) and `sql.shuffle.partitions` (right-size parallelism after shuffle). Everything else is usually premature tuning until you've looked at the Spark UI.
- **Failure mode:** leaving shuffle partitions at 200 with huge data creates tasks that each shuffle gigabytes and run forever (or spill/OOM). With tiny data, 200 partitions means 200 near-empty tasks — slow startup overhead and many small files on write.

### spark context vs sql context vs session

- `SparkContext` — engine-level: cluster connection, RDDs, broadcast/accumulators. Still there, buried under the session.
- `SQLContext` / `HiveContext` — Spark 1.x-era wrappers for SQL functionality. Prefer them only for legacy code.
- `SparkSession` — the single modern entry point, combining context + SQL + Hive + Streaming + Catalog.

```python
spark.sparkContext.setLogLevel("WARN")              # context-level tuning
spark.sql("SELECT * FROM v LIMIT 5").show()         # SQL via the session
spark.catalog.listTables()                           # catalog metadata
```

### Reading Data (csv, json, parquet, jdbc) with Options

- All reads funnel through `spark.read.format(...).options(...).load(...)`, with convenience helpers for the common formats:

```python
# CSV — the one that needs the most options
df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .option("delimiter", "|") \
    .csv("s3://bucket/raw/sales/")

# JSON
df = spark.read.json("path/to/events/*.json")

# Parquet (schema is stored in the file — no inference needed, and it's the performance format)
df = spark.read.parquet("path/to/table/")

# JDBC — one partition per `partitionColumn` range, not one connection hammered by everything
df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql://dbhost/dbname") \
    .option("dbtable", "users") \
    .option("user", "ro_user") \
    .option("password", os.environ["PG_PASS"]) \
    .option("numPartitions", 8) \
    .option("partitionColumn", "id") \
    .option("lowerBound", 1) \
    .option("upperBound", 10000000) \
    .load()
```

- **Key CSV options that bite in production:** `inferSchema=true` costs a full extra pass over the data and can infer the wrong type on a dirty sample; better to define an explicit schema. `multiline=true` is needed for embedded newlines. `header=true` is almost always wanted.
- **JDBC loading without `partitionColumn`** serializes reads through one connection — with 10M rows that's minutes to hours. Always supply `partitionColumn`/`lowerBound`/`upperBound` + `numPartitions` to parallelize.
- **Credentials rule:** never hardcode passwords. Read from environment variables or a secrets manager, as in the example above.
- **Glob patterns** let you read subsets: `.csv("path/2026/part-*.csv")` reads only matching files — useful for time-windowed jobs.

### Writing Data (save modes: overwrite, append, error, ignore)

- `mode` controls what happens when the output location already exists:

```python
df.write \
    .mode("overwrite") \     # delete existing, write fresh — destructive, be careful
    .partitionBy("year", "month") \
    .parquet("s3://bucket/curated/sales")

df.write.mode("append").parquet("s3://bucket/curated/sales")   # add to what's there
df.write.mode("error").parquet("...")   # fail if target exists (default — the safe mode)
df.write.mode("ignore").parquet("...")  # silently skip if target exists (data loss risk!)
```

- **`overwrite` is a footgun** — it replaces the whole target. In lakehouse formats (Delta) overwrite is transactional; with plain Parquet it is not, and a mid-write failure can leave a half-replaced directory.
- **`ignore` is worse** — it silently does nothing if data exists, so a rerun of a pipeline can quietly produce *nothing* while the job "succeeds." Prefer `error` or explicit checks.
- For **atomic, idempotent output** on lake storage, write to a temp path then rename, or use Delta's `overwrite` semantics with `replaceWhere` for partition-scoped overwrites.
- Common options: `coalesce(n)` to control the number of output files, `.option("compression", "snappy")`, `.option("maxRecordsPerFile", ...)` to bound file size.

### Partitioning and Bucketing on Write

- **Partitioning** (`partitionBy`) — physical directory layout that lets later reads skip data:

```python
df.write.mode("overwrite").partitionBy("year", "month").parquet("s3://bucket/sales")
# produces: s3://bucket/sales/year=2026/month=01/part-0000.parquet ...
```

- Partition pruning means a query filtering on `year=2026` reads only that directory instead of the whole table. This is the single biggest cheap win for query performance.
- **The trap:** partition on low-cardinality, *stable* columns only. Partitioning on `customer_id` creates a million directories with a few rows each — "small files problem," metadata explosion, and slower queries than no partitioning.
- **Bucketing** (`bucketBy`) — hash-buckets rows into a fixed number of files within each partition:

```python
df.write \
    .mode("overwrite") \
    .bucketBy(16, "customer_id") \
    .sortBy("customer_id") \
    .saveAsTable("sales_bucketed")
```

- Bucketing pre-arranges data so that a **join on the bucket column becomes a shuffle-free, sort-merge join** — both sides hash the same key to the same bucket file, so no data movement is needed. This is the classic optimization for repeated joins on the same key.
- Buckets must match on both sides (same bucket column + bucket count) to get the benefit. One bucketing, N joins on that key, all fast.
- **Scale inflection:** partitioning pays off at the point where a full scan starts hurting (gigabytes+). Bucketing pays off when you join the same fact table to the same dimension repeatedly in one pipeline — the setup cost is amortized across the joins.

## DataFrames

### What is a DataFrame

- A DataFrame is a **distributed collection of rows with a schema**, conceptually "Pandas on a cluster."
- Key differences from a Pandas DataFrame:
  - **Distributed** — data is split into partitions spread across executors.
  - **Lazy + optimized** — every operation goes through Catalyst, which can reorder/rewrite it.
  - **Typed schema, untyped rows** — columns have types, rows are generic (no per-column Python objects at runtime).
- The mental model: a DataFrame = a **query plan + a schema**, not a materialized table. It becomes concrete only when an action runs.

### Creating DataFrames

```python
# From a local Python collection (fine for small config/test data — NOT for real data)
data = [("Alice", 34), ("Bob", 45), ("Carol", 29)]
df = spark.createDataFrame(data, ["name", "age"])

# From a Pandas DataFrame (small — the conversion materializes on the driver)
df = spark.createDataFrame(pandas_df)

# From an RDD (legacy path)
df = rdd.toDF(["name", "age"])

# From reading storage — the real-world path
df = spark.read.parquet("s3://bucket/table/")

# From SQL
df = spark.sql("SELECT name, age FROM people WHERE age > 30")
```

- `createDataFrame` from a local list uses **local inference** on the driver — pass the explicit schema for non-trivial data, because inference will guess wrong on `None`/mixed types.

### Schema and StructType/StructField

- A schema is a `StructType` — an ordered list of `StructField`s, each with a name, data type, and nullable flag:

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType

schema = StructType([
    StructField("name", StringType(), True),
    StructField("age", IntegerType(), True),
    StructField("signed_up_at", TimestampType(), True),
])

df = spark.read \
    .option("header", "true") \
    .schema(schema) \
    .csv("s3://bucket/people.csv")
```

- **Why define schemas explicitly instead of inferring:** deterministic types, faster reads (no inference pass), and schema enforcement — a dirty CSV row with a non-integer age fails at parse time with a clear error instead of silently becoming `null` or an unintended `double`.
- Full type list: `StringType, IntegerType, LongType, DoubleType, DecimalType, BooleanType, TimestampType, DateType, ArrayType, MapType, StructType, BinaryType`. `DecimalType(p, s)` is the one people get wrong — a decimal that doesn't fit precision silently truncates/rounds in some paths.
- Schema is readable/writable via `df.schema`, and you can copy a schema between DataFrames to force a mismatch to fail loudly rather than coerce.

### Row Objects

- Each DataFrame element is a `Row` — an ordered, access-by-name tuple:

```python
rows = spark.createDataFrame([("Alice", 34)], ["name", "age"]).collect()
r = rows[0]
r["name"]      # 'Alice'  — access by name
r.age          # 34       — access by attribute
r[0]           # 'Alice'  — access by position
```

- Rows appear in `collect()`, `take()`, `foreach()`, and UDF inputs. In vectorized DataFrame code you rarely touch Row objects directly — you manipulate Column expressions.
- `Row(name="Alice", age=34)` can also be constructed directly, which is handy for building test data.

### show(), printSchema(), describe()

```python
df.show()                # first 20 rows, truncated — set df.show(n=50, truncate=False) for more
df.printSchema()         # tree of column names and types
df.describe("age", "salary").show()   # count/mean/stddev/min/max per column
df.count()               # total rows
df.columns               # list of column names
df.dtypes                # list of (name, type-string) pairs
df.limit(5).collect()    # first 5 rows as a list of Row
```

- `show()` is the most-used debugging tool and triggers a full job run every time. A notebook with ten `show()` calls is a notebook that scans the data ten times — comment them out or keep them behind a flag in production pipelines.

### Column Expressions

- A **Column** is a typed reference or computed value: `df.col("age")`, `col("age")`, `df["age"]`, or the plain string `"age"`. They compose into expressions:

```python
from pyspark.sql.functions import col, when, lit

df.withColumn("is_adult", col("age") >= 18)                       # boolean column
df.withColumn("age_plus_one", col("age") + 1)                     # arithmetic
df.withColumn("bucket", when(col("age") < 18, "minor").otherwise("adult"))
df.withColumn("constant", lit(42))                                # literal column
```

- **The key mental shift from Pandas:** columns are *symbols* in a query plan, not Python objects holding data. `col("age") + 1` does not add anything — it appends an arithmetic expression to the plan that Catalyst may reorder or push down before the JVM ever touches a row.
- Rule of thumb: if you catch yourself writing `df.select(df['a'] > 5)` or iterating with Python to build an expression, you're doing it right — building Column trees is the idiomatic PySpark way; doing per-row Python logic is the anti-pattern.

## Transformations

- All the operations below are **transformations** — lazy, plan-building, and chainable. None executes until an action fires.

### select, withColumn, withColumnRenamed

```python
from pyspark.sql.functions import col, expr

df.select("name", "age")                          # choose columns by name
df.select(col("name"), col("age") * 2)            # choose columns + expressions
df.selectExpr("name", "age * 2 as double_age")    # SQL-flavored expressions inline

df.withColumn("double_age", col("age") * 2)       # add or replace a column
df.withColumnRenamed("age", "years_old")          # rename a column

# withColumn chain gotcha: every withColumn rewrites the plan
df = df.withColumn("a", ...).withColumn("b", ...).withColumn("c", ...)
```

- `withColumn` **overwrites** the column if the name already exists and appends if it doesn't.
- **Gotcha:** each `withColumn` builds a new plan node; 50 chained `withColumn`s create 50 nodes that Catalyst generally folds together, but a huge chain can still bloat the plan. Grouping into a single `select(...)` or `selectExpr(...)` with all the derived columns is cleaner.
- `select` drops unlisted columns; `withColumn` keeps everything. Know which one you want or you'll silently lose columns.

### filter/where

```python
df.filter(col("age") >= 18)
df.where("age >= 18")          # identical — where is an alias
df.filter(col("name").startswith("A") & (col("age") > 30))

# Conditions that DON'T work the way Pandas users expect:
df.filter(col("age") > 18 and col("salary") > 100)   # SyntaxError — can't use `and` on Columns
df.filter((col("age") > 18) & (col("salary") > 100)) # correct — bitwise & (also | for OR)
```

- Two classic interview traps: use `&` / `|` / `~` (bitwise) not `and` / `or` / `not` on Columns, and remember **`NULL` comparisons are never true** — `col("x") != 5` drops NULL rows. Use `col("x").isNull()` / `.isNotNull()` explicitly.

### groupBy, agg, count

```python
from pyspark.sql.functions import count, sum, avg, max, min, countDistinct

df.groupBy("department").count().show()

df.groupBy("department") \
  .agg(
      count("*").alias("n"),
      sum("salary").alias("total_salary"),
      avg("salary").alias("avg_salary"),
      max("hire_date").alias("last_hire"),
      countDistinct("manager_id").alias("distinct_managers"),
  ).show()

# Multiple group-by columns
df.groupBy("department", "job_level").agg(sum("salary")).show()
```

- `groupBy` is a **wide** transformation — it triggers a shuffle (rows with the same key go to the same partition) before aggregating.
- `agg` takes named expressions; giving results `alias`es keeps the output schema readable. A common silent bug: two aggregations both named `sum(salary)` → the second overwrites the first's column in the output.

### orderBy/sort

```python
df.orderBy("age")                        # ascending by default
df.orderBy(col("age").desc())            # descending via Column
df.orderBy(col("department").asc(), col("salary").desc())   # multiple keys
df.sort("age")                           # alias of orderBy
df.sortWithinPartitions("age")           # sort only inside each partition (no global shuffle)
```

- `orderBy` is a global sort → full shuffle + single set of sorted partitions. `sortWithinPartitions` skips the global shuffle — useful only when you don't need overall order.
- Sorting is almost always the *last* step before write/action; sorting early just to re-sort later is wasted work.

### join (types: inner, left, right, outer, leftsemi, leftanti)

```python
df1.join(df2, "user_id")                             # inner join on a shared column
df1.join(df2, df1["user_id"] == df2["id"])           # join on different column names
df1.join(df2, on=["user_id"], how="left")            # left outer
df1.join(df2, on=["user_id"], how="inner")
df1.join(df2, on=["user_id"], how="right")
df1.join(df2, on=["user_id"], how="full")            # full outer
df1.join(df2, on=["user_id"], how="leftsemi")        # keep df1 rows that MATCH in df2
df1.join(df2, on=["user_id"], how="leftanti")        # keep df1 rows that DON'T match in df2
```

| Type | Keeps | Use case |
|---|---|---|
| inner | matching rows from both | default; rows present on both sides |
| left | all df1, matches from df2 (else NULL) | attach dimension attributes, keep all facts |
| right | all df2, matches from df1 (else NULL) | mirror of left; rarely used |
| full | all rows both sides, NULLs where missing | reconciliation, diffing two systems |
| leftsemi | df1 rows where a match exists | "which customers bought anything" |
| leftanti | df1 rows with no match | "which customers bought nothing" |

- **The trap most people hit:** after a `left` join, the column names from df2 that collide with df1 (or that appear on both sides) get suffixes `_1`/`_2` — surprise duplicate columns. Use explicit suffixes or select the needed columns to avoid ambiguity.
- `leftsemi`/`leftanti` are far cheaper than a `left` join + filter because they don't materialize the df2 columns at all — reach for them when you only need an existence test.

### union, distinct, dropDuplicates

```python
df1.union(df2)                       # appends rows; requires identical schemas & same order
df1.unionByName(df2)                 # safer: matches by column name (Spark 3+)
df.distinct()                        # removes fully-duplicate rows (shuffle)
df.dropDuplicates(["user_id"])       # keep one row per user_id — dedupe on subset
df.dropDuplicates(["user_id"]).count()   # dedupe then count
```

- `union` (no dedupe) is not `unionAll` from SQL. Column order must match exactly or data gets scrambled silently — `unionByName` avoids the footgun.
- `distinct()`/`dropDuplicates` are shuffle-heavy (a reduce-by-key on the whole row). On huge data, dedupe on the *minimum set of keys* that define uniqueness, not all columns.

### cast, when/otherwise, coalesce

```python
from pyspark.sql.functions import when, col, coalesce, lit

df.withColumn("age_int", col("age").cast("int"))            # cast
df.withColumn("age_int", col("age").cast(IntegerType()))    # or typed variant

df.withColumn("status_group",
    when(col("status") == "PENDING", "Open")
    .when(col("status").isin("COMPLETE", "FAILED"), "Closed")
    .otherwise("Unknown"))

df.withColumn("effective_price", coalesce(col("discounted_price"), col("list_price")))
```

- `cast` on failure produces `null`, **not an error** — a dirty string becoming NULL is silent data corruption unless you validate. Catch it with `.isNull()` checks or a `try_cast` where available.
- `coalesce` returns the first non-NULL of its arguments — the idiomatic way to implement fallback/default logic. (Note: this column-function `coalesce` is unrelated to the RDD-level `coalesce` repartitioning method.)
- `when/otherwise` chains are the `CASE WHEN` of DataFrame land; every `.when()` is one branch, the last `.otherwise()` is the else.

### Window Functions (row_number, rank, lag, lead, sum over window)

- Window functions compute values **across a group of rows related to the current row**, without collapsing the group (unlike groupBy). Perfect for "top-N per group" and running totals:

```python
from pyspark.sql import Window
from pyspark.sql.functions import row_number, rank, lag, lead, sum as _sum

w = Window.partitionBy("department").orderBy(col("salary").desc())

df.withColumn("rn", row_number().over(w)) \
  .filter(col("rn") == 1) \                       # top paid per department
  .select("department", "name", "salary")

df.withColumn("prev_salary", lag("salary", 1).over(w))        # previous row's value
df.withColumn("next_salary", lead("salary", 1).over(w))       # next row's value
df.withColumn("rank", rank().over(w))                          # dense/rank variants

# Running total per department (unbounded window)
w2 = Window.partitionBy("department").orderBy("date").rowsBetween(Window.unboundedPreceding, Window.currentRow)
df.withColumn("cum_salary", _sum("salary").over(w2))
```

- **Difference between the row-numberers:** `row_number()` gives 1,2,3,4 on ties; `rank()` gives 1,1,3,4 on ties (gaps); `dense_rank()` gives 1,1,2,3 on ties (no gaps). "Top-N per group" needs `row_number() = 1` semantics more often than people think.
- Windows are a shuffle too (rows with the same partition key must land on one executor). A window that partitions by a high-cardinality key plus sorts is one of the heavier patterns in the engine.
- Always `partitionBy` with a bounded key set; a window without partitionBy is a single giant partition — the whole sort happens on one executor.

### String functions, date functions

```python
from pyspark.sql.functions import (
    upper, lower, trim, length, substring, split, concat, concat_ws,
    regexp_replace, regexp_extract, lpad, rpad, initcap,
    year, month, dayofmonth, to_date, to_timestamp, date_add, date_sub,
    datediff, months_between, date_format, current_date, unix_timestamp,
)

df.withColumn("name_upper", upper("name"))
df.withColumn("name_clean", trim(lower("name")))
df.withColumn("domain", regexp_extract("email", "@(.*)$", 1))
df.withColumn("phone", regexp_replace("phone", r"[^0-9]", ""))   # strip non-digits
df.withColumn("full", concat_ws(" ", "first", "last"))

df.withColumn("dt", to_date("date_str", "yyyy-MM-dd"))           # parse string → date
df.withColumn("ts", to_timestamp("ts_str", "yyyy-MM-dd HH:mm:ss"))
df.withColumn("yr", year("dt"))
df.withColumn("days_since", datediff(current_date(), "dt"))
```

- **Format-string trap:** Spark uses Java SimpleDateFormat patterns (`yyyy`, `MM`, `dd`), not Pandas (`%Y`, `%m`, `%d`). Mixing them silently produces NULLs.
- Date parsing failures yield NULL, not exceptions — validate with `count` before/after, or you'll ship a report full of blanks.
- `regexp_replace` with a malformed regex fails the whole job at the action; wrap date/string parsing stages in validation so the failure is caught where the bad data is, not downstream.

## Actions

### What Actions Do

- Actions are the operations that **trigger the actual execution** of the lazy plan. Every action forces Spark to compile the DAG and run it (at least partially).
- Common actions:

```python
df.collect()                      # ALL rows to driver as a Python list of Row — dangerous for big data
df.take(5)                        # first 5 rows to driver
df.first()                        # first row
df.head()                         # same as first
df.count()                        # number of rows (triggers full scan)
df.show()                         # prints 20 rows (triggers run)
df.write.parquet("...")           # the action that materializes output
df.foreach(lambda r: ...)         # apply a function to each row on the executors
df.foreachPartition(...)          # apply a function to each partition (use for batch writes)
```

- **`collect()` is the most dangerous action.** It pulls every row to the driver, so it is bounded by the driver's heap, not the cluster's capacity. `collect()` on a 50 GB DataFrame = OOM (or an hour of shipping data over the network) even when the cluster is healthy.
- The safe pattern: `take`/`limit` for peeking, `collect` only after an aggregation that has already shrunk the data (`.groupBy(...).agg(...).collect()`), and `toPandas()` only on small results.
- `write` is an action too — the whole reason you build the pipeline. It's worth repeating: *you are not writing anything until `write` is called*, which confuses beginners who think `df = spark.read...df.filter(...)` already produced a file.

### Difference between Transformations and Actions

| Transformations | Actions |
|---|---|
| Lazy — only build the plan | Eager — trigger execution |
| Return a new DataFrame/RDD | Return a value, print, or write |
| Can be chained infinitely | End the pipeline |
| Re-run from scratch if not cached | — |

- The one-sentence interview answer: **transformations describe *what* to compute, actions force *when* to compute it.** Spark uses the gap to reorder, fuse, and push down the work.

## RDDs (brief)

### What is an RDD

- An **RDD (Resilient Distributed Dataset)** is the original Spark abstraction: an immutable, partitioned collection of elements that can be operated on in parallel, with built-in fault tolerance via lineage.
- Resilient = if partitions are lost, they're recomputed from the recorded lineage (the sequence of transformations), not restored from a replica.
- Created by `parallelize`, reading files (`sc.textFile`), or transforming existing RDDs (`map`, `flatMap`, `filter`, `reduceByKey`).
- You rarely touch RDDs in modern PySpark — DataFrames replaced them for 99% of work. `df.rdd` is still there if you truly need it.

### RDD vs DataFrame (Catalyst optimization, schema)

- **RDDs**: no schema, no optimizer — every transformation is executed as-is, and every element is a raw Python object. Flexible but unoptimizable.
- **DataFrames**: schema + Catalyst query optimizer + Tungsten code generation. Spark can *rewrite your plan* (push filters, reorder joins, prune columns) because it understands the structure. RDDs don't give it that lever.
- Measurable consequence: DataFrame/Spark SQL pipelines are typically **several times faster** than equivalent RDD pipelines on the same cluster, and they handle schema evolution, pushdown, and predicate pruning automatically.

| | RDD | DataFrame |
|---|---|---|
| Schema | No (opaque objects) | Yes (typed columns) |
| Optimization | None | Catalyst + Tungsten |
| Per-row overhead | High (Python objects) | Low (columnar/off-heap) |
| Serialization | Pickle | Optimized binary (Arrow-backed in 3.x) |
| Fault tolerance | Lineage | Lineage (same) |

### When you might use RDDs (rarely)

- Highly custom, low-level processing where the DataFrame API has no expression (rare).
- Interacting with third-party libraries that only understand RDDs (some legacy ML/geospatial code).
- Controlling partitioning/placement at a very fine grain that DataFrames abstract away.
- **The honest answer for interviews:** "I would use DataFrames/Spark SQL almost always; I'd reach for an RDD only when a UDF-based, per-element operation cannot be expressed as columnar logic — and I'd profile before concluding that's the case, because a UDF on a DataFrame is usually the wrong tool too."

## Spark SQL

### createOrReplaceTempView

```python
df.createOrReplaceTempView("sales")     # register the DataFrame as a SQL temp view

spark.sql("SELECT department, SUM(amount) AS total FROM sales GROUP BY department").show()
```

- `createOrReplaceTempView` registers a **session-scoped, in-memory** view — it does *not* persist data, it just makes the DataFrame queryable via SQL for the rest of the session.
- `createTempView` (fails if the name exists) vs `createOrReplaceTempView` (overwrites) vs `createGlobalTempView` (shared across sessions; use `global_temp.sales`).
- The view is a reference to the DataFrame's plan — re-executing a query on it re-runs the plan unless the DataFrame was cached.

### Spark SQL Queries on DataFrames

- Everything you can do in the DataFrame API, you can do in SQL, and both compile to the *same* Catalyst plan:

```python
df.createOrReplaceTempView("users")

result = spark.sql("""
    SELECT dept,
           COUNT(*) AS n,
           AVG(salary) AS avg_salary,
           ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
    FROM users
    WHERE hire_date >= '2020-01-01'
    GROUP BY dept
""")

result.show()

# You can also register a DataFrame built by SQL back to the API
result.createOrReplaceTempView("dept_stats")
spark.sql("SELECT * FROM dept_stats ORDER BY n DESC").show()
```

- SQL supports the full Spark dialect: CTEs (`WITH`), subqueries, window functions, `LATERAL VIEW explode` for arrays, and even UDFs registered via `spark.udf.register`.
- Spark SQL sits on the Hive metastore for catalog-backed tables: `spark.sql("SHOW TABLES")`, `DESCRIBE`, `MSCK REPAIR TABLE` etc.

### SQL over DataFrames

- **Interop rules:** a DataFrame can become a temp view (`df.createOrReplaceTempView`) and a SQL query can become a DataFrame (`spark.sql(...).toDF`). Both compile to identical physical plans, so mixing costs nothing.
- Use whichever is more readable for the operation. For one-off complex analytical queries, SQL is usually clearer. For programmatic pipelines, DataFrames with typed expressions give you refactorability and type safety.

### When to use SQL vs DataFrame API

- **Prefer SQL when:** the query is a complex analytical read (multi-join, windowed aggregation, subqueries); the team is SQL-first; you want copy-paste between the BI tool and Spark.
- **Prefer the DataFrame API when:** building reusable, parameterized pipelines; need to introspect schemas; want programmatic joins with `join` conditions; writing logic that branches on runtime values.
- **Interview answer:** "Both are front-ends to Catalyst — the same optimizer — so this is a readability and maintainability choice, not a performance one. I use SQL for analytical reads and the API for pipelines."

## Performance and Optimization

### Lazy Evaluation Benefits

- Listed early as a concept; the *performance* payoff is the theme:
  - Filter and projection **pushdown** — predicates (`year = 2026`) get pushed into the file reader, so whole Parquet row-groups never enter memory.
  - **Plan fusion** — adjacent transformations fuse into one pass over the data instead of N passes.
  - **Fail fast** — errors surface before expensive work, not after.
- The cost: debugging is harder (errors appear at actions, not at the transformation line), and two actions on the same lazy DataFrame re-run the whole plan from scratch unless you cache.

### Catalyst Optimizer

- Catalyst is Spark's **query optimizer** — it takes your logical plan and rewrites it into a faster physical plan using rules:

```python
# What you wrote (logical plan)
df.filter(col("dept") == "eng").select("name", "dept")...

# What Catalyst may do:
# 1. Predicate pushdown  — read only files/blocks containing dept='eng'
# 2. Projection pruning  — read only name & dept columns from storage
# 3. Reorder joins       — broadcast the small table first
# 4. Constant folding    — col("x") + 1 + 1 → col("x") + 2
# 5. Expression simplification — rewrite WHEN/ELSE trees
```

- Catalyst works in phases: **analysis** (resolve names/types) → **logical optimization** (rule-based rewrites) → **physical planning** (choose join algorithms, data sources) → **code generation** (Tungsten).
- Why you should care: Catalyst makes "write the readable plan" mostly free. The query you *write* is not the query that *runs* — which is exactly why you should profile before hand-tuning.

### Tungsten (off-heap memory, code generation)

- Tungsten is Spark's **execution engine overhaul** that makes Catalyst's plans fast:
  - **Off-heap memory / binary format** — rows stored as compact binary instead of JVM objects, massively cutting GC pressure and improving cache locality.
  - **Whole-stage code generation** — the engine generates JVM bytecode for an entire stage, eliminating virtual function calls and per-record object allocations.
  - **Cache-aware algorithms** — sorts and joins optimized for CPU cache lines.
- Net effect: modern Spark executes most columnar work at near-Java-optimized speed even though you wrote Python, as long as you use built-in expressions rather than Python UDFs.
- **The unspoken rule of thumb:** if your pipeline is UDF-heavy, Tungsten is working for *you* barely at all — every Python UDF round-trip defeats the code-generation gains.

### Shuffle and Why It's Expensive

- A **shuffle** is the movement of data across executors — required by `groupBy`, `join` on non-broadcast keys, `distinct`, `orderBy`, `repartition`, window partitioning, and `coalesce` (when increasing partitions).
- During a shuffle, each partition's rows are **hashed by key** and written to disk, then pulled by the right remote executor over the network, then re-read. That's: disk I/O + network I/O + serialization + recompute all in one operation.
- **Costs:** the shuffle phase alone is often 60–90% of a job's wall time. Everything before and after is fast; the shuffle is where time, memory, and disk go.
- **The symptom of a bad shuffle:** a stage with a huge "shuffle read" row in the Spark UI, or a long gap with no progress — data is being written/re-read across the network.
- Reduction strategies, in order: (1) filter/aggregate *before* joining; (2) broadcast small tables so no shuffle happens at all; (3) pre-partition/bucket on the join key; (4) tune `spark.sql.shuffle.partitions` so partition size is sane.

### Coalesce vs Repartition

- `repartition(n)` — full shuffle, **can increase or decrease** partitions, re-balances evenly.
- `coalesce(n)` — merge partitions **without a full shuffle** when decreasing (n < current); cheap because it just reduces the number of files/tasks by combining partition data on the same executors.

```python
df.repartition(4)                 # even split into 4 (full shuffle)
df.coalesce(2)                    # reduce to 2 partitions (no shuffle if going down)
df.repartition(col("region"))     # repartition by key → colocated by region (use for writes/joins)
```

- **Common mistake:** calling `coalesce(1)` to write one output file — it collapses everything to one partition, one task, and one executor: an hours-long single-threaded write. If you need fewer files, `coalesce` to a number that still gives parallelism (e.g., 8–16), or use `maxRecordsPerFile` instead.
- `repartition(col)` to match join keys on both sides turns a shuffling join into a shuffle-free sort-merge join — one of the classic tricks when you can't bucket.

### Broadcast Joins (small tables)

- When one side of a join is small (default under ~10 MB), Spark can **copy it to every executor** and skip the shuffle entirely — each executor joins its local partition against the in-memory copy.
- Enforced/controlled with a hint:

```python
# Auto: Spark broadcasts tables under spark.sql.autoBroadcastJoinThreshold (default 10m)

# Explicit hint (spark 3.x)
df_large.join(df_small.hint("broadcast"), "customer_id")

# Disable auto for a specific join if it's causing OOM
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
```

- **Trade-off:** the small table is copied to *every* executor's memory. A "small" table that's actually 5 GB gets copied 100 times = 500 GB of RAM and likely executor OOM. The threshold exists because the copy overhead is O(executors × table size).
- **Why it's the #1 interview optimization:** because it eliminates the shuffle entirely — the most expensive operation in Spark — and it's a one-line change.

### Bucketing and Partitioning for Joins

- Covered under writes; the payoff is on the read side: if both tables are **bucketed on the same join key with the same bucket count**, Spark can do the join with **no shuffle** — matching bucket files (same hash of the key) sit on the same executor, so rows that need to join are already colocated.
- Partitioning helps **prune** (skip whole directories for filtered queries); bucketing helps **colocate** (skip shuffles for joins/aggregations). They're complementary.
- Checklist for a shuffle-free bucketed join:
  - Both tables bucketed on the join key.
  - Same number of buckets (or a multiple) on both sides.
  - `spark.sql.sources.bucketing.enabled` on (default true in modern Spark).

### Caching and Persisting (cache, persist MEMORY_AND_DISK)

- `cache()` = `persist(StorageLevel.MEMORY_AND_DISK)` — keep the DataFrame's computed partitions in executor memory (spill to disk if needed) so subsequent actions reuse them instead of re-running the whole lineage.

```python
df_joined = df1.join(df2, "id").filter(...)
df_joined.cache()
df_joined.count()            # forces the computation ONCE; results are now cached
df_joined.write.parquet("out/a")   # reuses cached data
df_joined.write.parquet("out/b")   # reuses cached data
```

- **The pattern everyone forgets:** caching is *lazy* — the data isn't cached until the first action runs (`count()` or a write). Cache, then act, then reuse.
- Storage levels: `MEMORY_ONLY`, `MEMORY_AND_DISK`, `DISK_ONLY`, `MEMORY_AND_DISK_SER`, `OFF_HEAP`. For DataFrames, `MEMORY_AND_DISK` is the sane default — deserialized objects in RAM, spill to disk rather than recompute.
- **When caching pays:** the same DataFrame feeds multiple downstream actions/stages, or a loop iterates over a DataFrame. When caching hurts: single-use DataFrames (you just paid for an extra full materialization + memory footprint for nothing).
- **The failure mode:** caching everything "just in case" blows up executor memory, forcing spills, eviction, and GC pressure — the pipeline is slower than it was without caching. Cache the *narrow* (small, reused) results, not the wide source reads.
- Always `df.unpersist()` when done — cached data lives until evicted or the session ends.

### Spilling to Disk

- When executors run out of memory mid-task (sort, join, aggregation), Spark **spills** intermediate data to local disk and continues, rather than failing.
- Spilling is a **correctness-safe degradation but a performance cliff** — serializing to and from disk mid-operation can be 10–100× slower than the in-memory path. The Spark UI shows "shuffle spill (memory + disk)" — if disk spill is non-zero, that's your smoking gun for a memory problem.
- Fixes in priority order: reduce per-task data (more partitions), reduce shuffle output (filter early), add executor memory, use serialized storage levels.
- **Why it looks correct:** the job completes — no error! — so teams ship it. But the job is running at 1/20th of its potential, and "it completed" hides the cost.

### Adaptive Query Execution (AQE)

- AQE (Spark 3.0+, **on by default in 3.2+**) re-optimizes the physical plan **at runtime**, using stats gathered mid-execution:
  - **Dynamic coalescing of shuffle partitions** — reduces 200 tiny partitions to a few big ones when data is small.
  - **Dynamic join strategy switching** — promotes a join to broadcast at runtime if a table turns out small.
  - **Dynamic skew join handling** — splits a skewed partition into multiple tasks so one straggler doesn't bottleneck the stage.
  - **Optimize locality** — co-locates tasks with their data.
- Enable/verify:

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

- **Consequence:** "shuffle partition tuning is mostly dead." AQE handles the common cases automatically, so the manual `spark.sql.shuffle.partitions` tuning that dominated Spark 2.x guides is largely obsolete — modern answer is "let AQE handle it, tune only after the UI says otherwise."

### Dynamic Partition Pruning

- When joining a large partitioned fact table to a small dimension, **DPP** pushes the dimension's filter down to prune fact partitions *before* the shuffle:

```sql
-- Without DPP: scan ALL partitions of events, join, then filter
-- With DPP: filter the country table first, then only scan the matching event partitions
SELECT e.* FROM events e
JOIN countries c ON e.country_id = c.id
WHERE c.continent = 'Europe'
```

- Enabled by default in Spark 3.x (`spark.sql.optimizer.dynamicPartitionPruning.enabled`, on in 3.1+). The big win: joins where the right side's filter could have pruned the left side's partitions but a static plan couldn't know it until runtime.
- **Interview one-liner:** DPP + AQE + broadcast hints are the three "modern Spark just does it for you" optimizations — enable them, then spend tuning time only where the UI actually shows a problem.

## Spark Streaming (brief)

### Spark Structured Streaming

- Structured Streaming is Spark's **modern streaming API** — you write your query as if it were a batch DataFrame, and Spark turns it into a **continuously executing micro-batch** (or continuous processing) job.
- The key design bet: **"treat a stream as an unbounded table."** New data arriving = new rows appended to that table; your query is re-evaluated against the growing table.
- Same API as batch: `readStream` produces a DataFrame, you apply `filter`/`select`/`groupBy`/`window`, and `writeStream` publishes results.

### Micro-Batch Processing

- By default, Structured Streaming processes data in **micro-batches**: incoming data in the last trigger interval (default 0.5–1s) is processed as a batch job. This gives exactly-once semantics and reuses the batch engine.
- Micro-batch = near-real-time latency (sub-second to a few seconds), not true event-at-a-time latency. If you need low-millisecond latency, the continuous processing mode exists but with fewer features.
- **Trade-off to name in interviews:** micro-batch reuses all the batch optimizations (Catalyst, Tungsten, exactly-once with checkpoint) at the cost of latency; continuous mode trades those guarantees/complexity for lower latency.

### ReadStream / writeStream

```python
# Define the source — a directory of JSON files appearing over time
stream_df = spark.readStream \
    .schema(schema) \
    .json("s3://bucket/events/")

# Apply transformations like a batch DataFrame
agg = stream_df.groupBy("event_type") \
    .agg(count("*").alias("n")) \
    .withColumn("event_time", current_timestamp())

# Write the stream — every trigger interval, write the new results
query = agg.writeStream \
    .outputMode("update") \
    .format("console") \          # or "parquet", "delta", "kafka"
    .option("checkpointLocation", "s3://bucket/checkpoints/events") \
    .trigger(processingTime="5 seconds") \
    .start()

query.awaitTermination()
```

- Sources: Kafka, file directories (watch for new files), sockets, Delta tables, rate. Sinks: Kafka, files/Delta, console, memory, foreachBatch.
- `foreachBatch` is the power tool — it gives you a *batch* DataFrame of the micro-batch's data, so you can run arbitrary logic (dedupe, write to multiple sinks, merge into Delta with `merge` semantics) exactly once per batch.

### Checkpointing for Fault Tolerance

- The **checkpoint location** persists the stream's progress: which offsets have been consumed, the state of aggregations/windows, and in-flight batches.
- On a crash/restart, the stream resumes **exactly from the last committed offset** — this is what makes exactly-once processing possible. Losing the checkpoint (e.g., pointing it at a temp path) = lost or duplicated data.
- Rules: use a durable location (S3/ADLS/GCS), **never change** the checkpoint path mid-stream, and don't share one checkpoint between two stream queries.

### Output Modes: append, update, complete

- **Append** — write only the *new* rows since the last trigger. Valid when rows are never updated (simple filters/projections). Best performance; you are guaranteed no rewriting of old results.
- **Update** — write only rows that *changed* in the last trigger (new + updated). Requires upsert-capable sinks (Kafka, Delta). Stateful aggregations can use this.
- **Complete** — rewrite the *entire* result table every trigger. Only for aggregations, and only when the aggregate result is small enough to rewrite.
- **The gotcha:** choosing an invalid output mode for a query is a runtime error, not a warning — an aggregation that must rewrite old state cannot use append.

## Common PySpark Mistakes

### collect() on Huge Datasets

- **The wrong pattern:** calling `.collect()` on a large DataFrame "to see the data" or to loop over rows in Python.
- **Why it looks correct:** `collect()` is the most obvious way to get data into Python, and it works fine in tutorials and small test frames.
- **Production symptom:** driver OOM (OutOfMemoryError) — the driver heap fills because *all* results are shipped to one JVM; or, if it survives, a multi-hour network transfer while the cluster idles.
- **The fix:** `show()`/`take(n)`/`limit(n)` to peek, aggregate before collecting, or use `foreachPartition` for per-partition logic so results never aggregate on the driver.

### Using Python Loops on DataFrames

- **The wrong pattern:** `for row in df.collect(): ...` or `df.rdd.map(lambda r: <python>)` for anything that touches all the data, and UDFs when a built-in expression exists.
- **Why it looks correct:** Python loops are the natural tool for per-row logic; the DataFrame "API doesn't obviously cover it."
- **Production symptom:** jobs that run 10–100× slower than their equivalent columnar pipeline; the Spark UI shows heavy serialization time and near-zero codegen.
- **The fix:** use Column expressions (`when`, `coalesce`, string/date functions), and reserve UDFs for logic that genuinely cannot be expressed columnar — then prefer a **vectorized Pandas/Arrow UDF** over a scalar one.

### Not Understanding Shuffles (multiple joins causing data skew)

- **The wrong pattern:** chaining several large joins without regard to key distribution — e.g., joining on a skewed key (one city holds 80% of the rows) or joining after a `filter` that should have come first.
- **Why it looks correct:** joins are one word in the code (`join`), and small-scale tests run on evenly distributed test data.
- **Production symptom:** one task in a stage runs for an hour while 199 finish in seconds (data skew — the hot partition is the bottleneck); or a stage does a massive shuffle read because the small table was never broadcast.
- **The fix:** filter/aggregate before joining, broadcast genuinely small tables, check `explain()` for sort-merge vs broadcast, and enable AQE skew join. On skewed keys, consider salting the key.

### Over-Caching

- **The wrong pattern:** calling `.cache()`/`.persist()` on every DataFrame "for performance" without any reuse.
- **Why it looks correct:** caching is presented as *the* optimization; it costs nothing to type.
- **Production symptom:** executor memory exhaustion → spill to disk, cache eviction churn, and GC pressure — the job gets *slower* than the uncached version, and the UI shows cache entries being evicted as fast as they're written.
- **The fix:** cache only DataFrames that are read multiple times downstream and are the *result* of expensive transformations. Cache the narrow, post-aggregation results, not the wide raw reads. Call `.unpersist()` when done.

### Bad Partitioning Strategy (many small files)

- **The wrong pattern:** writing a DataFrame with 400 partitions to a partitioned-by-date lake → 400 tiny files per day; or partitioning on a high-cardinality column like `customer_id`.
- **Why it looks correct:** more partitions = more parallelism; partitioning on the "natural" key feels right.
- **Production symptom:** query times *increase* because scanning 40,000 tiny files costs more metadata/listing overhead than the data itself; the "small files problem" swamps the storage layer, and downstream readers (Presto, Hive, BI) grind to a halt.
- **The fix:** partition on low-cardinality, stable keys (date, region); cap files with `coalesce`/`maxRecordsPerFile`; use compaction (Delta `OPTIMIZE`, or periodic rewrites) to keep file counts sane.

### Using RDD When DataFrame Is Better

- **The wrong pattern:** reaching for `map`/`reduceByKey`/`sc.parallelize` for everyday ETL because the RDD API is "lower level and more powerful."
- **Why it looks correct:** RDD is the original API; tutorials from the Spark 1.x era still teach it.
- **Production symptom:** correct results at 3–5× the runtime of an equivalent DataFrame pipeline, plus more Python-object serialization and no pushdown to Parquet (whole files read instead of pruned).
- **The fix:** default to DataFrames/Spark SQL; RDD is justified only for custom low-level algorithms, and even then, profile first.

## Real-World Examples

### Reading Parquet Files, Cleaning Data, Aggregations

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import (
    col, when, coalesce, to_date, regexp_replace, trim, lower, count, avg, sum
)
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, DateType

spark = SparkSession.builder.appName("SalesCleanup").getOrCreate()

# Read a partitioned Parquet table — schema comes from the files
raw = spark.read.parquet("s3://data/raw/sales/")

# Clean: trim/lower text, parse dates, fill NULLs with a fallback
cleaned = raw \
    .withColumn("customer_email", lower(trim(raw["customer_email"]))) \
    .withColumn("product_code", regexp_replace(col("product_code"), r"\s+", "")) \
    .withColumn("sale_date", to_date(col("date_str"), "yyyy-MM-dd")) \
    .withColumn("currency", coalesce(col("currency"), lit("USD"))) \
    .withColumn("amount", col("amount").cast("decimal(18,2)")) \
    .filter(col("amount").isNotNull())          # drop rows where the cast failed

# Validate: count vs original, flag NULL dates
print("rows before:", raw.count())
print("rows after :", cleaned.count())
cleaned.filter(col("sale_date").isNull()).count()   # how many unparseable dates survived

# Aggregate by product/region for a report
report = cleaned.groupBy("region", "product_code") \
    .agg(
        count("*").alias("orders"),
        sum("amount").alias("revenue"),
        avg("amount").alias("avg_order_value"),
    )
```

- The structure to internalize: **read → validate → clean → aggregate → write** — validation between steps is what catches silent NULLs from failed casts/dates before they poison the report.

### Joining Two Large Tables Efficiently

```python
# Large fact (billions of rows) + large dimension (millions) — both "too big" to broadcast
facts  = spark.read.parquet("s3://data/curated/transactions/")      # key: customer_id
dim    = spark.read.parquet("s3://data/curated/customers/")          # key: customer_id

# 1. Filter BOTH sides before joining — shrink at the source
facts  = facts.filter(col("txn_date") >= "2026-01-01")
dim    = dim.select("customer_id", "segment", "country")             # prune unused columns

# 2. Repartition both sides on the join key so the join is shuffle-free
facts = facts.repartition(400, "customer_id")
dim   = dim.repartition(400, "customer_id")

# 3. Join, then aggregate — aggregation AFTER join is fine here; keys are balanced
result = facts.join(dim, "customer_id", how="left") \
    .groupBy("segment", "country") \
    .agg(sum("amount").alias("revenue"))

# 4. Persist the small aggregated result for reuse downstream
result.cache()
```

- The ordering rule: **filter early, project early, broadcast small, repartition on key, aggregate last.** If one side is genuinely small (under ~10 MB, or even up to ~1 GB if executors have room), use `.hint("broadcast")` instead of repartitioning — broadcast eliminates the shuffle completely.
- Verify with `result.explain()` — the physical plan should show the join type you intended (BroadcastHashJoin vs SortMergeJoin) and confirm partition count on the shuffle.
- If keys are skewed (one country = 90% of rows), enable AQE skew join or salt the key with a random suffix before grouping.

### Writing Aggregated Output to a Delta/Snowflake

```python
# --- Writing to Delta (lakehouse) ---
result.write \
    .mode("overwrite") \
    .format("delta") \
    .partitionBy("country") \
    .option("replaceWhere", "country IN ('US', 'CA', 'MX')") \   # overwrite only these partitions
    .save("s3://data/curated/revenue_by_country_delta")

# Delta merge (upsert) for incremental loads — the idempotent pattern
from delta.tables import DeltaTable

delta_tbl = DeltaTable.forPath(spark, "s3://data/curated/revenue_by_country_delta")
delta_tbl.alias("tgt") \
    .merge(result.alias("src"), "tgt.country = src.country AND tgt.segment = src.segment") \
    .whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .execute()

# --- Writing to Snowflake (warehouse) ---
result.write \
    .format("snowflake") \
    .option("sfUrl", "https://acme.snowflakecomputing.com") \
    .option("sfUser", os.environ["SF_USER"]) \
    .option("sfPassword", os.environ["SF_PASSWORD"]) \
    .option("sfDatabase", "ANALYTICS") \
    .option("sfSchema", "PUBLIC") \
    .option("sfWarehouse", "ETL_WH") \
    .option("dbtable", "REVENUE_BY_COUNTRY") \
    .option("truncate", "false") \
    .mode("append") \
    .save()
```

- Delta's `replaceWhere` makes `overwrite` *partition-scoped and atomic* — you don't clobber partitions your pipeline didn't touch, and a failed overwrite doesn't destroy the table. This is the production-grade replacement for naive `overwrite`.
- Snowflake writes are **mode-specific**: `append` stages data via the Snowflake connector's internal stage then copies; if `truncate=true`, it drops the whole table first. Credentials come from env vars, never the code.
- Idempotency check: re-running the Delta merge produces the same final state; re-running a naive `overwrite` of the full path is also idempotent but non-atomic without a transactional format.
- **Scale inflection:** at small scale, `overwrite` + reload is fine. Once other pipelines read your output or the load is incremental (daily updates), you need Delta merge / partition-scoped overwrite — because full overwrites either race with readers or touch files you didn't change.

## Interview Questions

### Spark vs Hadoop

- **What is the difference between Apache Spark and Hadoop MapReduce?**
  - Spark keeps intermediate data in memory across stages; MapReduce writes every intermediate result to disk between map and reduce.
  - Spark's DAG scheduler can run a job with many dependent steps as one execution (in-memory chaining); MapReduce forces a separate disk round-trip per step.
  - Consequence: for iterative algorithms (ML, graph) and interactive analytics, Spark is commonly 10–100× faster than MapReduce on the same hardware.
  - Trade-off: Spark needs enough RAM on executors (memory is the constraint), while MapReduce's disk-bound model is more forgiving of small memory but slower.
  - Modern framing: Spark is the successor — YARN can even run Spark *alongside* legacy MapReduce on the same Hadoop cluster, which is why "Spark vs Hadoop" is really "Spark vs the MapReduce engine."

### Transformations vs Actions

- **What is the difference between a transformation and an action in Spark?**
  - Transformations are lazy: they return a new DataFrame/RDD describing a plan (e.g., `filter`, `select`, `groupBy`, `join`) and do no computation.
  - Actions trigger execution: `count()`, `collect()`, `show()`, `take()`, `write` all force the plan to compile and run.
  - Interview depth: because Spark is lazy, it can optimize — push filters down to storage, prune columns, reorder joins, fuse stages — before a byte is read. The price is that errors surface at the action, not at the line where the bug was written.
  - Follow-up to expect: "You have a DataFrame read → filter → join → write. At what point does the file get read?" Answer: at `write` (the action), all at once, via the optimized plan.

### Lazy Evaluation

- **Explain lazy evaluation in Spark. Why is it useful?**
  - Lazy evaluation means transformations build a logical plan and nothing executes until an action demands a result.
  - Benefits: whole-pipeline optimization (filters pushed into readers, unused columns pruned, joins reordered), cheaper failure (schema/logic errors fail before expensive work), and one pass over data instead of one per operation.
  - Cost to acknowledge: debugging is indirect — a transformation bug shows up at the action, and two actions on the same un-cached DataFrame re-run the whole lineage from scratch.

### DataFrame vs RDD

- **When would you use a DataFrame instead of an RDD, and vice versa?**
  - DataFrames have a schema, so Catalyst can optimize them (predicate pushdown, projection pruning, plan rewrites) and Tungsten can execute them in a compact binary format.
  - RDDs are untyped and unoptimized — every element is a raw Python object — so they're slower for the same logical work and don't get storage pushdown.
  - Rule: DataFrames/SQL almost always. RDDs only for custom low-level algorithms or interop with libraries that need them; even then, profile before concluding a UDF-on-DataFrame is worse.
  - Interview bonus: a DataFrame is *backed by* an RDD — `df.rdd` still exists. The optimization story is "catalyst re-writes the plan; the plan, not the code, is what runs."

### Shuffle

- **What is a shuffle in Spark, and why is it expensive?**
  - A shuffle redistributes data across executors, required by operations like `groupBy`, `join` (non-broadcast), `distinct`, `orderBy`, and `repartition`.
  - It's expensive because each partition's rows are serialized, written to local disk, pulled by remote executors over the network, and re-read — disk I/O + network I/O + serialization, often 60–90% of job time.
  - How to reduce: filter/aggregate before joining, broadcast small tables, bucket/repartition on join keys, and let AQE coalesce tiny shuffle partitions.
  - Follow-up to expect: "Your join stage shows a huge shuffle read. Walk me through your debugging sequence." (Check the UI → check key skew → check broadcast eligibility → check partitioning.)

### Broadcast Join

- **How does a broadcast join work? When is it appropriate?**
  - Spark copies the small table to every executor's memory and joins it locally against each partition of the large table — no shuffle, no network data movement for the join itself.
  - Appropriate when one side is small enough to fit in executor memory comfortably — by default under `spark.sql.autoBroadcastJoinThreshold` (10 MB), though bigger tables work with more RAM.
  - Failure mode: a "small" table that's actually large gets copied per executor — 100 executors × 5 GB = 500 GB and executor OOM. When the table stops fitting, the broadcast becomes the bottleneck, not the join.
  - Spark 3 also promotes joins to broadcast at runtime via AQE if a side turns out smaller than expected.

### Partitioning

- **Explain partitioning in Spark. How does it affect performance?**
  - Partitioning is the physical split of a DataFrame into chunks processed in parallel — one task per partition, and the partition count is the parallelism of a stage.
  - Good partitioning = work balanced across cores; bad partitioning = idle cores, small files on write, or a single hot partition (skew) dominating the stage.
  - Two separate ideas to not conflate: runtime partitioning (`repartition`, `coalesce`, and shuffle partition counts) vs storage partitioning (`partitionBy` directory layout for pruning on read).
  - Tuning: more partitions than cores wastes scheduler overhead; too few underutilizes the cluster. AQE's dynamic coalescing removes most of this tuning in modern Spark.

### Catalyst

- **What does the Catalyst optimizer do?**
  - Catalyst turns your logical plan (what you wrote) into an optimized physical plan (what actually runs) through rule-based rewrites: predicate pushdown, projection pruning, constant folding, join reordering, and broadcasting small tables.
  - It works in phases: analysis → logical optimization → physical planning (choosing join algorithms like sort-merge vs broadcast hash) → Tungsten code generation.
  - Why it matters: you write readable code and Catalyst makes it fast; the query that runs is not the query you wrote. Use `df.explain(true)` to see both.

### Coalesce vs Repartition

- **Difference between `coalesce` and `repartition`?**
  - `repartition(n)` always does a full shuffle and can both increase and decrease partition count, evenly rebalancing data.
  - `coalesce(n)` decreases partitions by merging adjacent ones — no full shuffle (when n < current), so it's much cheaper but can leave data unevenly distributed.
  - Classic misuse: `coalesce(1)` to write one file collapses the job to one task — a long, single-threaded write. Coalesce to a few partitions that still parallelize, or control output files with `maxRecordsPerFile`.
  - Rule of thumb: use `repartition` when you need a rebalance or partition-by-key; use `coalesce` only for the cheap "reduce partition count" case, and never to 1.

### Cache vs Persist

- **Difference between `cache()` and `persist()`?**
  - `cache()` is `persist(StorageLevel.MEMORY_AND_DISK)` — shorthand.
  - `persist(level)` lets you choose the level: `MEMORY_ONLY`, `MEMORY_AND_DISK`, `DISK_ONLY`, `MEMORY_AND_DISK_SER`, `OFF_HEAP`.
  - Both are lazy — nothing is cached until the first action runs on the DataFrame.
  - When it helps: a DataFrame consumed by multiple downstream actions or a loop. When it hurts: single-use DataFrames — you pay extra memory for zero reuse, and over-caching pushes out other data and causes spills.
  - Interview depth: cache the *narrow, reused* results (post-aggregation), not wide raw reads, and `unpersist()` when done.

### When to Use PySpark

- **When should you use PySpark instead of Pandas (or vice versa)?**
  - PySpark: data that doesn't fit on one machine, production/regular ETL at scale, distributed joins/aggregations, streaming, and pipelines that must be repeatable and parallel.
  - Pandas: interactive exploration, feature engineering, and analytics on data that fits in RAM — with richer functions, vectorized speed on one core, and zero cluster overhead.
  - Decision rule: if it fits in Pandas and runs fast enough, use Pandas; the moment data stops fitting or you need a distributed, scheduled job, move to PySpark.
  - Common pattern worth naming: PySpark to aggregate/clean at scale, then `.toPandas()` on the small result for exploration or ML.

### Spark Session vs Spark Context

- **Difference between SparkSession and SparkContext?**
  - `SparkContext` is the low-level engine entry point: cluster connection, RDD API, broadcast variables, accumulators.
  - `SparkSession` (Spark 2.0+) is the unified entry point wrapping the context plus SQL, Hive, and streaming — the only object you normally create.
  - You still access `spark.sparkContext` for engine-level control, but day-to-day DataFrame/SQL/streaming work goes through the session.
  - Legacy footnote: before Spark 2.0, PySpark code juggled separate `SQLContext`/`HiveContext` — that's why old tutorials look different.

### Skew and Data Skew

- **What is data skew, and how do you fix it?**
  - Skew is an uneven distribution of keys: one partition holds a disproportionate share of rows (e.g., 80% of sales belong to one country), so that partition's task runs for hours while others idle.
  - Symptoms: one task in a stage far outlives all others; the Spark UI shows a lopsided shuffle read.
  - Fixes: enable AQE skew join (auto-splits hot partitions), salt the key (append a random suffix before grouping/joining, then aggregate it away), or broadcast the small side if that removes the shuffle entirely.
  - Root cause to check first: the join/groupBy key itself — sometimes a NULL key or a missing-join-value bucket (all unmatched rows under one "null" key) is the skew, and handling NULLs fixes it.

### UDFs and Performance

- **Why are Python UDFs slow, and when should you use them anyway?**
  - A scalar UDF serializes each row across the JVM↔Python boundary, executes Python (GIL included), and serializes back — the opposite of Tungsten's code generation. Typically tens to hundreds of times slower than a built-in expression.
  - Prefer built-in functions (`when`, `coalesce`, string/date functions); if a UDF is unavoidable, use a **Pandas/Arrow vectorized UDF** so a batch of rows crosses the boundary at once instead of one at a time.
  - Honest answer for when a UDF is right: logic that genuinely can't be expressed columnar (complex custom algorithms, calling third-party Python libraries per record). In those cases, minimize the rows fed to it — filter first.

### Exactly-Once in Streaming

- **How does Structured Streaming guarantee exactly-once processing?**
  - Through the **checkpoint** (persisted offsets + state + in-flight batch metadata) plus the sink's transactionality.
  - The source (e.g., Kafka) offsets are saved atomically with the sink write: on restart, the stream resumes from the last *committed* offset, so nothing is replayed (no duplicates) and nothing was dropped (no loss).
  - Constraint: exactly-once depends on the sink supporting atomic writes (Kafka, Delta). With an idempotent sink and `outputMode("update")`, re-processing a batch produces the same result.
  - Follow-up to expect: "What happens if you delete the checkpoint?" — At-least-once at best; likely duplicate/lost data because offset state is gone.

### Explain Plan

- **How do you debug a slow Spark job?**
  - Start with the Spark UI: look at the DAG, stage durations, shuffle read/write sizes, and any disk spill — those three (shuffle size, skew, spill) explain most slow jobs.
  - Read the physical plan: `df.explain(true)` — check the join algorithm (Broadcast vs SortMerge), verify partition counts, and confirm filters were pushed into the reader.
  - Then the checklist: filter before join? small table broadcast? partitions right-sized (or AQE left on)? caching only reused frames? skew present? serialization via UDFs?
  - Never tune blindly — every change should be motivated by something visible in the UI or the plan.

### Spark 3 Optimizations

- **What are the major Spark 3 performance features?**
  - **Adaptive Query Execution (AQE)** — runtime re-optimization: dynamic coalescing of shuffle partitions, dynamic broadcast promotion, dynamic skew join handling.
  - **Dynamic Partition Pruning** — push a dimension's filter down to prune fact partitions before the join executes.
  - **Better ANSI compliance + typed literals** — cleaner semantics, though ANSI mode can change behavior of existing queries (check before enabling globally).
  - Python side: Arrow-optimized columnar UDFs and `toPandas()` for efficient serialization.
  - Interview point: these are mostly *on by default* — the modern answer to "tune shuffle partitions" is "let AQE do it, then verify in the UI."

### Where to Check / What Happens When

- **What happens when you call `df.show()` on a DataFrame that was created but never materialized?**
  - It triggers a full job: the lazy plan compiles, the DAG is scheduled, and tasks run across the cluster before the 20 rows are printed. It is an action.
  - Implication: a notebook with many `show()` calls re-executes the whole pipeline each time — harmless on small test data, painful on production data.
  - Related trap: calling `show()` then `count()` then `write` on the same uncached DataFrame runs the pipeline three times.
