# Module 5: Data Platforms with Bruin

## Table of Contents

1. [What is Bruin?](#1-what-is-bruin)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts: Projects](#3-core-concepts-projects)
4. [Core Concepts: Pipelines](#4-core-concepts-pipelines)
5. [Core Concepts: Assets](#5-core-concepts-assets)
6. [Core Concepts: Variables](#6-core-concepts-variables)
7. [Core Concepts: Commands](#7-core-concepts-commands)
8. [Building an End-to-End Pipeline (NYC Taxi)](#8-building-an-end-to-end-pipeline-nyc-taxi)
9. [Materialization Strategies](#9-materialization-strategies)
10. [Data Quality Checks](#10-data-quality-checks)
11. [Bruin MCP (AI Integration)](#11-bruin-mcp-ai-integration)
12. [Bruin Cloud (Deployment)](#12-bruin-cloud-deployment)
13. [Homework Guide](#13-homework-guide)

---

## 1. What is Bruin?

Bruin is an **end-to-end data platform** that consolidates the entire modern data stack into a single tool. Instead of wiring together multiple separate services, Bruin handles:

| Capability | What it replaces |
|---|---|
| Data ingestion | Fivetran, Airbyte |
| Data transformation | dbt |
| Orchestration | Airflow, Prefect, Dagster |
| Data quality | Great Expectations, dbt tests |
| Metadata & lineage | OpenLineage, Datahub |

### The Modern Data Stack (context)

A typical pipeline flow:
1. **Extract** data from third-party sources or databases
2. **Load** into a data warehouse or data lake
3. **Transform** — clean, model, aggregate
4. **Orchestrate** — schedule and manage dependencies
5. **Quality** — validate accuracy and completeness before delivery to consumers

Bruin unifies all five steps so you don't need to be a DevOps engineer, data architect, and pipeline engineer simultaneously.

---

## 2. Installation & Setup

### Install Bruin CLI

```bash
curl -LsSf https://getbruin.com/install/cli | sh
bruin version
```

### IDE Extension

Install the **Bruin extension** for VS Code or Cursor. This adds a Bruin render panel for:
- Viewing compiled SQL with actual variable values
- Running assets and pipelines directly from the IDE
- Viewing the lineage graph

### Initialize a project

```bash
# From the default template
bruin init default my-first-pipeline

# From the zoomcamp template (NYC taxi)
bruin init zoomcamp my-taxi-pipeline

cd my-taxi-pipeline
```

Bruin requires the project to be git-initialized. `bruin init` handles this automatically and adds `.bruin.yml` to `.gitignore`.

---

## 3. Core Concepts: Projects

A **project** is the root directory for your entire Bruin data pipeline. It is the entry point the CLI uses to understand directory structure, locate assets, and resolve connections.

### Directory structure

```text
my-pipeline/
├── .bruin.yml          # Environments & connections (LOCAL ONLY, never commit)
└── pipeline/
    ├── pipeline.yml    # Pipeline configuration
    └── assets/
        ├── ingestion/
        ├── staging/
        └── reports/
```

### `.bruin.yml` — Environment & Connection Config

This file is **always added to `.gitignore`** — it contains secrets and must never be pushed to your repo.

```yaml
default_environment: default

environments:
  default:
    connections:
      duckdb:
        - name: duckdb-default
          path: duckdb.db

  production:
    connections:
      bigquery:
        - name: bq-prod
          project: my-gcp-project
          dataset: production
      motherduck:
        - name: motherduck
          token: <your-token>
```

**Built-in connection types:**
- DuckDB, MotherDuck
- PostgreSQL, MySQL
- BigQuery, Redshift, Snowflake
- Custom connections (API keys, secrets)

**Why use multiple environments?**
- Run locally without touching production data
- Different teams get different credential access
- Default to `dev` to prevent accidental production runs

---

## 4. Core Concepts: Pipelines

A **pipeline** is a grouping mechanism for assets that share the same schedule and execution context. A project can contain multiple pipelines.

### Key rule: one schedule per pipeline

If assets need to run hourly vs. daily, they belong in separate pipelines.

### Pipeline directory structure

```text
project/
├── .bruin.yml
└── pipelines/
    ├── nyc-taxi/
    │   ├── pipeline.yml
    │   └── assets/
    └── another-pipeline/
        ├── pipeline.yml
        └── assets/
```

### `pipeline.yml`

```yaml
name: nyc_taxi
schedule: monthly         # hourly | daily | monthly | cron expression
start_date: "2019-01-01"
default_connections:
  duckdb: duckdb-default
variables:
  taxi_types:
    type: array
    items:
      type: string
    default: ["yellow"]
```

| Setting | Description |
|---|---|
| `name` | Pipeline identifier |
| `schedule` | When to run |
| `start_date` | Historical start for full refresh |
| `default_connections` | Which connections this pipeline uses |
| `variables` | Custom runtime parameters |

**Connection scoping:** Even though connections are defined project-wide in `.bruin.yml`, each pipeline declares which ones it uses. This limits secret exposure and is important in multi-team environments.

---

## 5. Core Concepts: Assets

An **asset** is a single file that performs a specific task — almost always creating or updating a table/view in the destination database. Assets are the building blocks of a pipeline.

Each asset file has two parts:
1. **Definition** (metadata block) — name, type, connection, materialization, quality checks
2. **Content** (code) — the SQL query, Python function, or YAML config

### Asset types

| Type | File extension | Use case |
|---|---|---|
| SQL | `.sql` | Transformations, aggregations |
| Python | `.py` | Ingestion, data processing, ML |
| YAML/Seed | `.asset.yml` | Static lookup tables from CSV |
| R | `.r` | Statistical analysis |

### Naming convention

Asset names map to database schema and table names:
- `assets/ingestion/trips.py` → `ingestion.trips`
- `assets/staging/trips.sql` → `staging.trips`
- `assets/reports/trips_report.sql` → `reports.trips_report`

### SQL asset

```sql
/* @bruin
name: staging.trips
type: duckdb.sql

depends:
  - ingestion.trips
  - ingestion.payment_lookup

materialization:
  type: table
  strategy: time_interval
  incremental_key: pickup_datetime
  time_granularity: timestamp

columns:
  - name: pickup_datetime
    type: timestamp
    primary_key: true
    checks:
      - name: not_null
@bruin */

SELECT
    t.pickup_datetime,
    t.dropoff_datetime,
    t.fare_amount,
    p.payment_type_name
FROM ingestion.trips t
LEFT JOIN ingestion.payment_lookup p
    ON t.payment_type = p.payment_type_id
WHERE t.pickup_datetime >= '{{ start_datetime }}'
  AND t.pickup_datetime < '{{ end_datetime }}'
```

### Python asset

```python
"""@bruin
name: ingestion.trips
type: python
image: python:3.11

materialization:
  type: table
  strategy: append

columns:
  - name: pickup_datetime
    type: timestamp
    description: "When the meter was engaged"
@bruin"""

import os
import json
import pandas as pd

def materialize():
    start_date = os.environ["BRUIN_START_DATE"]
    end_date = os.environ["BRUIN_END_DATE"]
    taxi_types = json.loads(os.environ["BRUIN_VARS"]).get("taxi_types", ["yellow"])

    # fetch and build dataframe ...
    return final_dataframe   # Bruin handles inserting to the destination
```

The `materialize()` function returns a DataFrame; Bruin handles writing it to the database.

### YAML seed asset

```yaml
name: ingestion.payment_lookup
type: duckdb.seed
parameters:
  path: payment_lookup.csv
columns:
  - name: payment_type_id
    type: integer
    primary_key: true
    checks:
      - name: not_null
      - name: unique
  - name: payment_type_name
    type: string
    checks:
      - name: not_null
```

### Lineage & Dependencies

Declare `depends` in the asset definition to set execution order. When a dependency finishes, Bruin automatically triggers downstream assets. The VS Code extension renders the full lineage graph.

---

## 6. Core Concepts: Variables

Variables are **dynamically initialized at each pipeline run** and allow parameterization without changing code.

### Built-in variables (always available)

| Variable | Description |
|---|---|
| `start_date` / `BRUIN_START_DATE` | Beginning of the scheduled interval |
| `end_date` / `BRUIN_END_DATE` | End of the scheduled interval |

Schedule → date window mapping:

| Schedule | Start | End |
|---|---|---|
| Monthly | First day of month | Last day of month |
| Daily | Start of day | End of day |
| Hourly | Start of hour | End of hour |

**In SQL (Jinja templating):**

```sql
WHERE pickup_date >= '{{ start_date }}'
  AND pickup_date < '{{ end_date }}'
```

**In Python (environment variables):**

```python
import os, json

start_date = os.environ["BRUIN_START_DATE"]
end_date   = os.environ["BRUIN_END_DATE"]

# Custom variables
taxi_types = json.loads(os.environ["BRUIN_VARS"]).get("taxi_types", ["yellow"])
```

### Custom variables

Define in `pipeline.yml`:

```yaml
variables:
  taxi_types:
    type: array
    items:
      type: string
    default: ["yellow", "green"]
```

Override at runtime:

```bash
bruin run ./pipeline/pipeline.yml --var 'taxi_types=["yellow"]'
```

---

## 7. Core Concepts: Commands

### `bruin run` — Execute a pipeline

```bash
# Run entire pipeline
bruin run ./pipeline/pipeline.yml

# Run with a date range
bruin run ./pipeline/pipeline.yml \
  --start-date 2022-01-01 \
  --end-date 2022-02-01

# Full refresh (drops and recreates tables)
bruin run ./pipeline/pipeline.yml --full-refresh

# Run a specific asset and all downstream assets
bruin run ./pipeline/pipeline.yml --asset staging.trips --downstream

# Run with variable override and specific environment
bruin run ./pipeline/pipeline.yml \
  --var 'taxi_types=["yellow"]' \
  --environment production
```

### Common run flags

| Flag | Description |
|---|---|
| `--start-date DATE` | Set execution start date |
| `--end-date DATE` | Set execution end date |
| `--full-refresh` | Drop and recreate all tables |
| `--downstream` | Run asset plus all dependents |
| `--upstream` | Run asset plus all dependencies |
| `--environment ENV` | Use a specific environment (dev/prod) |
| `--var KEY=VALUE` | Override custom variable |

### `bruin validate` — Check configuration

```bash
bruin validate ./pipeline/pipeline.yml
```

Checks for:
- Circular dependencies
- Broken asset references
- Misconfigured connections
- Invalid materialization strategies

**Always validate before running.**

### `bruin lineage` — View dependency graph

```bash
bruin lineage ./pipeline/pipeline.yml
```

Outputs the upstream/downstream relationships between all assets.

### `bruin query` — Ad-hoc SQL

```bash
bruin query \
  --connection duckdb-default \
  --query "SELECT COUNT(*) FROM staging.trips"
```

---

## 8. Building an End-to-End Pipeline (NYC Taxi)

### Architecture: three-layer pattern

```
Ingestion layer  →  Staging layer  →  Reports layer
(raw data in)       (clean & join)     (aggregated out)
```

### Initialize from the zoomcamp template

```bash
bruin init zoomcamp my-taxi-pipeline
cd my-taxi-pipeline
```

### Project structure

```text
my-taxi-pipeline/
├── .bruin.yml
├── README.md
└── pipeline/
    ├── pipeline.yml
    └── assets/
        ├── ingestion/
        │   ├── trips.py               # Python: fetch from NYC taxi API
        │   ├── requirements.txt
        │   ├── payment_lookup.asset.yml  # Seed: CSV lookup table
        │   └── payment_lookup.csv
        ├── staging/
        │   └── trips.sql              # SQL: clean & join
        └── reports/
            └── trips_report.sql       # SQL: aggregate daily stats
```

### Layer 1 — Ingestion

**`ingestion/trips.py`** — fetches parquet files from the NYC taxi dataset URL:

```
https://d37ci6vzurychx.cloudfront.net/trip-data/{taxi_type}_tripdata_{year}-{month}.parquet
```

- Materialization strategy: `append` (new data added each run)
- Uses `BRUIN_START_DATE` / `BRUIN_END_DATE` to determine which months to fetch
- Uses `BRUIN_VARS` to read the `taxi_types` variable

**`ingestion/payment_lookup.asset.yml`** — loads `payment_lookup.csv` as a seed table with `not_null` and `unique` quality checks on `payment_type_id`.

### Layer 2 — Staging

**`staging/trips.sql`** — cleans and enriches raw trips:

- Depends on: `ingestion.trips`, `ingestion.payment_lookup`
- Strategy: `time_interval` — deletes rows in the window, then re-inserts
- Uses `QUALIFY ROW_NUMBER()` to deduplicate on composite key
- Joins payment lookup to add `payment_type_name`

### Layer 3 — Reports

**`reports/trips_report.sql`** — aggregates by day, taxi type, and payment type:

- Depends on: `staging.trips`
- Strategy: `time_interval` on `trip_date` (date granularity)
- Computes `trip_count`, `total_fare`, `avg_fare`

### Execution order (managed by Bruin)

```
ingestion.trips ──────┐
                      ├──→ staging.trips ──→ reports.trips_report
ingestion.payment_lookup ┘
```

Ingestion assets run first (in parallel where possible), then staging, then reports.

### Running the pipeline

```bash
# Validate first
bruin validate ./pipeline/pipeline.yml

# Test with a small range
bruin run ./pipeline/pipeline.yml \
  --start-date 2022-01-01 \
  --end-date 2022-02-01

# Full historical run
bruin run ./pipeline/pipeline.yml --full-refresh

# Check results
bruin query --connection duckdb-default \
  --query "SELECT COUNT(*) FROM ingestion.trips"
```

---

## 9. Materialization Strategies

| Strategy | Behavior | Best for |
|---|---|---|
| `append` | Inserts new rows, never touches existing data | Immutable event logs |
| `replace` | Truncates the entire table and rebuilds | Small reference tables |
| `time_interval` | Deletes rows in the time window, re-inserts | Incremental time-partitioned data |
| `merge` | Upsert based on key columns | Slowly changing dimensions |
| `delete+insert` | Deletes matching rows, then inserts | Custom key-based updates |
| `view` | Creates a virtual view (no data stored) | Lightweight transformations |

**When to use `time_interval`:**
- Data is organized by a timestamp column
- You want to reprocess specific intervals without a full rebuild
- The WHERE clause in the SQL **must** filter to the same time window

---

## 10. Data Quality Checks

Quality checks are defined in the asset's column definitions and run automatically after the asset completes.

### Built-in column checks

```yaml
columns:
  - name: pickup_datetime
    type: timestamp
    checks:
      - name: not_null
  - name: payment_type_id
    type: integer
    checks:
      - name: not_null
      - name: unique
  - name: trip_count
    type: bigint
    checks:
      - name: non_negative
```

| Check | Description |
|---|---|
| `not_null` | Column must have no NULL values |
| `unique` | All values must be distinct |
| `non_negative` | Numeric values must be >= 0 |
| `positive` | Numeric values must be > 0 |
| `accepted_values` | Only specific values are allowed |

### Custom checks (SQL-based)

```yaml
custom_checks:
  - name: row_count_greater_than_zero
    query: |
      SELECT CASE WHEN COUNT(*) > 0 THEN 1 ELSE 0 END
      FROM staging.trips
    value: 1
```

The query must return the expected `value` for the check to pass.

---

## 11. Bruin MCP (AI Integration)

**MCP = Model Context Protocol** — a standard for AI agents to interact with tools.

Bruin MCP lets AI agents in your IDE (Cursor, VS Code, Claude Code) communicate directly with Bruin to:
- Write pipeline code and asset configs
- Validate and run pipelines
- Query data conversationally
- Troubleshoot errors and debug issues
- Explain pipeline logic

### Installation

**Cursor:**
Go to **Settings → Tools & MCP → New MCP Server** and add:

```json
{
  "mcpServers": {
    "bruin": {
      "command": "bruin",
      "args": ["mcp"]
    }
  }
}
```

**VS Code (Copilot):**
Create `.vscode/mcp.json`:

```json
{
  "servers": {
    "bruin": {
      "command": "bruin",
      "args": ["mcp"]
    }
  }
}
```

**Claude Code:**

```bash
claude mcp add bruin -- bruin mcp
```

### Building a pipeline with MCP

The zoomcamp template includes a prompt in its generated `README.md`. Paste it into your AI agent to scaffold the entire pipeline end-to-end automatically.

When given the prompt, the agent will:
1. Create all asset files (ingestion, staging, reports)
2. Configure materialization and dependencies
3. Add quality checks and column metadata
4. Validate with `bruin validate`
5. Run with a test date range
6. Verify results with `bruin query`

### Conversational data analysis

Once the pipeline has run, query data naturally:
- "How many days of data do we have in the staging table?"
- "Which day had the highest trip count?"
- "Which asset is responsible for the aggregation logic?"

---

## 12. Bruin Cloud (Deployment)

Bruin Cloud is the **fully managed, hosted version** of Bruin. It runs the same open-source CLI engine but adds:
- Managed scheduling and infrastructure
- UI for monitoring runs and quality checks
- AI-powered metadata generation
- Data governance features

### Setup steps

1. Sign up at [getbruin.com](https://getbruin.com/)
2. Connect your GitHub repository (direct OAuth or Personal Access Token)
3. Set up connections (same names as in `.bruin.yml`, credentials stored securely in cloud)
4. Navigate to **Pipelines** — Bruin validates all assets and lineage
5. Enable the pipeline → it immediately runs for the last scheduled interval

### Connections in the cloud

Use the same connection names as your local `.bruin.yml`. Credentials are stored securely by Bruin Cloud — you provide them once in the UI.

### Monitoring

- Per-asset success/failure status
- Quality check results
- Full lineage visualization
- AI-powered conversational analysis of your data

---

## 13. Homework Guide

The homework covers Bruin CLI concepts using the zoomcamp template pipeline.

### Setup

```bash
# Install Bruin
curl -LsSf https://getbruin.com/install/cli | sh

# Initialize the zoomcamp template
bruin init zoomcamp my-pipeline
cd my-pipeline

# Configure .bruin.yml with DuckDB connection
# Edit .bruin.yml to add your DuckDB path
```

### Question-by-question answers

---

**Q1: Bruin Pipeline Structure**

> In a Bruin project, what are the required files/directories?

**Answer: `pipeline.yml` and `assets/` only**

Initializing with `bruin init zoomcamp` creates only `pipeline.yml` and `assets/` inside the pipeline directory. `.bruin.yml` is **not** part of the required pipeline structure — it is optional and used locally for environment/connection overrides (and is hidden by `.gitignore` when present).

---

**Q2: Materialization Strategies**

> Which incremental strategy processes a specific interval period by deleting and inserting data for that time period?

**Answer: `time_interval`**

The `time_interval` strategy:
1. Deletes rows in the pipeline's current time window
2. Re-inserts data from the SQL query for that same window

This makes it idempotent — rerunning the same interval always produces the same result. The `WHERE` clause in your SQL must filter to the same time window to avoid inconsistencies.

---

**Q3: Pipeline Variables**

> How do you override an array variable when running the pipeline to only process yellow taxis?

**Answer: `bruin run --var 'taxi_types=["yellow"]'`**

Array variables must be passed as a JSON array string. Correct syntax:

```bash
bruin run ./pipeline/pipeline.yml --var 'taxi_types=["yellow"]'
```

The `--var` flag accepts `KEY=VALUE` format. For arrays, the value is a JSON-encoded array string (use single quotes to avoid shell interpretation).

---

**Q4: Running with Dependencies**

> You've modified `ingestion/trips.py` and want to run it plus all downstream assets. Which command?

**Answer: `bruin run --select ingestion.trips+`**

The `+` suffix after an asset name means "and all downstream assets". This is the selector syntax for running a specific asset and its full downstream dependency chain.

```bash
bruin run --select ingestion.trips+
```

---

**Q5: Quality Checks**

> You want to ensure `pickup_datetime` never has NULL values. Which quality check?

**Answer: `name: not_null`**

```yaml
columns:
  - name: pickup_datetime
    type: timestamp
    checks:
      - name: not_null
```

The `not_null` check validates that every row in the column has a non-NULL value. It runs automatically after the asset completes.

---

**Q6: Lineage and Dependencies**

> Which command visualizes the dependency graph between assets?

**Answer: `bruin lineage`**

```bash
bruin lineage ./pipeline/pipeline.yml
```

This displays the directed acyclic graph (DAG) of asset dependencies, showing upstream and downstream relationships.

---

**Q7: First-Time Run**

> Running a Bruin pipeline for the first time on a new DuckDB database. What flag ensures tables are created from scratch?

**Answer: `--full-refresh`**

```bash
bruin run ./pipeline/pipeline.yml --full-refresh
```

`--full-refresh` overrides incremental strategies and drops/recreates all tables. Use this when:
- Running a pipeline for the first time (no existing tables)
- Fixing data quality issues that require a complete rebuild
- Backfilling historical data from `start_date`

### Submitting

Submit your answers at: https://courses.datatalks.club/de-zoomcamp-2026/homework/hw5
