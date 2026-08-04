# Apache Airflow Interview Notes

Comprehensive notes on Apache Airflow for data engineering interviews. Covers core concepts, architecture, DAGs, operators, executors, scheduling, monitoring, best practices, real-world pipelines, common mistakes, and interview questions.

---

## What is Airflow

### Overview
- **Apache Airflow** is an open-source **workflow orchestration** platform used to programmatically author, schedule, and monitor workflows.
- Workflows are defined as **code** (Python), which makes them versionable, testable, and maintainable — a key advantage over GUI-only tools.
- Originally created by **Airbnb** in 2014 to manage their increasingly complex scheduling needs. It was open-sourced in 2015 and became a **top-level Apache Software Foundation project** in 2019.
- Core job: **schedule and run tasks in the right order**, handle failures, retries, dependencies, and provide a UI to observe everything.
- Airflow is **not** a data processing framework — it does not compute or transform data itself. It **orchestrates** tools that do (Spark, dbt, Databricks, Snowflake, etc.).

### Key Concepts (the "must-know" vocabulary)
- **DAG (Directed Acyclic Graph)**: a collection of tasks with defined dependencies. Direction shows execution order, "acyclic" means no cycles — a task can never depend on itself (directly or transitively).
- **Task**: the smallest unit of work in a DAG (an operator instantiated with parameters).
- **Operator**: a template/class that defines *what* a task does (run a bash command, execute Python, run SQL).
- **Scheduler**: the component that decides *when* tasks should run, based on schedules and dependencies.
- **Worker**: the component that actually *executes* tasks.
- **Executor**: defines *how* tasks are run (sequentially, in parallel on one box, distributed, in Kubernetes pods).
- **Web server / UI**: graphical interface to inspect DAGs, trigger runs, view logs, and manage connections/variables.
- **Metadata database**: stores all state — DAG definitions, task instances, run history, variables, connections.

### Airflow vs Cron Jobs
Cron works fine for simple cases but fails on realistic pipelines:

| Concern | Cron | Airflow |
|---|---|---|
| Dependencies | None — every job is independent | Explicit, with ordering and branching |
| Retries | Not built-in; you hand-roll retry logic | Native retries with exponential backoff |
| Monitoring | You hack together logging/mail | Built-in UI, logs, SLAs, alerting |
| Failure handling | Job fails silently or spams email | Retries, alerts, email/slack/PagerDuty |
| Backfill | Very hard to run historical jobs | Built-in `backfill` and `catchup` |
| Parameterization | None | Variables, `conf`, templating |
| State/history | None | Every run recorded in metadata DB |
| Scalability | Local only | Distributed executors (Celery/K8s) |

- **Rule of thumb**: single command that runs on a schedule with no dependencies → cron is fine. Anything with dependencies, retries, data lineage, or monitoring needs → Airflow.

### Airflow vs dbt
- Airflow = **orchestrator** (schedules and coordinates when things run, in what order, with retries and monitoring).
- dbt = **transformation tool** (turns SQL models into tables/views in the warehouse, handles incremental logic, tests, docs).
- They are **complementary**, not competitors. The standard modern pattern:
  - Airflow runs dbt as a task (via `DbtRunOperator` or `DbtTaskGroup`).
  - dbt handles the transformation logic (SQL), Airflow handles the "when/how/who retries" logic.
- Quick distinction: **Airflow schedules the pipeline; dbt transforms the data.**

### Airflow vs Dagster vs Prefect (brief comparison)
- **Airflow**: most mature, largest community, biggest ecosystem, default "boring but safe" choice in enterprises. Heavier scheduling semantics (execution date vs data interval confusion), DAGs are static.
- **Dagster**: emphasizes **software-defined assets** and data-aware scheduling — assets and their dependencies are first-class, better lineage and testing. Strong for data teams that think in terms of datasets, not pipelines.
- **Prefect**: focuses on **developer experience**, dynamic DAGs, easier local dev, decorator-based flows, flexible scheduling. Popular with smaller teams and rapid prototyping.
- **When to pick which**: enterprise stability/community → Airflow; asset/lineage-centric data platform → Dagster; DX and flexibility → Prefect.

---

## Airflow Architecture

### High-Level Components
Airflow is a **multi-service** system. The main pieces are:

```
                    ┌─────────────────────┐
                    │    Web Server (UI)  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐      ┌──────────────────┐
                    │  Scheduler          │◄────►│  Metadata DB     │
                    │  (plans, triggers)  │      │  (Postgres/MySQL)│
                    └──────────┬──────────┘      └──────────────────┘
                               │ (via executor → queue)
                    ┌──────────▼──────────┐
                    │   Message Queue     │  e.g. Redis / RabbitMQ
                    │  (CeleryExecutor)   │
                    └──────────┬──────────┘
                    ┌──────────▼──────────┐
                    │     Workers         │  execute tasks
                    └─────────────────────┘
```

### Scheduler
- **Responsibilities**:
  - Parses the `dags/` folder to discover DAG definitions.
  - Decides which DAG runs to create based on `schedule_interval`.
  - Decides which task instances are *runnable* (dependencies satisfied, within concurrency limits).
  - Submits runnable tasks to the executor and marks them as `queued`.
  - Handles retries, timed-outs, and SLA checks.
- **Key insight**: the scheduler is a **stateless-ish planner**, not a data processor. It should never do heavy work — heavy work belongs in tasks.
- Runs periodically (default `SCHEDULER_HEARTBEAT_SEC = 5` seconds) to re-evaluate the world.

### Worker
- Executes tasks that the scheduler has submitted via the executor.
- Each executed task creates a **TaskInstance (TI)** — one specific run of one task inside one DAG run.
- In distributed setups, workers run on separate machines and pull work from the queue.

### Web Server (UI)
- Flask-based app that renders the Airflow UI (tree view, graph view, Gantt, calendar, logs, variables, connections).
- **Reads** state from the metadata DB — it is largely a presentation layer.
- Provides manual triggers, DAG toggles, and admin screens for connections/variables/pools.

### Metadata Database
- Stores everything: DAG definitions, DAG runs, task instances, their states, logs references, connections (encrypted), variables, pools, user accounts.
- Default is **SQLite** (for dev/testing only). Production uses **PostgreSQL** or **MySQL**.
- **Critical**: the metadata DB is a single point of failure. Run it with backups and proper HA in production.
- Recommended to run the **DB and scheduler on separate machines** from workers for production-grade setups.

### Message Queue (Redis / RabbitMQ)
- Used **only** when running with the **CeleryExecutor**.
- Acts as a broker: the scheduler (via the executor) publishes "run this task" messages; Celery workers consume and execute them.
- The queue decouples the scheduler from the workers, enabling horizontal scaling of workers.

### Executors and How They Fit
- An **executor** is the mechanism the scheduler uses to actually run tasks. It is a pluggable component configured in `airflow.cfg` or via env vars.
- The scheduler submits tasks to the executor; the executor decides the physical execution strategy (local process, thread pool, celery worker, pod).

### Components Diagram Explanation
1. **Scheduler** reads DAGs from the `dags/` folder and the DB → computes what should run.
2. Scheduler hands runnable task instances to the **executor**.
3. Executor (e.g., Celery) pushes the task to a **queue** (Redis/RabbitMQ).
4. A **worker** pulls the task, executes it, and writes the result/state back to the **metadata DB**.
5. The **web server** reads the DB to show you the live state in the UI.

---

## DAGs

### What is a DAG
- **Directed Acyclic Graph**: a set of tasks connected by directed edges (dependencies) that form an acyclic structure.
- Think of it as a **blueprint** of the workflow. It is *declarative*: you define structure and scheduling; Airflow decides when individual runs execute.
- One DAG file can contain **one or more** DAGs, but the convention is one logical pipeline per DAG (and usually one DAG per file).

### DAG Definition in Python

#### Classic style (DAG class)
```python
from datetime import datetime, timedelta

from airflow import DAG
from airflow.operators.python import PythonOperator

def extract():
    print("Extracting data...")

def transform():
    print("Transforming data...")

default_args = {
    "owner": "data-team",
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
    "email_on_failure": True,
    "email": ["data-eng@company.com"],
}

with DAG(
    dag_id="etl_demo",
    default_args=default_args,
    description="Simple ETL demo",
    schedule_interval="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["etl"],
) as dag:
    t1 = PythonOperator(task_id="extract", python_callable=extract)
    t2 = PythonOperator(task_id="transform", python_callable=transform)
    t1 >> t2
```

#### TaskFlow style (recommended for new code)
```python
from datetime import datetime

from airflow.decorators import dag, task

@dag(
    dag_id="etl_demo_taskflow",
    schedule="@daily",
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["etl"],
)
def etl_demo_taskflow():
    @task
    def extract() -> dict:
        return {"rows": 1000}

    @task
    def transform(data: dict) -> dict:
        return {"processed": data["rows"]}

    @task
    def load(data: dict):
        print(f"Loading {data['processed']} rows")

    load(transform(extract()))

etl_demo_taskflow()
```

### DAG Parameters
- **`dag_id`**: unique identifier of the DAG. Must be unique across the deployment. Convention: `snake_case`, descriptive (e.g., `daily_sales_etl`).
- **`schedule_interval`** / **`schedule`**: how often the DAG runs (see cron section below).
- **`start_date`**: the *logical* start date; defines the anchor for the schedule and the earliest date the DAG may run for. It does **not** mean "run once now".
- **`catchup`**: if `True`, the scheduler creates DAG runs for every interval between `start_date` and now that was missed. Default `True` (dangerous with far-past start dates) — set to `False` unless you want a backfill.
- **`max_active_runs`**: max number of DAG runs that can be *simultaneously active* (default 16). Protects against overlapping runs of slow pipelines.
- **`max_active_tasks`**: max tasks running in parallel across all runs of this DAG.
- **`concurrency`** (deprecated in 2.x, superseded by `max_active_tasks`).
- **`tags`**: labels for UI filtering.
- **`description`**: short human-readable summary.
- **`catchup` vs `backfill`**: `catchup` = automatic creation of missed runs when the scheduler restarts; `backfill` = explicit command `airflow dags backfill <dag_id> --start-date ... --end-date ...`.

### default_args
- A dict of **default values applied to every operator/task** in the DAG, unless overridden at the task level.
- Common entries:
  - `retries`: number of automatic retries on failure.
  - `retry_delay`: time to wait between retries (`timedelta`).
  - `retry_exponential_backoff`: if `True`, delays grow exponentially.
  - `max_retry_delay`: cap on the backoff delay.
  - `depends_on_past`: if `True`, a task only runs if the *previous DAG run's* instance of that task succeeded. Useful for strict incremental pipelines, but risky (a failed run blocks the future forever until fixed).
  - `email_on_failure` / `email_on_retry`: send email on those events.
  - `email`: list of addresses.
  - `owner`: who owns the DAG (shown in UI, used for DAG-level permissions).
  - `start_date`, `end_date`.
  - `queue`, `pool`, `priority_weight`: execution-related defaults.

### DAG File Structure
- DAG files live in the **`dags/` folder** (or subfolders) configured by `dags_folder` in `airflow.cfg`.
- The scheduler **re-parses** these files periodically (default `min_file_process_interval = 30s`).
- Every `.py` file in the folder is imported by the scheduler — so **keep DAG files fast and dependency-light** (see "Heavy logic in the scheduler" in Common Mistakes).
- Naming: `dag_id` (not the filename) is what matters for the UI. Convention: filename matches `dag_id`.
- DAG files should be **static** — the structure must be identical every time it's parsed, or you get "DAG seems to be missing" warnings and schedule weirdness.

### Why a DAG Must Be Idempotent and Static
- **Idempotent**: re-running the DAG (retry, backfill, catchup) must produce the *same correct result* as the first run. No duplicate rows, no double counting. Achieved with `INSERT ... ON CONFLICT`, `DELETE + INSERT` keyed on the business date, or "delete the data interval then re-run".
- **Static**: the DAG graph should not change based on runtime data (e.g., no `if` loops that create tasks depending on a DB query result at parse time). Schedules and structure are defined once at parse time. If the graph must depend on data, that's a sign to split the DAG or use dynamic task mapping (2.3+).

### schedule_interval vs schedule (cron expressions)
- `schedule_interval` accepts:
  - Cron presets: `@daily`, `@hourly`, `@weekly`, `@monthly`, `@yearly`, `@once`.
  - `None`: DAG only run manually/triggered externally.
  - A cron string: `"0 8 * * *"` (every day at 08:00), `"0 */6 * * *"` (every 6 hours), `"30 2 * * 1-5"` (weekdays 02:30).
- Airflow 2.4+ introduced **timetables** and `schedule` as an alias; cron strings remain the most common.
- Important nuance: with a cron schedule, a run for interval *N* is triggered **after** the interval *N* ends. `0 8 * * *` with the run on Aug 3 at 08:00 actually covers data for **Aug 2** (its `execution_date` is Aug 2). This trips everyone up — see Data Interval.

### catchup and backfill
- **catchup**:
  ```bash
  # In DAG: catchup=False means "only create the current/most recent run"
  with DAG(..., start_date=datetime(2020, 1, 1), schedule_interval="@daily", catchup=False):
      ...
  ```
  - With `catchup=True`, turning on a DAG with a 2020 `start_date` in 2024 creates ~4 years of runs immediately. Usually not what you want.
- **backfill** (explicit, on-demand historical runs):
  ```bash
  airflow dags backfill my_dag --start-date 2024-01-01 --end-date 2024-01-10
  ```
  - Runs the DAG for each interval in the range, honoring task dependencies and retries.
  - Useful to populate a fresh warehouse or fix a failed window.

---

## Tasks and Operators

### What is a Task
- A task = an **instantiated operator** with an `task_id`, parameters, and a place in the graph.
- When a DAG run executes a task, we get a **TaskInstance** — the run record with its own state.
- Rules: every task has a unique `task_id` within the DAG; tasks can be given retries, timeouts, pools, queues.

### Operators (built-in, and how they map to "what to run")
| Operator | Purpose |
|---|---|
| `BashOperator` | Run any shell command/script |
| `PythonOperator` | Run a Python callable |
| `EmptyOperator` | Logical grouping/pass-through (no work) |
| `BranchPythonOperator` | Choose downstream path based on Python logic |
| `EmailOperator` | Send an email |
| `SQLExecuteQueryOperator` | Run a single SQL statement against a DB |
| `SnowflakeOperator` / `SnowflakeSqlApiOperator` | Run SQL on Snowflake (provider: `apache-airflow-providers-snowflake`) |
| `PostgresOperator` | Run SQL on Postgres (provider) |
| `S3Operator` / `S3ToRedshiftOperator` | S3 operations (provider: `apache-airflow-providers-amazon`) |
| `DatabricksRunNowOperator` | Trigger a Databricks job/notebook (provider) |
| `DbtRunOperator` / `DbtTaskGroup` | Orchestrate dbt runs (provider: `airflow-dbt-python` or `dbt-airflow`) |
| `DockerOperator` | Run a task inside a Docker container |
| `KubernetesPodOperator` | Run a task in a Kubernetes pod |

Example:
```python
from airflow.operators.bash import BashOperator

run_script = BashOperator(
    task_id="run_cleanup_script",
    bash_command="python /opt/scripts/cleanup.py",
    retries=1,
)
```

### Sensors (wait until a condition is true)
- A sensor is a special operator that **polls until a condition is met**, then succeeds (or times out).
- Common sensors:
  - `ExternalTaskSensor`: wait for a task in *another* DAG to finish (cross-DAG dependency).
  - `S3KeySensor`: wait for a file/prefix to appear in S3.
  - `FileSensor`: wait for a file on the filesystem.
  - `ExternalTaskSensor`: wait for another DAG's task.
  - `HttpSensor`: poll a URL until it returns a valid status.
  - `TimeSensor`: wait until a given datetime.
- Sensor parameters: `timeout` (seconds before it fails), `poke_interval` (seconds between checks), `mode` (`poke` = blocking poll, `reschedule` = release the slot between checks).
- Prefer **`mode="reschedule"`** for long waits so you don't hold a worker slot the whole time.

```python
from airflow.sensors.external_task import ExternalTaskSensor

wait_for_landing = ExternalTaskSensor(
    task_id="wait_for_ingest",
    external_dag_id="ingest_dag",
    external_task_id="load_raw",
    timeout=3600,
    poke_interval=60,
    mode="reschedule",
)
```

### Task Dependencies (`>>` and `<<`)
- `>>` means "depends on" (left runs before right).
  ```python
  extract >> transform >> load
  # equivalent: transform.set_upstream(extract); load.set_upstream(transform)
  ```
- `<<` is the reverse: `load << transform` means transform runs before load.
- Fan-out and fan-in:
  ```python
  start >> [task_a, task_b, task_c] >> end        # parallel after start, converge at end
  [t1, t2] >> t3 >> [t4, t5]
  ```
- Dependencies create the *graph*; they also drive the scheduler's "is this task runnable?" logic.

### TaskFlow API (new style) vs Classic
- **Classic**: instantiate operators and chain them. Data passed via `xcom_push`/`xcom_pull` explicitly.
- **TaskFlow (2.0+)**: use `@task` decorators; function **inputs/outputs are automatically passed via XComs** without manual push/pull.
- TaskFlow features:
  - Automatic XCom serialization of return values.
  - Type hints in signatures act as documentation + validation (with `airflow.typing_compat`).
  - Supports `@dag` decorator on the top-level function.
  - Multiple outputs via returning a tuple/dict (with `multiple_outputs=True`).
- TaskFlow + classic can coexist in one DAG (a TaskFlow task can be used in `>>` chains like any operator).
- Dynamic Task Mapping (2.3+): run a task once per item in a list:
  ```python
  @task
  def process_file(path: str): ...
  @task
  def list_files() -> list[str]: ...

  files = list_files()
  process_file.expand(path=files)
  ```

### Task Groups
- `TaskGroup` visually groups tasks in the UI (nested folders in Graph view) without changing execution semantics.
- Useful for pipelines with many repeated sub-steps (e.g., a per-table load group).

```python
from airflow.utils.task_group import TaskGroup

with TaskGroup(group_id="extract_layer") as extract_group:
    t1 = PythonOperator(task_id="extract_a", python_callable=lambda: None)
    t2 = PythonOperator(task_id="extract_b", python_callable=lambda: None)

start >> extract_group >> end
```

### XComs (cross-communication)
- **XCom** = "cross-communication": a mechanism for tasks to **exchange small bits of data**.
- Each XCom is a `(dag_id, task_id, key, timestamp)` row in the metadata DB.
- Usage:
  ```python
  # classic push/pull
  def my_func(ti):
      ti.xcom_push(key="rows", value=42)

  def read_func(ti):
      rows = ti.xcom_pull(task_ids="my_func", key="rows")
  ```
  With TaskFlow, return values are auto-pushed and inputs auto-pulled.
- **Limitations (critical for interviews)**:
  - XComs are stored in the **metadata database** → not for large data. Push small values (IDs, counts, file paths, config).
  - Default max XCom size may be limited (depends on DB); huge payloads bloat the DB and slow the scheduler.
  - For real data exchange use external systems: write to S3/GCS and pass the path, write to a table, or use a temp file.
  - Options to persist larger data: `enable_xcom_pickling` (legacy), custom `xcom_backend` (e.g., S3-backed).

### Task Retries and Retry Logic
- Controlled by `retries`, `retry_delay`, `retry_exponential_backoff`, `max_retry_delay`.
- State during retries: `up_for_retry` (was `up_for_retry`, not counted as failure yet).
- The **scheduler** (not the worker) handles scheduling retries after the delay.
- Retry vs re-run: retries are automatic; a human can also **clear** a TaskInstance to re-run it.
- Best practice: set `retries >= 2` for production tasks, and design tasks so a retry is safe (idempotency).

---

## Executors

### SequentialExecutor
- Runs **one task at a time** in a single process.
- Default executor; **for testing only**.
- Not parallel at all — will not scale past simple demos. Uses the local DB for queuing.

### LocalExecutor
- Runs tasks **in parallel on a single machine** using subprocesses.
- Good for small/medium workloads where one beefy machine is enough.
- Uses a DB-backed queue internally to track queued tasks.

### CeleryExecutor
- **Distributed**: scheduler pushes tasks to a broker queue (**Redis** or **RabbitMQ**); a pool of **Celery workers** on multiple machines consume and execute.
- Scales horizontally by adding workers.
- Adds operational complexity: you must run and maintain the message broker and celery workers.
- Best for organizations that need many parallel tasks without Kubernetes.

### KubernetesExecutor
- **Every task runs in its own Kubernetes Pod**.
- Pod is created on task execution and destroyed after — strong isolation, per-task resources (CPU/memory/GPU), and clean scaling.
- The scheduler talks to the K8s API; pods pull the image and run the task.
- Excellent isolation and resource control; more infrastructure to manage (a K8s cluster).
- `kubernetes_queue` and `KubernetesPodOperator` share this philosophy.

### How to Choose an Executor
- Testing/demo → **SequentialExecutor** (or LocalExecutor).
- Single machine, moderate parallelism → **LocalExecutor**.
- Multiple machines, need horizontal scaling of workers, already use Redis/RabbitMQ → **CeleryExecutor**.
- Already on Kubernetes, want per-task isolation/resources → **KubernetesExecutor**.
- Consideration order: team ops skills → infrastructure available → parallelism needed → isolation required.

### Airflow 2.0+ TaskFlow API (executor-agnostic reminder)
- TaskFlow is about *authoring* (not execution); executors are about *running*. The two compose: you can write TaskFlow DAGs and run them on any executor.
- Airflow 2.x also added: stable REST API, scheduler HA with multiple schedulers, timezone-aware scheduling, deferrable operators (2.2+), dynamic task mapping (2.3+), and a better UI.

---

## Hooks and Connections

### What are Hooks
- A **hook** is an interface (a wrapper/abstraction) to an **external system**: databases, cloud APIs, services.
- Hooks are used *inside* operators to actually talk to the external system.
- They also provide:
  - **Connection management**: reading credentials from the Airflow Connections store.
  - **Connection reuse/pooling**: hooks reuse connections and provide connection close/cleanup.
  - **Standard API**: `get_conn()`, `run(sql)`, etc.
- You can use hooks directly in PythonOperator callables (common pattern for custom logic):

```python
from airflow.providers.postgres.hooks.postgres import PostgresHook

def query_helper():
    hook = PostgresHook(postgres_conn_id="analytics_db")
    conn = hook.get_conn()
    with conn.cursor() as cur:
        cur.execute("SELECT count(*) FROM orders")
        print(cur.fetchone())
```

### Setting Up Connections in the Airflow UI
- Go to **Admin → Connections → Add**.
- Fields: `conn_id` (unique name used in code), `conn_type` (e.g., postgres, snowflake, http, s3, slack), `host`, `schema`, `login`, `password`, `port`, `extra` (JSON for provider-specific settings like account, region, warehouse).
- Connection IDs are the **only thing you reference in DAG code** — credentials themselves never appear in DAG files.
- Can also be defined in `airflow.cfg`, via env vars (`AIRFLOW_CONN_<CONN_ID>`), or the CLI (`airflow connections add`).
- In Airflow 2.x, connection passwords are **encrypted** in the DB using the Fernet key configured in `airflow.cfg`.

### Common Hooks
- `SnowflakeHook` (from `apache-airflow-providers-snowflake`) — used by `SnowflakeOperator`.
- `PostgresHook` / `RedshiftSQLHook` — for Postgres/Redshift.
- `HttpHook` — generic REST calls (used by `SimpleHttpOperator`).
- `S3Hook` — S3 object operations.
- `GCSHook`, `BigQueryHook`, `AzureBlobStorageHook` — cloud-specific.
- `DockerHook`, `SlackHook`, `DbtHook`.

### Using Hooks in Operators (pass a connection id)
- Every provider operator takes a `conn_id` parameter and internally uses the matching hook:
```python
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator

run_snowflake_sql = SnowflakeOperator(
    task_id="load_to_snowflake",
    snowflake_conn_id="snowflake_analytics",
    sql="""
        COPY INTO analytics.orders
        FROM @STAGE.orders_csv
        FILE_FORMAT = (TYPE = CSV, SKIP_HEADER = 1);
    """,
)
```

### How Connections Store Credentials
- Stored in the **metadata DB** (table `connection`), password field **encrypted** with the Fernet key.
- The `extra` field holds provider-specific JSON (e.g., Snowflake `account`, `warehouse`, `database`).
- Accessible in DAGs via `{{ conn.<conn_id> }}` in templates or `BaseHook.get_connection()` in code.
- **Never commit credentials to the repo.** Use Connections, Vault, or env vars, and restrict who can edit connections.

---

## Variables and Pools

### Airflow Variables
- Key-value store in the metadata DB, editable via UI (Admin → Variables), CLI, or REST API.
- Used for **configuration values**: target table names, thresholds, feature flags, paths.
- Access in code:
  ```python
  from airflow.models import Variable

  table_name = Variable.get("target_table", default_var="default_table")
  # in templates: {{ var.value.target_table }}
  # JSON variable: {{ var.json.my_config }}
  ```
- Notes: cached after first access; keep secrets out of Variables if avoidable (they show in UI unless hidden). For secrets prefer **Connections or a Secret backend** (e.g., AWS Secrets Manager, Vault).

### Pools
- **Pools limit the parallelism of tasks**: only `slot_count` tasks from the pool can run concurrently.
- Use pools to:
  - Limit concurrent load on an external system (e.g., max 5 tasks hitting Snowflake at once).
  - Reserve capacity for critical tasks.
  - Prevent DAGs from exhausting all worker slots.
- Define in UI (Admin → Pools) or CLI. Assign with `pool="my_pool"` in the task or `default_pool` in DAG defaults.
- If all slots are taken, tasks wait in `queued` state.
- The `default_pool` has 128 slots and is used when no pool is specified (configurable).

```python
heavy_task = PythonOperator(
    task_id="heavy_transform",
    python_callable=do_heavy_work,
    pool="snowflake_pool",
    priority_weight=10,
    queue="high_prio",
)
```

### Queues
- Tasks can be tagged with a `queue`; workers can be started with `--queues` to only pull from specific queues.
- Enables routing work to specialized workers (e.g., GPU workers, memory-heavy workers).
- With CeleryExecutor: `queue="gpu_queue"` on a task, and start some workers with `celery worker -q gpu_queue`.

### Variables vs Environment Variables
- **Airflow Variables**: dynamic, editable at runtime, visible in UI, live in DB, accessible in templates — good for things that change without a redeploy.
- **Environment variables**: set at deployment/config time, better for static per-environment config and **secrets** (can be injected from the orchestrator/Vault), no UI exposure.
- Recommendation: **secrets → Connections or env vars/secret backends**; dynamic tunables → Variables; deploy-time config → env vars.

---

## Scheduling Concepts

### Schedule Interval, Execution Date, Logical Date
- **`schedule_interval`**: the cadence (cron or preset).
- **`execution_date`** (Airflow 1.x / legacy term): the **start of the data interval** being processed, NOT when the run actually executes. Renamed **`logical_date`** in Airflow 2.2+.
- For `@daily`, a run with `execution_date = 2024-08-02` runs at 2024-08-03 00:00 and processes **data from Aug 2**.
- Data interval = `[execution_date, execution_date + schedule_interval)`.
- In templating: `{{ ds }}` gives `execution_date` in YYYY-MM-DD; `{{ next_ds }}`, `{{ prev_ds }}`, `{{ data_interval_start }}`, `{{ data_interval_end }}` are common templates.

### start_date and catchup Behavior
- `start_date` anchors the schedule. The scheduler only creates runs whose data interval start >= `start_date`.
- With `catchup=False` (recommended default), the scheduler only creates a run for the **most recent** eligible interval.
- With `catchup=True`, every missed interval from `start_date` to now gets a run — this is how accidental massive backfills happen (see Common Mistakes).

### Data Interval Concept in Airflow 2.x
- Formalized via **Timetable**: each schedule defines a sequence of *data intervals*; each run is associated with exactly one interval.
- `data_interval_start` and `data_interval_end` are the authoritative bounds for a run — use them to filter queries: `WHERE event_date BETWEEN '{{ data_interval_start }}' AND '{{ data_interval_end }}'`.

### Backfill (running past DAG runs)
```bash
airflow dags backfill etl_demo \
  --start-date 2024-06-01 \
  --end-date 2024-06-30 \
  --reset-dagruns \
  -i   # ignore dependencies on upstream DAGs if needed
```
- Re-runs all intervals in range, even if they previously succeeded.
- Perfect for populating a new warehouse, correcting logic, or replaying failed windows.

### External Triggers
- **Manual trigger** from the UI (Run → Trigger DAG) or:
  ```bash
  airflow dags trigger etl_demo
  ```
- Trigger with config (`conf`) — a JSON dict passed to the run:
  ```bash
  airflow dags trigger etl_demo --conf '{"run_id": "special", "table": "orders_2024"}'
  ```
  In DAG: `{{ dag_run.conf["table"] }}` or in code `context["dag_run"].conf`.
- REST API: `POST /dags/{dag_id}/dagRuns` with a JSON body containing `conf`.
- Useful for ad-hoc runs, parameterized DAGs, and event-driven pipelines.

---

## Monitoring and Observability

### Airflow UI Views
- **Tree view**: grid of DAG runs (columns) × tasks (rows); colored squares show task states. Good for spotting a single failing task across many runs.
- **Graph view**: the DAG structure with dependencies; zoom into Task Groups. Good for understanding the pipeline.
- **Gantt**: timeline of task durations per run. Great for finding bottlenecks (which task is slow?).
- **Calendar**: which days the DAG ran and with what status (Airflow 2.x).
- **Code view**: shows the DAG source file.
- **Details**: tags, owner, schedule, next run.
- **Runs tab**: list of DAG runs with states and trigger types.
- Also: **Admin** screens (Connections, Variables, Pools, XComs), and **Browse** (Task Instances, DAG Runs) for deep-dives.

### Task States
- **queued**: scheduler decided it's runnable and submitted to the executor; waiting for a slot.
- **running**: currently executing on a worker.
- **success**: completed without errors.
- **failed**: task errored and is past its retries.
- **up_for_retry**: task failed but a retry is scheduled (not a final failure — don't page on this).
- **up_for_reschedule**: sensor in `reschedule` mode waiting for the next poke.
- **skipped**: skipped due to branching, `BranchPythonOperator`, `trigger_rule`, or `skip_on_exit_code`/upstream skip with default trigger rule.
- **upstream_failed**: a dependency failed, so this task wasn't attempted.
- **removed**: task no longer exists in the DAG definition but there's a historical instance.
- **no_status** / **none**: task created but the run hasn't reached it yet.

### SLAs (Service Level Agreements) and Alerts
- Define `sla` (a `timedelta`) per task. If the task runs longer than the SLA, Airflow:
  - Marks the TaskInstance's `sla_miss` and fires an **SLA miss email** (or callback).
  - The SLA **does not** fail the task — it's purely a notification/observability signal.
- Set globally in `default_args` or per task.
- SLA miss emails go to `sla_miss_email_addresses` in `airflow.cfg` or DAG-level `sla_miss_callback`.
- Useful for: "the load must finish within 2 hours or the dashboard team needs to know."

### Email / Alerting on Failures
- Task-level: `email_on_failure=True`, `email_on_retry=True`, `email` list in `default_args`.
- Custom callbacks (much better than email):
  ```python
  from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator

  def notify_failure(context):
      SlackWebhookOperator(
          task_id="slack_notify",
          webhook_token="...",
          message=f"DAG {context['dag'].dag_id} failed on task {context['task_instance'].task_id}",
      ).execute(context)

  with DAG(..., default_args={"on_failure_callback": notify_failure}) as dag:
      ...
  ```
- `on_success_callback`, `on_failure_callback`, `on_retry_callback` are the hooks to wire PagerDuty/Slack/email.
- Sending email requires an SMTP connection configured in `airflow.cfg` (`smtp_host`, `smtp_user`, `smtp_password`, `smtp_starttls`).

### Logs and Log Retrieval
- Task logs are written by workers; by default to `{AIRFLOW_HOME}/logs/`.
- **Log retrievers** let you centralize: configure a **remote log handler** (CloudWatch, GCS, S3, Azure Blob, Elasticsearch) so logs persist outside the worker.
- In the UI: click a task instance → **Log** tab; you can also see XCom values, rendered templates, and task details.
- CLI: `airflow tasks logs <dag_id> <task_id> <run_id>`.
- Best practice: remote log storage + log retention, especially when workers are ephemeral (Kubernetes pods are destroyed after the run → logs vanish without a remote handler).

---

## Best Practices

### Idempotent Tasks
- A retry or backfill must produce the **same outcome** as the first run.
- Techniques:
  - **Incremental load keyed on data interval**: `DELETE WHERE partition_date = '{{ ds }}'` then `INSERT`.
  - Use `MERGE` / `INSERT ... ON CONFLICT DO UPDATE`.
  - Write to a staging table then swap partitions.
- Why: Airflow *will* retry and *will* be backfilled. Non-idempotent pipelines cause double-counting and silent data corruption.

### Small, Focused Tasks
- **One logical unit of work per task** — a task should do one thing and be describable in a sentence.
- Small tasks = easier retries (retry a tiny step, not a 3-hour monster), clearer failure attribution, better parallelism.
- If a task takes 2 hours and fails at the end, your retry re-does everything — consider splitting or checkpointing.

### Don't Do Heavy Processing in the Scheduler
- The scheduler re-imports every DAG file periodically. Any top-level code runs on **every parse**.
- So: no DB queries at import time, no API calls, no file scans, no `os.system` at module level.
- Fetch config inside tasks or use Variables (they're cached); keep DAG file import time under ~1s.
- Heavy parse times slow the whole scheduler and can delay *every* DAG in the deployment.

### Keep DAGs Simple and Static
- Structure must be deterministic at parse time (no data-dependent task creation).
- Use Task Groups to organize, not to add conditional logic at parse time.
- If you need dynamic behavior, use Dynamic Task Mapping or branch operators inside tasks.

### Properly Handle Secrets
- **Never** hardcode credentials in DAG code or `default_args`.
- Use Airflow Connections (encrypted at rest), Variables (hidden option), or a **Secret Backend** (Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault).
- Configure `secrets_backend` in `airflow.cfg` so secrets are fetched at runtime, not baked into files.
- Review access: restrict who can edit/view connections (RBAC).

### Backfill Strategy
- Start new DAGs with `catchup=False`; opt into historical runs deliberately via `airflow dags backfill`.
- Before backfilling: make sure the DAG is idempotent, resource limits are sensible, and the target system can handle the load.
- Use `--rerun-failed` / `--reset-dagruns` carefully; test with a small date range first.

### Code Reviews for DAGs
- Review for: idempotency, retries set, secrets handling, no heavy parse-time logic, correct `schedule_interval`/`start_date`/`catchup`, sensible pools/queues, monitoring (SLA + alerts).
- Adopt linting + unit tests: `pytest` with `airflow` DAG import tests, `dagbag` assertions, and CI checks that import every DAG to catch syntax errors early.
- Version control DAGs like any application code; promote via deploy pipelines (not manual edits on the server).

---

## Real-World Data Pipeline

### Example: Daily ETL Pipeline (extract → dbt transform → load to warehouse)

```python
from datetime import datetime, timedelta

from airflow.decorators import dag, task
from airflow.operators.bash import BashOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator

DBT_DIR = "/opt/dbt/project"

@dag(
    dag_id="daily_etl",
    schedule="0 3 * * *",          # 03:00 UTC daily
    start_date=datetime(2024, 1, 1),
    catchup=False,
    max_active_runs=1,
    default_args={
        "owner": "data-eng",
        "retries": 2,
        "retry_delay": timedelta(minutes=10),
        "retry_exponential_backoff": True,
        "email_on_failure": True,
        "email": ["data-oncall@company.com"],
    },
    tags=["etl", "production"],
)
def daily_etl():
    @task
    def extract_from_db():
        from airflow.providers.postgres.hooks.postgres import PostgresHook
        hook = PostgresHook(postgres_conn_id="oltp_primary")
        sql = """
            INSERT INTO landing.orders (order_id, customer_id, amount, created_at)
            SELECT order_id, customer_id, amount, created_at
            FROM public.orders
            WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
              AND created_at < CURRENT_DATE;
        """
        hook.run(sql)

    # Orchestrate a dbt run for the transform layer
    dbt_run = BashOperator(
        task_id="dbt_transform",
        bash_command=f"cd {DBT_DIR} && dbt run --profiles-dir {DBT_DIR}/profiles",
    )

    # Verify row counts on the final mart table
    check_mart = SnowflakeOperator(
        task_id="check_mart_freshness",
        snowflake_conn_id="snowflake_analytics",
        sql="""
            SELECT CASE
                WHEN max(updated_at) >= DATEADD(day, -1, CURRENT_TIMESTAMP())
                THEN 'OK' ELSE 'STALE'
            END
            FROM analytics.mart_daily_orders;
        """,
    )

    extract_from_db() >> dbt_run >> check_mart

daily_etl()
```

### Orchestrating dbt Runs from Airflow
- Two common approaches:
  1. **`dbt` Python package / CLI in a task** (`BashOperator` or `PythonOperator` calling `dbt run`). Simple; you control the project dir and profiles.
  2. **`DbtRunOperator` / `DbtTaskGroup`** from providers (`dbt-airflow` or `airflow-dbt-python`): gives per-model tasks, native docs/test integration, and a nice graph.
- Model order comes from dbt's own DAG; Airflow just decides *when* to invoke it (e.g., after raw extraction, before the mart check).
- Example with DbtTaskGroup:
  ```python
  from dbt_airflow.core.task_group import DbtTaskGroup
  dbt_tg = DbtTaskGroup(
      group_id="dbt_transform",
      dbt_manifest_path="/opt/dbt/target/manifest.json",
      dbt_project_path="/opt/dbt",
      dbt_profile_path="/opt/dbt/profiles",
      select=["+mart_orders"],
  )
  extract >> dbt_tg >> check_mart
  ```

### Orchestrating Databricks Notebooks from Airflow
- `DatabricksRunNowOperator` (or `DatabricksSubmitRunOperator`) triggers a **job/notebook** in Databricks without Airflow running code on the cluster itself.
- Set up a **Databricks connection** in Airflow with host, token, and (optionally) a cluster/job.
```python
from airflow.providers.databricks.operators.databricks import DatabricksRunNowOperator

run_notebook = DatabricksRunNowOperator(
    task_id="run_ml_training",
    databricks_conn_id="databricks_default",
    job_id=12345,            # or notebook_params with DatabricksSubmitRunOperator
    notebook_params={"input_date": "{{ ds }}", "model_name": "churn_v3"},
)
```
- Pattern: Airflow handles the orchestration/failure/retries; Databricks handles Spark compute.

### Sensor to Wait for Snowflake Stage Files
```python
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.sensors.external_task import ExternalTaskSensor

# Pattern 1: poll Snowflake for the file list in a stage
wait_for_files = SnowflakeOperator(
    task_id="wait_for_stage_files",
    snowflake_conn_id="snowflake_analytics",
    sql="""
        SELECT 1 FROM TABLE(INFORMATION_SCHEMA.STAGE_FILE_INFO(
            ('@INGEST_STAGE/daily/', 'fileName')) );
    """,
)

# Pattern 2 (more robust): use a custom PythonOperator that polls until count > 0
def poll_stage():
    from airflow.providers.snowflake.hooks.snowflake import SnowflakeHook
    hook = SnowflakeHook(snowflake_conn_id="snowflake_analytics")
    result = hook.get_records("""
        SELECT count(*) FROM TABLE(INFORMATION_SCHEMA.STAGE_FILE_INFO(
            ('@INGEST_STAGE/daily/', 'fileName')))
    """)
    if result[0][0] == 0:
        raise AirflowSkipException("no files yet")  # or retry via task retries
    return result[0][0]

wait = PythonOperator(task_id="wait_for_stage_files", python_callable=poll_stage,
                      retries=10, retry_delay=timedelta(minutes=5))
```
- Note: stage file info is a metadata query (no data read) — cheap to poll. Prefer SQL polling over `LIST`-in-a-bash-hack.

---

## Common Mistakes

### Setting start_date in the Past Causing Unwanted Backfills
- Classic trap: `start_date=datetime(2020,1,1)` with `catchup=True` (default) → thousands of runs created instantly on deploy.
- Fix: `catchup=False` in the DAG, and/or set `start_date` sensibly; run explicit backfills when you actually want history.

### Heavy Logic in the Scheduler (DAG parsing overhead)
- Top-level code in DAG files runs on **every parse** (default every ~30s). DB/API calls there slow the whole scheduler.
- Fix: move expensive work into task callables; use Variables (cached); keep parse time low; unit-test import time.

### Overusing XComs (large data via XComs)
- XComs live in the metadata DB. Pushing DataFrames or big JSON blobs bloats the DB, slows the scheduler, and can hit size limits.
- Fix: push file paths/IDs only; store payloads in S3/GCS/warehouse; use custom xcom backend if truly needed.

### Not Setting Retries
- Default `retries=0` → a transient network blip fails the task and (with email_on_failure) pages people.
- Fix: `default_args={"retries": 2, "retry_delay": timedelta(minutes=5)}` in production DAGs.

### catchup=True Unexpectedly Re-Running Old Runs
- Same root cause as the start_date trap; also happens after scheduler downtime (it "catches up" on missed intervals).
- Fix: understand `catchup`; prefer `False` unless backfill is intended; use `max_active_runs` and backfill CLI for controlled replays.

### Long-Running Tasks Without Resource Limits
- A task that runs for hours with no `execution_timeout` can hang forever and eat a worker slot.
- Fix: `execution_timeout=timedelta(hours=2)` per task; `dagrun_timeout` on the DAG; use pools and `max_active_tasks`.

### Non-Idempotent Tasks
- `INSERT` without dedup, append-only writes keyed on nothing, or relying on "this runs once" assumptions.
- Fix: write operations that tolerate re-execution (upserts, delete-then-insert per `data_interval`, staging + swap).

### Other common foot-guns worth mentioning
- **`depends_on_past=True`** with `catchup=False` and a failed run → the DAG is permanently blocked (one failure freezes all future runs). Use sparingly and clear the bad run.
- **Sensors in `poke` mode** holding worker slots for hours — use `mode="reschedule"`.
- **Forgetting `end_date`** for one-off jobs or accidentally scheduling with `None` schedule expecting "hourly".
- **Timezone confusion** — Airflow schedules in UTC by default; make sure `ds`/`data_interval_*` match business expectations.
- **Multiple schedulers without HA config** — in Airflow 2.x you can run several schedulers, but they must share the same DB and config; misconfiguration causes double-triggering.
- **Not monitoring SLA/alerts** — a pipeline that fails silently is worse than one that fails loudly.

---

## Interview Questions (15+)

### Basics
1. **What is Apache Airflow and what problem does it solve?** — Open-source workflow orchestration; schedules and monitors pipelines with dependencies, retries, and a UI. Distinguish from data processing frameworks (Spark) and transformation tools (dbt).
2. **Airflow vs cron: when would you use one over the other?** — Cron for simple fire-and-forget; Airflow for dependencies, retries, monitoring, backfills, parameterization, cross-DAG coordination.
3. **What is a DAG? Why must it be acyclic?** — Directed Acyclic Graph of tasks + dependencies. Acyclic because a cycle means circular dependency/never-ending scheduling; each task must be able to resolve to a start.
4. **Walk me through the components of the Airflow architecture.** — Scheduler, workers, web server, metadata DB (Postgres/MySQL), message queue (Redis/RabbitMQ with Celery), executor; explain the flow of a task from scheduling to execution.
5. **What is the difference between a DAG, a task, and an operator?** — DAG = workflow/graph; operator = template of work; task = operator instantiated with params in a DAG. TaskInstance = one execution of a task in a specific DAG run.

### Operators and Sensors
6. **What is the difference between an operator and a sensor?** — Operator performs an action; sensor waits for a condition (polls) and then succeeds. Both produce tasks; sensors have `poke_interval`, `timeout`, `mode`.
7. **What is the difference between an operator and a hook?** — Operator defines the task/action; hook is the interface to the external system (DB/API). Operators typically use hooks internally to interact with systems.
8. **How do you set task dependencies in Airflow?** — `>>` / `<<` / `set_upstream` / `set_downstream`; fan-in/fan-out patterns; and TaskFlow implicit dependencies via function calls.

### Executors
9. **Compare SequentialExecutor, LocalExecutor, CeleryExecutor, and KubernetesExecutor.** — Sequential (1 task, testing), Local (parallel, one box), Celery (distributed via Redis/RabbitMQ), Kubernetes (each task in its own pod). How to choose based on scale/infra.
10. **How does the scheduler decide which tasks are ready to run?** — Dependencies (upstream succeeded), schedule interval, concurrency limits (max_active_runs/max_active_tasks), pools with free slots, not in a state that blocks (e.g., past retry_delay elapsed).

### XComs and Data Passing
11. **What are XComs and what are their limitations?** — Cross-communication: small key-value data between tasks, stored in metadata DB. Limits: DB-backed, size limits, slow; use external storage for real data, pass references.
12. **What is the TaskFlow API?** — `@task`/`@dag` decorators; automatic XCom push/pull of return values/inputs; type-hinted signatures; dynamic task mapping (expand) in 2.3+.

### Scheduling
13. **What is the difference between execution_date / logical_date and when a DAG actually runs?** — Logical date = start of the data interval (e.g., Aug 2 data for the Aug 3 00:00 run); the run executes *after* the interval closes. Why you filter queries on the data interval.
14. **Explain catchup vs backfill.** — Catchup: automatic creation of missed runs between start_date and now (dangerous when start_date is far past). Backfill: explicit `airflow dags backfill` for a chosen range, e.g., to populate a new warehouse.
15. **How do you run a past DAG and what do you need to ensure first?** — Backfill or manual trigger with conf; first verify idempotency, retries, resource limits, and that catchup=False so you don't accidentally replay everything.
16. **What happens if a task fails? Describe the retry process.** — Scheduler marks TI up_for_retry, waits retry_delay (with optional exponential backoff), re-submits until retries exhausted, then failed; on_failure_callback / email fired.
17. **What is `depends_on_past` and when would you use it?** — Task runs only if the same task in the previous DAG run succeeded. For strict incremental pipelines; risky because a single failure blocks all future runs until cleared.

### Connections and Config
18. **How do you connect Airflow to an external system, e.g., Snowflake?** — Install provider package, create a Connection in Admin → Connections (conn_id, type, account/warehouse + extra JSON), reference `snowflake_conn_id` in `SnowflakeOperator`/`SnowflakeHook`. Credentials stored encrypted; never in code.
19. **What are Airflow Variables and Pools, and when would you use each?** — Variables: runtime config key-values. Pools: cap concurrent tasks (protect external systems, reserve capacity). Queues route tasks to specialized workers.

### Concepts & Ecosystem
20. **Airflow vs dbt: when do you use each?** — Airflow orchestrates when/order/retries; dbt transforms (SQL models, tests, docs). Complementary: Airflow triggers dbt runs as tasks.
21. **Why must DAGs be idempotent, and how do you make a task idempotent?** — Retries and backfills re-run tasks; idempotency means same result every time (upserts, delete+insert per data interval). Non-idempotent → duplicate data.
22. **What is an SLA in Airflow?** — A `timedelta` threshold per task; if the task runs longer, an SLA miss is fired (email/callback). It does not fail the task — pure observability.
23. **How do you monitor a large Airflow deployment?** — UI views (tree/graph/Gantt/calendar), task states, remote log aggregation, SLAs + on_failure callbacks to Slack/PagerDuty, DAG-level metrics (parse time, scheduler heartbeats), pools and queue depth monitoring.
24. **What's the difference between Airflow 1.x and 2.x?** — TaskFlow API, scheduler HA, stable REST API, timezone-aware, deferrable operators, dynamic task mapping, better UI, `schedule`/timetables, removed some deprecated mechanics (e.g., old executor pinning).
25. **How would you prevent two overlapping runs of the same DAG?** — `max_active_runs=1`, `catchup=False`, and/or a time-limit sensor; also `dagrun_timeout` to kill stuck runs.

---

## Quick Reference Cheat Sheet

```python
# Common imports
from airflow import DAG
from airflow.decorators import dag, task
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.utils.task_group import TaskGroup

# Scheduling
schedule="@daily"          # cron presets: @hourly @weekly @monthly @yearly @once
schedule="0 8 * * *"       # daily 08:00
start_date=datetime(2024, 1, 1)
catchup=False              # avoid surprise backfills
max_active_runs=1

# Templating
# {{ ds }}             logical date YYYY-MM-DD
# {{ next_ds }}        next logical date
# {{ data_interval_start }} / {{ data_interval_end }}
# {{ conn.<conn_id> }} connection ref
# {{ var.value.<name> }} / {{ var.json.<name> }}
# {{ dag_run.conf["key"] }} trigger config

# Retries
default_args = {"retries": 2, "retry_delay": timedelta(minutes=5),
                "retry_exponential_backoff": True, "execution_timeout": timedelta(hours=2)}

# Key CLI commands
# airflow dags backfill <dag_id> --start-date ... --end-date ...
# airflow dags trigger <dag_id> --conf '{"k": "v"}'
# airflow tasks logs <dag_id> <task_id> <run_id>
```

---

*Last updated: 2026. Verify current provider/API details against the Airflow version you use — Airflow moves fast (2.9+/3.x by the time you read this).*
