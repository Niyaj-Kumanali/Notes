# dbt (data build tool) — Interview Notes

Comprehensive study notes covering the data build tool for data engineering and analytics engineering interviews. Written in readable bullet-point style with working code examples.

---

## What is dbt

### What dbt stands for

- dbt = **d**ata **b**uild **t**ool.
- It is an open-source, command-line (and cloud) tool that lets analytics engineers and data engineers **transform data in the warehouse**.
- dbt does **not** extract data and **does not** load data. It sits in the middle of the modern ELT pipeline and handles only the **transform** step.
- dbt is often described as "the T in ELT" or "the ETL of ETL." Once raw data lands in your warehouse (Snowflake, BigQuery, Databricks, Redshift, Postgres, DuckDB, etc.), dbt takes over.
- Instead of copy-pasting SQL between tools, dbt treats each transformation as a **version-controlled, testable, documented SQL file** called a model.

### dbt as transformation layer in ELT pipeline (not ETL)

- The modern data stack favors **ELT (Extract, Load, Transform)** over the older **ETL (Extract, Transform, Load)**.
- In ELT:
  1. **Extract** — data is pulled from source systems (APIs, databases, files, events).
  2. **Load** — raw data is loaded into a cloud data warehouse (often with tools like Fivetran, Airbyte, Stitch).
  3. **Transform** — raw tables are cleaned, joined, aggregated, and shaped into analytics-ready models. **This is where dbt lives.**
- In dbt, transformation happens **inside the warehouse** using the warehouse's compute (e.g., Snowflake virtual warehouses), so there is no need to move large volumes of data back and forth between systems.
- Key contrast with ETL: ETL transforms before loading, often on a separate server, which is slower and requires more infrastructure for the same amount of data.
- Since dbt only writes SQL, it is extremely portable — the same dbt project can target Snowflake, BigQuery, Databricks, Redshift, Postgres, and others simply by changing the connection profile.

### dbt philosophy: analytics engineering, code-first approach

- dbt popularized the role of the **analytics engineer** — a person who bridges the gap between data engineering and business analysis.
- The philosophy is **code-first**: transformations are written in plain SQL + Jinja, stored in a git repo, reviewed through pull requests, and deployed through CI/CD — just like application code.
- Everything is **version controlled**. Models, tests, docs, macros, and configurations live in a git repository.
- **DRY (Don't Repeat Yourself)** principles are encouraged: macros, reusable SQL, and modular models eliminate copy-paste SQL.
- **Modularity**: large, monolithic SQL queries are broken into small, composable models. Each model is one logical step in a pipeline.
- **Documentation and tests are first-class citizens**, not afterthoughts. dbt auto-generates a documentation site (dbt docs) and supports data quality tests directly in the project.

### dbt Core vs dbt Cloud

- **dbt Core** is the free, open-source CLI tool. You run commands like `dbt run`, `dbt test`, `dbt build` from your terminal. You manage scheduling, orchestration, and environment yourself.
- **dbt Cloud** is the commercial SaaS offering. It adds:
  - A web-based **IDE** for writing and running dbt.
  - **Scheduling and jobs** (cron-based runs of dbt commands).
  - **Environment and credential management** (stored secrets, no need to maintain profiles.yml locally).
  - **CI/CD** built in (run tests on PRs, then promote to prod).
  - **Documentation hosting**, lineage/impact analysis UI, and a semantic layer.
- Both use the exact same underlying dbt project code, models, and configs. Code written for Core runs in Cloud and vice versa.
- Cost: dbt Core is free (MIT license); dbt Cloud is subscription-based with tiers (Developer, Team, Enterprise).

### dbt's place in the modern data stack

- The "modern data stack" typically includes:
  - **Ingestion**: Fivetran, Airbyte, Stitch, custom pipelines.
  - **Storage / Compute**: Snowflake, Databricks, BigQuery, Redshift.
  - **Transformation**: dbt (the de-facto standard).
  - **BI / Visualization**: Looker, Tableau, Power BI, Metabase, Mode.
  - **Reverse ETL / Activation**: Hightouch, Census.
- dbt generates the "trusted layer" (marts, dimension/fact tables) that BI tools query. Because dbt runs inside the warehouse, BI tools query tables directly without an extra transformation engine.
- Warehouse features (e.g., Snowflake micro-partitions, BigQuery partitioning/clustering) can be configured directly from dbt, making dbt the single point of control for the analytics dataset layer.

### Why dbt (benefits)

- **Version control**: all code is in git; changes are reviewable, revertable, and auditable.
- **Testing**: built-in and custom data tests catch nulls, duplicates, referential integrity issues, and more before bad data reaches dashboards.
- **Documentation**: auto-generated docs site with lineage (DAG), model descriptions, column descriptions, and test status.
- **Modularity / reusability**: models, macros, and packages reduce duplication and enforce standards.
- **Dependency management**: dbt builds models in dependency order automatically, so you never hand-order your SQL scripts.
- **Environment parity**: the same code runs in dev, staging, and prod by parameterizing schema/database via `target`.
- **Community ecosystem**: hundreds of packages (dbt_utils, dbt_expectations, dbt_metrics) speed up development.

---

## dbt Projects and Structure

### What a dbt project contains

A dbt project is a directory containing configuration files, SQL files, YAML files, and more. The two most important files are:

- **`dbt_project.yml`** — the top-level project configuration.
- **`profiles.yml`** — the connection/credential configuration (typically lives in `~/.dbt/`, not inside the project, so credentials aren't committed).

### dbt_project.yml structure

Example:

```yaml
# dbt_project.yml
name: my_project
version: '1.0.0'
config-version: 2

profile: my_project_profile

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
analysis-paths: ["analyses"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

vars:
  start_date: '2020-01-01'

models:
  my_project:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: table
      +schema: marts
```

Key points to remember for interviews:

- `name` and `version` identify the project.
- `profile` links to the matching profile in `profiles.yml`.
- `config-version: 2` is the current config format (older projects used version 1).
- `model-paths`, `seed-paths`, etc. tell dbt where to look for each type of file. The defaults are shown above.
- `clean-targets` lists directories to remove on `dbt clean` (the compiled `target/` directory and installed packages).
- The `models:` block applies **inheritable configs** at the folder level. Nested folders inherit configs from parent folders.
- `vars` defines project-level variables, accessible in code via `{{ var('start_date') }}`.

### models/ directory and subdirectories

- Every SQL file in `models/` (and its subdirectories) becomes a dbt model.
- Common folder conventions (this is a recommended pattern, not a hard rule):
  ```
  models/
  ├── staging/
  │   ├── stg_customers.sql
  │   ├── stg_orders.sql
  │   └── sources.yml
  ├── intermediate/
  │   ├── int_customer_orders.sql
  │   └── int_order_items.sql
  ├── marts/
  │   ├── dim_customers.sql
  │   ├── dim_products.sql
  │   └── fact_orders.sql
  └── schema.yml
  ```
- **Staging models** (`stg_*`) lightly clean and standardize raw source data. They should be as close to a 1:1 mirror of the source as possible — rename columns, cast types, dedupe.
- **Intermediate models** (`int_*`) hold logic shared by multiple marts models — e.g., pre-joined datasets.
- **Mart models** (`dim_*`, `fact_*`, `report_*`) are business-facing, denormalized, and query-ready for BI.
- The folder structure drives default materializations and schemas via `dbt_project.yml`.

### snapshots/, tests/, macros/, seeds/, analyses/

- **`snapshots/`** — SQL files that implement slowly changing dimension (SCD) tracking. Run via `dbt snapshot`.
- **`tests/`** — folder for singular tests (SQL files that return rows when a test fails) and custom generic test definitions. Built-in generic tests are declared in `.yml` files next to models.
- **`macros/`** — reusable Jinja/SQL snippets (e.g., `generate_schema_name.sql`). Files can contain one or more `{% macro %}` blocks.
- **`seeds/`** — CSV files loaded into the warehouse. Run via `dbt seed`.
- **`analyses/`** — SQL files that are compiled (by `dbt compile`) but never run as models. Useful for exploratory queries and "one-off" reports that aren't part of the DAG.

### dbt profiles.yml and warehouse connection

- `profiles.yml` holds connection credentials and is stored outside the project by default (`~/.dbt/profiles.yml`).
- It is typically gitignored so secrets are never committed.

```yaml
# ~/.dbt/profiles.yml
my_project_profile:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: abc12345.us-east-1
      user: analytics_engineer
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: ANALYST
      database: ANALYTICS_DB
      warehouse: TRANSFORM_WH
      schema: DBT_DEV
      threads: 4
    prod:
      type: snowflake
      account: abc12345.us-east-1
      user: dbt_service_user
      password: "{{ env_var('DBT_PASSWORD_PROD') }}"
      role: DBT_PROD_ROLE
      database: ANALYTICS_DB
      warehouse: PROD_TRANSFORM_WH
      schema: DBT_PROD
      threads: 8
```

- **`target:`** selects the active output (e.g., `dev` or `prod`).
- **`outputs:`** — one or more named environments. Common fields:
  - `type`: warehouse adapter (snowflake, bigquery, redshift, postgres, databricks, duckdb...).
  - Warehouse-specific connection fields (account, project, host, port, database, etc.).
  - `schema`: the default target schema for models that do not override it.
  - `threads`: how many models dbt can build concurrently.
- `env_var()` lets you keep secrets out of the repo — read from environment variables at runtime.
- You can also set `DBT_PROFILES_DIR` to point dbt at a non-default profiles location (useful in CI).

### How to initialize a project (dbt init)

- `dbt init my_project` scaffolds a new project with a sensible folder layout and a sample `dbt_project.yml`.
- If you run `dbt init` interactively, it may ask which adapter (warehouse) to configure and create a starter profile.
- You can also use `dbt init <project_name> --adapter snowflake` to skip the prompt.
- After init, put your warehouse credentials in `~/.dbt/profiles.yml`, then run `dbt run` to verify the connection.
- `dbt debug` checks the connection, profile, and project config and prints useful diagnostics.

### Models materialization types: view, table, incremental, ephemeral

- A **materialization** is the strategy dbt uses to build a model in the warehouse.
- `view` (default) — model is created as a database view; cheap to create, always up to date, but re-queried each time it is referenced.
- `table` — model is created as a physical table, dropped and recreated on every `dbt run`; good for models queried often.
- `incremental` — model is a table that receives only new/changed rows on subsequent runs; best for large append-heavy datasets.
- `ephemeral` — model is inlined as a CTE into models that reference it; never materialized in the warehouse; good for lightweight reusable logic.
- Materializations can be set in `dbt_project.yml`, in `{{ config(...) }}` inside the model file, or with `--select` flags / `dbt build --full-refresh`.

---

## Models

### What is a dbt model

- A dbt model is a **single SQL `SELECT` statement**, saved as a `.sql` file in the `models/` directory.
- Each model represents one logical transformation step. dbt wraps your SELECT in the appropriate DDL/DML (CREATE VIEW, CREATE TABLE, MERGE/INSERT...) depending on its materialization.
- Because models are just SELECT statements, they are easy to review, test, and reason about.
- Models can reference other models, sources, seeds, and snapshots, forming a **dependency graph** (DAG).

### .sql files in models directory

- The file name becomes the model name. `models/staging/stg_orders.sql` creates a relation called `stg_orders` (unless renamed with `alias`).
- Every model file has the same shape:

```sql
-- models/marts/fact_orders.sql
{{ config(materialized='table') }}

SELECT
    o.order_id,
    o.customer_id,
    o.order_date,
    o.status,
    o.total_amount
FROM {{ ref('stg_orders') }} AS o
WHERE o.order_date >= '2020-01-01'
```

- dbt compiles this Jinja-templated file into plain SQL before executing it.

### Jinja templating in models

- Models are written in **Jinja + SQL**. Jinja is a Python templating engine (the same one used by Flask/Ansible) that dbt uses to make SQL dynamic.
- Three Jinja constructs you must know for interviews:
  1. `{{ ... }}` — expressions that evaluate and print a value (e.g., `{{ ref('stg_orders') }}`, `{{ this }}`).
  2. `{% ... %}` — statement/control blocks (e.g., `{% if is_incremental() %}`, `{% for col in columns %}`).
  3. `{# ... #}` — comments, not rendered in output SQL.

### ref() function vs source() function

- `{{ ref('model_name') }}` references **another model, seed, or snapshot** in your project.
- `{{ source('source_name', 'table_name') }}` references a **raw table** that was defined in a sources.yml file.
- dbt resolves both into the correct fully-qualified relation name (including target schema/database), so you never hardcode schema names.
- `ref()` automatically builds the dependency graph: if model B references model A, dbt knows B must run after A, and `dbt test` / `dbt docs` track the relationship.
- `ref()` also ensures that in the same run, dbt uses the newly built version of upstream models.
- Use `source()` for raw ingested tables; use `ref()` for everything your project builds (models, seeds, snapshots).

```sql
-- staging model using source()
SELECT
    id::bigint          AS customer_id,
    created_at::date    AS created_date,
    lower(email)        AS email
FROM {{ source('raw', 'customers') }}
```

### Building a dependency graph

- dbt reads all `ref()` and `source()` calls across your models and builds a **Directed Acyclic Graph (DAG)**.
- The DAG determines:
  - The **order** in which models run (dependencies first).
  - Which models are affected by a change (**downstream impact**).
  - What `dbt test` runs after each model (tests are attached to nodes in the DAG).
  - The lineage graph shown in `dbt docs`.
- The DAG must be acyclic — if model A refs B and B refs A, dbt raises a "circular dependency" error.

### Running order (dbt determines automatically from refs)

- You never specify run order manually. If `stg_orders` feeds `int_orders` which feeds `fact_orders`, dbt runs them in that order.
- When a model references another, dbt inserts the referenced model earlier in the run and resolves the ref to the just-created table/view.
- This is a major advantage over hand-written scripts where ordering is your responsibility and easy to get wrong.

### dbt run, dbt build commands

- `dbt run` — builds all models (and snapshots with `--select`/tags as configured). Executes each model's materialization.
- `dbt build` — builds models **and** runs their tests, plus loads seeds and runs snapshots in one command. Best for full pipeline runs.
- Both support `--select`, `--exclude`, `--full-refresh`, and node-selection syntax (see the dbt Workflow section).

---

## dbt Sources

### What are sources (raw tables in warehouse)

- A **source** describes a raw table that already exists in your warehouse and was loaded by an external tool (Fivetran, Airbyte, your own pipeline).
- dbt does not load or manage source tables; it only references and tests them.
- Defining sources in a `.yml` file lets you:
  - Reference them cleanly with `{{ source('name', 'table') }}`.
  - Test raw data (e.g., not_null on keys).
  - Track freshness.
  - Document raw tables alongside your models.

### sources.yml definition

```yaml
# models/staging/sources.yml
version: 2

sources:
  - name: raw
    database: ANALYTICS_DB
    schema: RAW_DATA
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _loaded_at
    tables:
      - name: customers
        description: "Raw customer records from the CRM."
        freshness:
          warn_after: {count: 24, period: hour}
        columns:
          - name: id
            description: "Primary key of the customer."
            tests:
              - not_null
              - unique
      - name: orders
        description: "Raw orders table."
        columns:
          - name: order_id
            tests:
              - unique
              - not_null
```

- `name` — the source name used as the first argument to `source()`.
- `database` / `schema` — where the raw tables physically live.
- `tables` — list each raw table and its columns/tests.
- `freshness`, `loaded_at_field`, `warn_after`, `error_after` — configuration for source freshness checks (see below).

### Source freshness checks

- dbt can check how recent the data in a source table is.
- Configuration:
  - `loaded_at_field`: the timestamp column that records when the row was loaded.
  - `warn_after: {count: N, period: hour|day|...}`: warn if data older than N hours/days.
  - `error_after: {count: N, period: ...}`: fail if data older than N.
- Freshness can be configured at the **source level** (applies to all tables) and **overridden per table**.

```yaml
# per-source example
sources:
  - name: raw
    loaded_at_field: _loaded_at
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
```

### dbt source freshness command

- `dbt source freshness` — checks freshness for all sources that have freshness configured, without running models.
- `dbt build` also runs freshness checks as part of the pipeline.
- `dbt test --select source:raw` runs only the tests defined on source tables.
- The command returns nonzero exit code if any source is beyond `error_after`, so it can gate a pipeline in CI.

### Why track source freshness

- If a source pipeline silently stops (API auth expires, sync job fails), freshness checks **catch it early** before analysts trust stale data.
- It gives stakeholders a guarantee that dashboards are built on data of a known recency.
- It is cheap and easy to set up once per source, and it turns "the source might be stale" into an automated alert.

---

## Materializations

### view (default)

- Default materialization in dbt.
- dbt creates a `CREATE VIEW` from your SELECT. No data is stored; the query runs every time the view is accessed.
- Pros: always current, no storage cost, fast to create.
- Cons: slower queries for heavy transformations (recomputed each read), cannot be indexed/clustered.
- Best for: staging models, lightweight transforms, frequently re-run pipelines.

```sql
{{ config(materialized='view') }}

SELECT * FROM {{ ref('stg_orders') }}
```

### table

- dbt creates a physical table using `CREATE TABLE ... AS SELECT`, replaced on each run.
- Pros: query performance (data pre-computed), can be clustered/partitioned.
- Cons: run takes longer (full rebuild), needs re-running when upstream data changes.
- Best for: marts/dimensions/facts that BI tools query frequently, final reporting tables.

```sql
{{ config(materialized='table', cluster_by=['order_date']) }}

SELECT * FROM {{ ref('int_orders') }}
```

### incremental

- Physical table that is **built once fully**, then only inserts/merges **new or changed rows** on subsequent runs.
- Pros: dramatically faster runs on large datasets (only touch recent data), lower compute cost.
- Cons: more complex to configure correctly; risk of missing updates/deletes if misconfigured.
- Best for: large, append-heavy fact tables (events, orders, logs) where a full rebuild is wasteful.

### ephemeral

- No database object at all. dbt inlines the model's SQL as a **CTE** into every model that references it.
- Pros: no warehouse objects, useful for shared helper logic.
- Cons: if many models reference an ephemeral model, the CTE is duplicated into each query (compiled SQL gets large); cannot be queried directly or tested with `dbt test` unless downstream.
- Best for: small reusable logic like `int` cleanup steps used by a handful of models. Avoid for wide/heavy transformations.

```sql
{{ config(materialized='ephemeral') }}

SELECT
    customer_id,
    COUNT(*) AS order_count
FROM {{ ref('stg_orders') }}
GROUP BY 1
```

### How to choose materialization for a model

- Start simple: staging as views, marts as tables.
- If a model is queried frequently and the underlying data changes rarely → `table`.
- If a model is very large and grows with new rows → `incremental`.
- If a model is small logic reused inside other models only → `ephemeral`.
- General rule: only add complexity (incremental) when you have a measured need. Premature incremental models are a common anti-pattern.

### Incremental model syntax

Full example with `is_incremental()`, `unique_key`, and an `updated_at` filter:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

SELECT
    order_id,
    customer_id,
    order_date,
    status,
    total_amount,
    updated_at
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

- `is_incremental()` returns True when running an incremental model that already exists → wrap the "only new rows" filter inside it.
- `unique_key` tells dbt which column(s) identify a row so it can update changed rows instead of only appending.
- `{{ this }}` refers to the model's own target relation, used to find the latest processed timestamp.
- Without `unique_key`, dbt defaults to `append` behavior (only inserts), which can create duplicates on re-runs.
- Common `incremental_strategy` values by warehouse: `merge` (Snowflake, Databricks, BigQuery), `delete+insert` (Snowflake, Databricks, BigQuery, Redshift), `append` (all), `insert_overwrite` (BigQuery, Databricks).

---

## Tests

### What are dbt tests

- Tests are **assertions about your data**. A test passes if it returns zero rows; it fails if it returns any rows.
- Tests run after their model is built. A failing test stops downstream execution in `dbt build` and fails the run.
- Tests are written declaratively in `.yml` files (generic tests) or as standalone SQL files (singular tests).
- Tests convert into SQL queries that get compiled by dbt, just like models.

### Built-in tests

The four built-in generic tests:

- `unique` — every value in a column must be distinct.

```yaml
- name: order_id
  tests:
    - unique
```

- `not_null` — the column must never contain NULLs.

```yaml
- name: customer_id
  tests:
    - not_null
```

- `relationships` — values in this column must exist in a referenced table's column (referential integrity / foreign key check).

```yaml
- name: customer_id
  tests:
    - relationships:
        to: ref('dim_customers')
        field: customer_id
```

- `accepted_values` — values must be in a fixed allowed list.

```yaml
- name: status
  tests:
    - accepted_values:
        values: ['pending', 'shipped', 'delivered', 'cancelled']
```

- These can be combined with additional arguments like `severity: warn` or `severity: error`, `config: {enabled: false}`, and `where` filters.

```yaml
- name: email
  tests:
    - not_null:
        severity: warn
    - unique:
        where: "is_active = true"
```

### Custom tests

- **Singular tests** — a plain SQL file in `tests/` that returns failing rows. Custom, one-off assertions.

```sql
-- tests/orders_total_is_positive.sql
SELECT order_id
FROM {{ ref('fact_orders') }}
WHERE total_amount <= 0
```

- **Generic tests** — parameterized, reusable test macros in `macros/`. Define a test macro with the same name and dbt treats it as a test block.

```sql
-- macros/assert_is_greater_than.sql
{% test assert_is_greater_than(model, column_name, min_value) %}

    SELECT *
    FROM {{ model }}
    WHERE {{ column_name }} <= {{ min_value }}

{% endtest %}
```

```yaml
# then reference it in a schema.yml
- name: total_amount
  tests:
    - assert_is_greater_than:
        min_value: 0
```

- For SQL dialects: singular tests are select statements; dbt wraps them in `SELECT count(*) FROM ( <test sql> )` internally to detect failing rows. Generic test macros receive `model` and other args and return a failing-rows SELECT.

### Defining tests in .yml files

- Tests live in `schema.yml` (or `sources.yml`) files next to models/sources.
- Structure: under `models:` → `- name: <model>` → `columns:` → `- name: <column>` → `tests:`.

```yaml
version: 2

models:
  - name: fact_orders
    description: "Order fact table at line-item grain."
    columns:
      - name: order_id
        tests:
          - not_null
          - unique
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id
```

### dbt test command

- `dbt test` — runs all tests in the project (needs models built first, since tests depend on relations existing).
- `dbt build` — builds models and runs their tests together.
- Targeted runs: `dbt test --select fact_orders`, `dbt test --select staging`, `dbt test --select source:raw`.
- `dbt test --store-failures` — stores failing rows in tables (schema `dbt_test__audit` by default) so you can inspect them later.

### Testing critical business logic

- Focus tests on **keys, filters, and business rules**:
  - Primary keys are `not_null` and `unique`.
  - Foreign keys exist in parent tables (`relationships`).
  - Categorical columns match expected values (`accepted_values`).
  - Revenue/quantity amounts are non-negative and reasonable.
  - Row counts reconcile between staging and marts (e.g., no unexpected explosion in grain).
- Test at the **source and staging layer** (raw data quality) and at the **mart layer** (final business truth).
- Not every column needs a test — test what protects business value and catches breakage early.

---

## Seeds

### What are dbt seeds

- Seeds are **CSV files** stored in the `seeds/` directory that dbt loads into tables in your warehouse.
- The CSV file name becomes the table name.
- Loaded via `dbt seed`, which creates the table and inserts rows.

Example seed `seeds/state_codes.csv`:

```csv
state_code,state_name
CA,California
NY,New York
TX,Texas
FL,Florida
```

- Once loaded, other models reference it with `{{ ref('state_codes') }}`.

### When to use seeds

- Small, **static reference data** that doesn't come from an external system (country/state lists, department mappings, calendar dimensions, office locations).
- Configuration/lookup tables used across multiple models.
- NOT for large or frequently-changing data — for that, use an ingestion tool or source.
- Seeds are meant to be version-controlled with your code, so the data itself lives in git.

### dbt seed command

- `dbt seed` — loads all seeds.
- `dbt seed --select state_codes` — load one seed.
- `dbt seed --full-refresh` — drop and recreate seed tables (useful after you edit the CSV).
- Seeds are part of the DAG and can be referenced with `ref()`.
- In `dbt build`, seeds are loaded before models run.

---

## Snapshots

### What are snapshots

- Snapshots implement **slowly changing dimension (SCD) Type 2** tracking in dbt.
- A snapshot records the full history of changes to a table: when a row's attributes change, the old version is kept with an `dbt_valid_from` timestamp and `dbt_valid_to` set to the change time, and a new version is added.
- This lets you answer questions like "what did this record look like on this date?"

### When to use snapshots

- Auditing: track when and how records changed.
- Historical reporting: reconstruct state as of any point in time (e.g., "what was the customer's plan last month?").
- Reference data that changes over time and needs an auditable history.
- Use for slowly-changing reference/entity data (customer profiles, product metadata), NOT for large fact tables or append-only event data.

### Snapshot configuration

```sql
-- snapshots/customer_snapshot.sql
{% snapshot customer_snapshot %}

{{
    config(
        target_database='ANALYTICS_DB',
        target_schema='snapshots',
        unique_key='customer_id',
        strategy='timestamp',
        updated_at='updated_at'
    )
}}

SELECT
    customer_id,
    first_name,
    last_name,
    email,
    plan,
    updated_at
FROM {{ source('raw', 'customers') }}

{% endsnapshot %}
```

- `target_schema` — schema where snapshot tables live (default is `snapshots`).
- `unique_key` — identifies a row for change detection.
- `strategy` — two options:
  - `timestamp`: compares an `updated_at` column to detect changes (recommended).
  - `check`: compares a list of columns (`check_cols`) for changes; `check_cols='all'` compares every column.

```sql
-- check strategy variant
{% snapshot product_snapshot %}
{{ config(
    target_schema='snapshots',
    unique_key='product_id',
    strategy='check',
    check_cols=['name', 'price', 'category']
) }}

SELECT product_id, name, price, category
FROM {{ source('raw', 'products') }}

{% endsnapshot %}
```

- dbt automatically adds `dbt_scd_id`, `dbt_updated_at`, `dbt_valid_from`, `dbt_valid_to` columns to the snapshot table.

### dbt snapshot command

- `dbt snapshot` — runs all snapshots (or select with `--select`).
- `dbt build` also runs snapshots.
- On first run, the full current state is loaded with `dbt_valid_from` set to now.
- On later runs, changed rows get `dbt_valid_to` set and new rows are inserted with a new `dbt_valid_from`.
- Current records have `dbt_valid_to IS NULL`.

### Type 2 SCD via dbt snapshots

- Snapshot table example output:

```
customer_id | plan    | dbt_valid_from       | dbt_valid_to
------------+---------+----------------------+--------------------
1001        | Basic   | 2024-01-05 09:00:00  | 2024-03-12 14:00:00
1001        | Premium | 2024-03-12 14:00:00 | NULL
```

- Query the current record: `WHERE dbt_valid_to IS NULL`.
- Query the record as of a date: `WHERE dbt_valid_from <= :as_of AND (dbt_valid_to IS NULL OR dbt_valid_to > :as_of)`.
- In BI, current-state queries filter on `dbt_valid_to IS NULL`; historical queries join on dates or filter with `dbt_valid_from`/`dbt_valid_to`.

---

## Macros and Jinja

### What are dbt macros

- Macros are **reusable snippets of Jinja/SQL** defined in `macros/*.sql` files.
- They work like functions: define once with `{% macro name(args) %}`, then call anywhere with `{{ name(args) }}`.
- Macros can return text (compiled into SQL), create control flow, loop over columns, and even accept a `model` argument (used heavily by generic tests and packages).

### Macro file structure

```sql
-- macros/string_helpers.sql
{% macro lower_trim(column_name) %}
    LOWER(TRIM({{ column_name }}))
{% endmacro %}
```

```sql
-- usage in a model
SELECT
    {{ lower_trim('email') }} AS email
FROM {{ ref('stg_customers') }}
```

- File names do not need to match macro names, but the macro name is what you call.
- Macro files can contain multiple macros.
- Naming convention: prefix macros with a namespace (e.g., `dbt_utils.` or your project name) to avoid collisions.

### Common Jinja constructs

**Variables and expressions:**

```sql
{{ var('start_date') }}   -- read a project var
{{ this }}                 -- the current model's relation
{{ target.name }}          -- current target name (e.g., 'dev')
{{ target.schema }}        -- target schema for this run
{{ run_started_at }}       -- timestamp when the run started
```

**if statements:**

```sql
{% if target.name == 'dev' %}
    WHERE created_at >= '2024-01-01'
{% endif %}
```

```sql
{% if is_incremental() %}
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**for loops:**

```sql
SELECT
{% for col in ['a', 'b', 'c'] %}
    {{ col }}{% if not loop.last %},{% endif %}
{% endfor %}
FROM {{ ref('stg_orders') }}
```

**Looping over a model's columns (macro):**

```sql
{% macro select_all_but_exclude(exclude_columns) %}

    SELECT
    {% for col in adapter.get_columns_in_relation(ref('stg_customers')) | map(attribute='name') | list %}
        {% if col not in exclude_columns %}
            {{ col }}{% if not loop.last %},{% endif %}
        {% endif %}
    {% endfor %}
    FROM {{ ref('stg_customers') }}

{% endmacro %}
```

### dbt_utils package and commonly used macros

- `dbt_utils` is the most popular dbt package with ~100+ utility macros.
- Commonly used macros:
  - `{{ dbt_utils.generate_surrogate_key(['customer_id', 'order_id']) }}` — build a stable surrogate/hash key across warehouses.
  - `{{ dbt_utils.date_spine(datepart="day", start_date="'2024-01-01'", end_date="current_date") }}` — generate a date dimension.
  - `{{ dbt_utils.union_relations(relations=[ref('stg_2019'), ref('stg_2020')]) }}` — union tables with matching columns.
  - `{{ dbt_utils.star(from=ref('stg_orders'), except=['raw_json']) }}` — select all columns except a few.
  - `{{ dbt_utils.pivot(...) }}` and `{{ dbt_utils.unpivot(...) }}` — pivot/unpivot transformations.
  - `{{ dbt_utils.datediff('start_date', 'end_date', 'day') }}` — cross-dialect date difference.
- Use `{{ dbt_utils. }}` namespace by adding the package to `packages.yml` and running `dbt deps`.

### Why macros reduce duplication

- Write a transformation once (e.g., cleansing logic, surrogate key generation, standard dedup) and reuse it across every model.
- Standardizes business logic: one definition of "active customer" or "converted revenue" used everywhere.
- Makes SQL portable across warehouses (dbt_utils abstracts dialect differences).
- Centralizes maintenance: fix a bug in a macro and every model using it is fixed in the next run.

---

## dbt Packages

### What are dbt packages

- dbt packages are **reusable libraries of models, macros, tests, and documentation** published on the dbt Hub (hub.getdbt.com) or hosted in git.
- They let you install vetted, community-built functionality instead of reinventing it.
- Common examples: dbt_utils, dbt_expectations, dbt_metrics, dbt_artifacts, codegen, dbt_date.

### packages.yml and installing packages

- `packages.yml` at the project root declares dependencies.

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
  - package: dbt-labs/codegen
    version: "0.10.0"
  - package: calogica/dbt_expectations
    version: "0.9.0"
  - git: "https://github.com/company/internal-package.git"
    revision: main
```

- Run `dbt deps` to install them. Installed packages go into `dbt_packages/` (ignored by git).
- Pin versions with ranges (`[">=1.0.0", "<2.0.0"]`) or exact versions for reproducibility.
- Package macros are namespaced: `{{ dbt_utils.surrogate_key(...) }}`.
- Package models can be selected with `--select package:` and disabled in `dbt_project.yml`.

### dbt_utils, dbt_expectations, codegen common packages

- **dbt_utils** — general-purpose utilities: surrogate keys, date spine, unions, pivots, star/except column selection, cross-database functions.
- **dbt_expectations** — a large library of "great-expectations-style" generic tests: `expect_column_values_to_be_between`, `expect_row_count_to_equal`, `expect_column_values_to_be_unique`, distribution checks, `expect_column_mean_to_be_between`, etc. Great for advanced data quality.
- **codegen** — code generation macros to bootstrap projects:
  - `{{ codegen.generate_model_yaml(model_name='stg_orders') }}` — auto-generate schema.yml from a model's columns.
  - `{{ codegen.generate_base_model(source_name='raw', table_name='orders') }}` — scaffold a staging model with column renames/casts.
  - `{{ codegen.generate_source_yaml(database_name='ANALYTICS_DB', schema_name='RAW_DATA') }}` — scaffold a sources.yml.
- **dbt_metrics** (now partially superseded by the dbt semantic layer) — define reusable metrics (e.g., revenue, active users) computed over models.
- **dbt_date** — calendar/date utilities across warehouses.

### dbt deps command

- `dbt deps` — reads `packages.yml` and installs/updates all declared packages.
- Always run `dbt deps` after cloning a repo or editing `packages.yml`.
- The installed code lives in `dbt_packages/`, which is listed in `clean-targets` and removed by `dbt clean`.
- In dbt Cloud/CI, run `dbt deps` before `dbt run`/`dbt build` so models compile against installed packages.

---

## dbt and Snowflake Integration

### Setting up Snowflake connection in profiles.yml

```yaml
# ~/.dbt/profiles.yml
snowflake_profile:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: xyz12345.us-east-1
      user: DBT_USER
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: TRANSFORM_ROLE
      database: ANALYTICS_DB
      warehouse: TRANSFORM_WH
      schema: DBT_DEV
      threads: 4
      client_session_keep_alive: false
```

- Connection fields for Snowflake:
  - `account` — account identifier (e.g., `abc12345.us-east-1`, `xyz.us-east-2.aws`).
  - `user` / `password` — Snowflake user credentials.
  - `role` — Snowflake role used for the session (recommend a dedicated transform role, not ACCOUNTADMIN).
  - `database` / `schema` / `warehouse` — defaults for the session.
  - `threads` — parallelism for model execution (Snowflake scales well; 4-8 typical).
  - Optional: `private_key_path`/`private_key_passphrase` for key-pair auth, `authenticator: externalbrowser` for SSO/OKTA.
- Best practice: create a dedicated service user + role for dbt with grants scoped to the analytics database and its schemas; never use ACCOUNTADMIN in production.

### Warehouse, database, schema, role

- **Warehouse**: compute resource. dbt needs a warehouse that can run the SQL for models. You can set a default warehouse in the profile and override per model with `snowflake_warehouse`.
- **Database**: the logical container (e.g., `ANALYTICS_DB`). All dbt objects typically live in one database.
- **Schema**: namespaced container within the database. dbt's recommended pattern is a schema per layer: `staging`, `intermediate`, `marts`, `snapshots`, and dev schemas like `dbt_<name>` for each developer.
- **Role**: governs privileges. Grant the dbt role USAGE on the warehouse/database and appropriate privileges (CREATE TABLE/VIEW, SELECT, INSERT) on the schemas dbt touches.
- Use dbt's `generate_schema_name` macro or per-folder `+schema:` config to control where each model lands.

### dbt best practices with Snowflake

- **Use the `merge` incremental strategy** for upserting changed rows; Snowflake's MERGE handles `unique_key`-based updates efficiently.
- **Leverage automatic clustering** on large tables: define `cluster_by` on frequently-filtered columns (e.g., `order_date`) and Snowflake maintains clustering automatically.
- **Understand micro-partitions**: Snowflake stores data in ~16MB immutable micro-partitions. Good clustering + pruning means queries only scan relevant partitions — this is why `cluster_by` on large tables matters.
- Use **small warehouses for staging/light models and larger warehouses for heavy marts** via `snowflake_warehouse` config per model.
- Prefer `timestamp`-based incremental filters over `check` snapshots where possible (fewer full scans).
- **Monitor credit spend**: cache-heavy CI runs and dev runs; use `target`-specific warehouse sizing (small for dev).

### Incremental strategies in Snowflake

- `merge` (default, recommended) — does an UPSERT keyed on `unique_key`: inserts new rows and updates changed rows in one statement. Best when rows can be updated.
- `delete+insert` — deletes rows matching `unique_key` from the target that appear in the source, then inserts all source rows. Useful when columns are `NOT NULL`-constrained or when you want a clean replace; slightly less efficient than merge.
- `append` — plain `INSERT`, used when `unique_key` is not set. Only add rows, never update/delete (must guarantee no re-runs create duplicates).
- Example config:

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge',
    cluster_by=['order_date'],
    snowflake_warehouse='TRANSFORM_WH_LARGE'
) }}
```

### Snowflake-specific configs in dbt

- `snowflake_warehouse` — override the warehouse for a model (or folder via dbt_project.yml `+snowflake_warehouse:`).
- `cluster_by` — column(s) to cluster on (list for multiple); works on `table` and `incremental` materializations.
- `materialized='incremental'` with `incremental_strategy` — see above.
- `transient` — create transient tables to avoid fail-safe storage costs.
- `secure` — create secure views (restrict downstream exposure of underlying tables).
- `query_tag` — set a query tag so Snowflake logs from dbt runs are identifiable (great for cost attribution).
- `copy_grants` — preserve grants when recreating tables.
- `merge_update_columns` — for merge strategy, restrict which columns get updated on conflict.

---

## dbt Cloud vs dbt Core

### dbt Core

- Open source, MIT-licensed **command-line tool**.
- Free forever; you install it via pip (`pip install dbt-snowflake`, `dbt-bigquery`, etc.), brew, or Docker.
- Workflow: you write code locally, run `dbt run`/`dbt test` from your terminal, and use an external orchestrator (Airflow, Dagster, Prefect, cron) for scheduling.
- Credentials live in `profiles.yml` on the machine/CI runner.
- Full flexibility and control; no hosted infrastructure required.
- You own: scheduling, retries, alerting, environments, secrets management.

### dbt Cloud

- Managed SaaS offering by dbt Labs.
- Key features:
  - **Web IDE** — write/edit models, preview results, run jobs in the browser.
  - **Jobs & scheduling** — define jobs (e.g., "dbt build --select marts") run on a schedule with retries, timeouts, and success/failure notifications.
  - **Managed environments & credentials** — dev/prod environments, no `profiles.yml` to maintain; secrets stored safely.
  - **CI/CD** — "dbt Cloud build" runs tests on pull requests automatically; with deferred environments, CI runs only changed models and their dependencies (fast, cheap).
  - **Documentation & lineage** — hosts your dbt docs site and gives an interactive lineage/impact view.
  - **Semantic layer** — define metrics and query them via API/BI connectors.
  - **Artifacts & metadata API** — run results, exposures, and lineage exposed via API.
- Requires a paid subscription beyond the Developer free tier.

### When to choose each

- **Choose dbt Core when**: you want zero cost, you already have robust orchestration (Airflow etc.), you prefer full control, or you're a solo developer/Analyst in a small team.
- **Choose dbt Cloud when**: you want managed scheduling + CI/CD out of the box, no infrastructure to run, centralized credential management for a team, hosted docs, and minimal setup friction.
- Many teams start with Core and migrate to Cloud as team size and governance needs grow. Core and Cloud use the same project code, so migration is low-risk.

### dbt run vs dbt build vs dbt test vs dbt compile

- `dbt run` — materializes models (view/table/incremental) and runs snapshots selected; does not run tests.
- `dbt test` — runs tests against existing relations (models must already be built).
- `dbt build` — runs, in order: seeds, snapshots, models, and their tests. The "one command to rule them all" for a full pipeline.
- `dbt compile` — generates the final SQL for every node (writes to `target/`) **without executing** anything. Useful for debugging/benchmarking the compiled SQL or for feeding compiled SQL to other tools.
- `dbt run-operation <macro>` — executes a macro directly (e.g., `dbt run-operation generate_schema_name`), often used for one-off admin tasks.

---

## dbt Workflow

### dbt run

```bash
dbt run
```

- Builds all models in dependency order using their materializations.
- Common flags:
  - `dbt run --select stg_orders` — run a single model.
  - `dbt run --select +fact_orders` — run the model and all its upstream dependencies.
  - `dbt run --select fact_orders+` — run the model and everything downstream of it.
  - `dbt run --select stg_*` — run all models matching the glob.
  - `dbt run --select tag:daily` — run all models tagged `daily`.
  - `dbt run --exclude staging` — skip a folder.
  - `dbt run --full-refresh` — force full rebuild of incremental/table models.

### dbt test

```bash
dbt test
```

- Runs all tests in the project.
- Selection flags work the same as `dbt run` (`--select`, `--exclude`, tags).
- `dbt test --store-failures` — persist failing rows to a table for investigation.
- Note: tests reference materialized relations, so run `dbt run` (or `dbt build`) first.

### dbt build

```bash
dbt build
```

- Loads seeds, runs snapshots, builds models, and runs tests in dependency order — all in one command.
- Stops a branch of the DAG if a model or its test fails, preventing bad data from propagating downstream.
- The recommended command for scheduled production runs and CI.
- `dbt build --full-refresh` — full rebuild plus tests.

### dbt run --select and --exclude

Selection syntax cheatsheet:

```bash
dbt run --select stg_orders             # one model
dbt run --select stg_orders,int_orders  # two models (comma)
dbt run --select +fact_orders           # model + upstream deps
dbt run --select fact_orders+           # model + downstream deps
dbt run --select +fact_orders+          # both directions
dbt run --select stg_*                  # wildcard
dbt run --select models/staging         # path selection
dbt run --select tag:finance            # by tag
dbt run --select source:raw             # all models of a source
dbt run --select package:dbt_utils      # all models in a package
dbt run --select fact_orders --exclude stg_*  # combined
```

- `+` before a node = "and everything upstream"; `+` after = "and everything downstream".
- Selection speeds up dev loops and lets CI test only what changed.

### dbt run --full-refresh

```bash
dbt run --full-refresh
dbt run --select fact_events --full-refresh
```

- Forces dbt to drop and rebuild incremental (and table) models completely.
- Use when:
  - Your incremental logic changed (new columns, new filter).
  - You need to backfill historical rows.
  - The incremental table has drifted or has bad data.
- Expensive on large tables — schedule intentionally, not on every run.

### dbt compile

```bash
dbt compile
dbt compile --select fact_orders
```

- Renders the final SQL for selected nodes into `target/compiled/<project>/...`.
- Does not connect to/execute against the warehouse (except for Jinja that needs `adapter.` functions).
- Great for debugging Jinja logic, sharing compiled SQL with DBAs, or capturing SQL for external tools.

### dbt docs generate / dbt docs serve

```bash
dbt docs generate
dbt docs serve
```

- `dbt docs generate` — creates the documentation site from your YAML descriptions plus the DAG from `ref()`/`source()` calls. Output in `target/catalog.json`, `manifest.json`, etc.
- `dbt docs serve` — serves the docs locally (default port 8080).
- Docs site features:
  - Lineage graph of models.
  - Column-level descriptions and tests.
  - Source freshness status.
  - Search across models/columns.
- `exposures` in YAML connect models to downstream dashboards/reports so docs show business impact.

### CI/CD with dbt

- **CI flow** (on every pull request):
  1. Clone the repo, `dbt deps`.
  2. Run tests on changed models: `dbt build --select state:modified+` (or use `dbt-cloud build` for deferred environments).
  3. Fail the PR if tests fail — never merge broken SQL.
- **State-based selection** uses `--state path/to/manifest.json` (with `--defer`) so CI builds only what changed and reuses production tables for everything else (huge cost/time savings).
- **Deferred runs**: in `dbt build --defer --state prod-manifest.json`, unselected upstream models are "deferred" to production relations instead of being rebuilt.
- **Production deployment**: schedule `dbt build` (or jobs in dbt Cloud) on a cron with alerting (Slack/email) on failure.
- **Typical jobs**:
  - Job 1: `dbt build --select staging+` — frequent (hourly).
  - Job 2: `dbt build --select marts` — less frequent (daily).
  - Job 3: `dbt snapshot` + `dbt source freshness` — daily.
- In dbt Cloud, use **environment variables** for credentials and the "CI job" type which automatically uses deferred state and creates a temp schema per PR.

---

## Common dbt Mistakes

### Writing model code that is too complex

- Monolithic 500-line models with dozens of CTEs are hard to review, test, and reuse.
- Fix: break into staging → intermediate → marts layers; one logical step per model.
- Keep each model focused: clean one thing, join one thing, aggregate one thing.

### Not using ref()

- Hardcoding table names (`FROM analytics_db.marts.fact_orders`) or using source tables directly in marts breaks the dependency graph.
- Fix: always `ref()` models and `source()` raw tables so dbt resolves names, orders runs, and tracks lineage correctly.
- Hardcoding also breaks environment parity (dev vs prod schemas differ).

### Not testing critical models

- Only writing tests for a handful of models means data quality bugs reach dashboards silently.
- Fix: test keys (unique/not_null), referential integrity, accepted values, and row-count sanity on staging and marts.

### Hardcoding schema/database names

- `FROM raw_data.customers` or `INSERT INTO prod_schema.x` breaks across environments and is brittle.
- Fix: use `source()`/`ref()` and `{{ target.schema }}`; let dbt manage the schema via `dbt_project.yml` and `generate_schema_name`.

### Too many heavy materializations

- Making every model a `table` leads to long runs and huge storage/compute bills.
- Fix: default staging to views, use tables only where performance matters, incremental only for genuinely large models.

### Ignoring source freshness

- No freshness checks means stale raw data silently propagates to reports.
- Fix: configure `freshness` + `loaded_at_field` on sources and run `dbt source freshness` in CI/scheduled jobs.

### Not using incremental models for large tables

- Rebuilding a 500M-row table every run is slow and expensive.
- Fix: use `incremental` with `is_incremental()` filter and `unique_key`; add `--full-refresh` only when needed.

### Other common mistakes worth mentioning

- Using `SELECT *` in marts (surprising schema changes break downstream).
- Forgetting `unique_key` on incremental models → duplicate rows on re-run.
- Filtering on `updated_at` with `>` instead of `>=` (misses rows with identical timestamps).
- Putting all models in one schema (no layer separation) → permission and governance problems.
- Not versioning packages (`packages.yml` without pinned versions) → build breakage.
- Running `dbt run` without `dbt deps` after cloning → compile errors.
- Not using `--full-refresh` after changing incremental logic → stale/wrong table.
- Deploying directly to prod without testing on PR → broken pipelines.

---

## Interview Questions

### ELT vs ETL

**Q1. Explain the difference between ETL and ELT, and where dbt fits in.**

- ETL: transform before loading into the warehouse, typically on a dedicated server (costly for big data, hard to scale).
- ELT: load raw data first, transform in the warehouse using the warehouse's compute.
- dbt is the **T in ELT** — it transforms already-loaded raw data using SQL, running entirely in the warehouse. This is why dbt works with modern cloud warehouses: you can leverage their massive compute and storage directly.

**Q2. Why did the industry move from ETL to ELT?**

- Cloud warehouses (Snowflake, BigQuery, Databricks) decoupled storage from compute and made storage cheap; it became faster/cheaper to load raw data and transform on demand.
- ELT keeps raw data available for reprocessing — new transformations can be applied to full history without reloading.
- ELT scales horizontally with warehouse compute; ETL servers became a bottleneck.
- Transformation logic in ELT is code (dbt SQL), which is version-controlled, testable, and auditable — unlike opaque ETL GUIs.

### dbt vs hand-written SQL

**Q3. Why use dbt instead of just writing SQL scripts and scheduling them?**

- Dependency graph: dbt determines run order from `ref()` — hand-written scripts require fragile manual ordering.
- Modularity/DRY: reuse models and macros; hand-written SQL is duplicated across scripts.
- Testing: built-in tests (unique, not_null, relationships) catch bad data; hand-written SQL rarely has assertions.
- Documentation: auto-generated docs + lineage; hand-written SQL has no docs.
- Version control + code review: dbt projects are git-native with PR-based workflows.
- Incremental/materialization handling: dbt generates correct DDL/DML per warehouse (MERGE vs INSERT etc.) — you'd write it by hand otherwise.
- Portability: same project runs on Snowflake, BigQuery, Redshift, Postgres by swapping the profile.

**Q4. What is an analytics engineer, and what role does dbt play in that role?**

- Analytics engineering sits between data engineering (infrastructure, ingestion) and analytics (BI, insights).
- The analytics engineer owns the transformation layer: modeling data into clean, documented, tested datasets.
- dbt is the primary tool: SQL-based, version-controlled, with tests and docs built in — enabling a code-first approach to analytics.

### ref vs source

**Q5. What is the difference between `ref()` and `source()` in dbt?**

- `ref('model_name')` — references models, seeds, and snapshots your project builds. Registers the dependency in the DAG; resolves to the correct schema/database for the current target.
- `source('source_name', 'table_name')` — references raw warehouse tables ingested by external tools, defined in sources.yml. Also registers a dependency (for ordering/tests/freshness) but the table is not created by dbt.
- Rule of thumb: `source()` for raw inputs, `ref()` for everything else.

**Q6. What happens when you change a column in an upstream model?**

- Because downstream models use `ref()`, dbt rebuilds them and the new column is available. Tests run against affected nodes; docs lineage shows impact.
- If a downstream model does `SELECT *`, the new column flows through (which can be surprising — hence avoid `SELECT *` in marts). If it selects columns explicitly, nothing breaks but you may need to add the new column where needed.

### Materializations

**Q7. Explain the four materializations in dbt and when to use each.**

- `view` (default): stored query, always current, cheap; best for staging/light models. Slower reads for heavy transforms.
- `table`: physical snapshot, rebuilt per run; best for frequently-queried marts. Longer runs, storage cost.
- `incremental`: only new/changed rows processed; best for large append-heavy facts. Fast, but needs correct `unique_key` + filter.
- `ephemeral`: compiled into referencing models as a CTE; no warehouse object; best for small reusable logic. Not queryable/testable directly.

**Q8. What is the default materialization in dbt, and how do you change it?**

- Default is `view`.
- Change via `{{ config(materialized='table') }}` in the model, via `+materialized:` in `dbt_project.yml` per folder, or per run with `dbt run --full-refresh` (which is not a materialization change but affects rebuild behavior).

### Incremental models

**Q9. How do incremental models work? Walk through the SQL.**

- On first run, the table is built fully (the `is_incremental()` branch is False).
- On subsequent runs, `is_incremental()` is True, so the filter applies — only rows newer than `MAX(updated_at)` in `{{ this }}` are processed.
- With `unique_key`, dbt uses a merge strategy to upsert changed rows instead of blind appending; without it, behavior is append-only.

```sql
{{ config(materialized='incremental', unique_key='order_id', incremental_strategy='merge') }}

SELECT * FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
{% endif %}
```

**Q10. What happens if you don't specify a `unique_key` on an incremental model?**

- dbt uses the `append` strategy: it only inserts new rows, never updates/removes.
- Re-running a job (e.g., reprocessing a window) creates duplicates — the append strategy assumes source rows never repeat.
- If sources can be updated, use `unique_key` + `merge` (or `delete+insert`) so changed rows are updated and deleted rows can be removed.

**Q11. When would you use `dbt run --full-refresh`?**

- After changing incremental logic (new columns, different filter).
- To backfill history or fix drifted/bad data in the incremental table.
- When switching materialization or `unique_key` config.
- It drops and rebuilds the whole table — use sparingly on large tables due to cost/time.

**Q12. What are the main incremental strategies in Snowflake?**

- `merge` — upsert by `unique_key` (update + insert). Default and recommended.
- `delete+insert` — delete matching `unique_key` rows from target, then insert source rows. Works well when target columns are NOT NULL constrained or you want clean replaces.
- `append` — plain insert, no upserting. Only when rows never change after load.

### Tests

**Q13. What are the built-in dbt tests, and how do they work under the hood?**

- `unique`, `not_null`, `relationships`, `accepted_values`.
- Under the hood each generic test is a Jinja macro that generates a `SELECT` returning failing rows; dbt counts them — zero rows = pass, any rows = fail.

**Q14. What is the difference between a singular test and a generic test?**

- Singular: a one-off SQL file in `tests/` (e.g., "order totals must be positive"). Not parameterized.
- Generic: a reusable Jinja macro in `macros/` (e.g., `test assert_is_greater_than(model, column_name, min_value)`) that can be declared for many models/columns in YAML.
- Use singular for specific one-time assertions, generic for repeatable checks.

**Q15. How do you store failing test rows for debugging?**

- `dbt test --store-failures` persists failing rows into tables (default schema `dbt_test__audit`).
- Good for investigating why a test failed without re-running logic manually.

### Snapshots

**Q16. What are dbt snapshots and how do they implement SCD Type 2?**

- Snapshots track full change history of a table.
- Configuration: `unique_key`, `strategy` (`timestamp` with `updated_at`, or `check` with `check_cols`), `target_schema`.
- dbt adds `dbt_scd_id`, `dbt_updated_at`, `dbt_valid_from`, `dbt_valid_to`. On a change, the old row gets `dbt_valid_to` set and a new row is inserted with `dbt_valid_from`.
- Current records have `dbt_valid_to IS NULL`.
- Use for slowly-changing entity/reference data (customer profiles, product metadata); not for facts or high-churn event data.

**Q17. When would you use a snapshot vs just keeping a `updated_at` column?**

- A plain `updated_at` only tells you *when* the current version changed — you can't reconstruct past states.
- Snapshots keep every version, so you can query "what did this record look like on date X" and audit every change. Trade-off: more storage and slightly more complex queries.

### Jinja

**Q18. What is Jinja and why does dbt use it?**

- Jinja is a templating engine (Python) that lets you write dynamic SQL — conditionals, loops, variables, and reusable macros.
- It powers `ref()`, `config()`, `var()`, `is_incremental()`, and user-defined macros, making models DRY and environment-aware.
- Compiled to plain SQL before execution; `dbt compile` shows the rendered output.

**Q19. What is the difference between `{{ }}`, `{% %}`, and `{# #}` in Jinja?**

- `{{ expression }}` — outputs a value (e.g., `{{ ref('stg_orders') }}`).
- `{% statement %}` — control flow, loops, macro definitions (e.g., `{% if is_incremental() %}`, `{% for ... %}`).
- `{# comment #}` — stripped from compiled SQL.

**Q20. What is `dbt_utils` and give examples of macros you've used.**

- A community package of reusable cross-database macros.
- Examples: `generate_surrogate_key`, `date_spine`, `union_relations`, `star`, `pivot`/`unpivot`, `datediff`.
- Installed via `packages.yml` + `dbt deps`.

### dbt Cloud vs Core

**Q21. dbt Cloud vs dbt Core: what are the trade-offs?**

- Core: free, open-source CLI, full control; you own scheduling/credentials/CI; requires an external orchestrator.
- Cloud: managed IDE, scheduling, credentials, CI/CD, hosted docs, semantic layer; costs money and adds vendor dependency.
- Both share the same project format — switching is straightforward.

**Q22. How do you schedule dbt jobs in a production environment?**

- dbt Core: external orchestrator (Airflow, Dagster, Prefect, cron) that runs `dbt deps`, `dbt build`, `dbt source freshness`, with retries and alerting.
- dbt Cloud: native jobs with cron schedules, notifications, retries; job types for production and CI.

### dbt in CI/CD

**Q23. Describe a dbt CI/CD pipeline you would set up.**

- On PR: run `dbt deps` then `dbt build --select state:modified+` (or Cloud CI job with deferred state) to build/test only changed models. Fail the PR on test failures.
- On merge to main: production job runs full `dbt build` on a schedule; deploy artifacts (manifest.json) for state-based selection.
- Add `dbt source freshness` alerting and store `dbt build --store-failures` audit tables.
- Use environment-specific targets (dev/prod) so no hardcoded names leak between environments.

**Q24. What are dbt artifacts, and how are they used in CI/CD?**

- JSON artifacts produced on runs: `manifest.json`, `run_results.json`, `catalog.json`, `sources.json`.
- `manifest.json` + `--state` enable **state-based selection** (`--select state:modified`), so CI builds only changed nodes.
- `--defer` (with state) uses production relations for unbuilt upstream nodes, so CI doesn't rebuild the whole DAG — major speed/cost win.

**Q25. What does `--select state:modified+` do?**

- Selects nodes modified since the reference state (plus their upstream/downstream as indicated by `+`), so you only build/test what actually changed in the PR. Relies on a `--state manifest.json` from the last successful prod run.

### Additional common questions

**Q26. What is a DAG in dbt?**

- A directed acyclic graph of all nodes (models, seeds, snapshots, tests, sources) and their dependencies derived from `ref()`/`source()` calls.
- Drives run order, incremental rebuild impact, docs lineage, and targeted testing. It must stay acyclic — circular refs error out.

**Q27. How does dbt handle environments (dev vs prod)?**

- Through `target` in profiles.yml and Jinja: `{{ target.name }}`, `{{ target.schema }}`. Same code, different schemas/warehouses.
- Schema isolation: dev models land in `dbt_<username>` schemas, prod in `staging/marts/snapshots` — via `dbt_project.yml` and `generate_schema_name`.

**Q28. What is a seed and when should you use it?**

- A CSV in `seeds/` loaded into the warehouse by `dbt seed`, referenceable via `ref()`.
- Use for small static reference data (state codes, mappings, calendar). Not for large/dynamic data — use sources + ingestion.

**Q29. How do you handle secrets in dbt?**

- `env_var('DBT_PASSWORD')` in profiles.yml; environment variables set in your shell or dbt Cloud environments. Never hardcode secrets in the repo; gitignore `profiles.yml`.

**Q30. What is the difference between `dbt compile` and `dbt run`?**

- `compile` renders final SQL to `target/compiled/` without executing; `run` executes the materializations against the warehouse. Compile is for debugging/inspection.

**Q31. How do you handle changing schemas in upstream source tables?**

- Define sources with explicit columns/tests; add `dbt` column-level tests to catch breakage; use `codegen` to regenerate YAML; avoid `SELECT *` where stability matters; snapshot slowly-changing reference data.
- Monitor via source freshness and PR CI tests.

**Q32. What are dbt exposures?**

- YAML declarations (`exposures:`) linking models to downstream dashboards/reports (Looker, Tableau) and their owners.
- Exposures appear in docs lineage so you see what business assets depend on a given model — useful for impact analysis and ownership.

**Q33. How would you debug a model that returns unexpected results?**

- `dbt compile --select <model>` to inspect rendered SQL.
- `dbt run-operation` or `dbt compile` with a subset selection.
- Test intermediate models individually with `--select`.
- Check tests upstream (`dbt test --select staging`), verify source freshness, and review incremental filter logic (did `--full-refresh` change behavior?).

**Q34. What are the benefits of dbt packages and how do you install them?**

- Reuse community-tested macros/models/tests; standardize logic; speed up development.
- Declare in `packages.yml`, run `dbt deps`, then call with namespace e.g. `{{ dbt_utils.surrogate_key([...]) }}`. Pin versions for reproducibility.

**Q35. How do you ensure data quality in a dbt project?**

- Layered tests: source tests (freshness, not_null keys), staging tests (dedupe checks), mart tests (business rules, relationships, accepted_values).
- CI gating on PRs, `dbt build` in prod with failure alerts, `--store-failures` for debugging, and a documented data dictionary.
