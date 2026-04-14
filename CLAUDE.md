# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the **Data Engineering Zoomcamp** — a free 9-week online course teaching data engineering fundamentals through hands-on modules. The repository contains course materials, code examples, homework assignments, infrastructure templates, and a personal capstone project.

## Technology Stack by Module

| Module | Directory | Technologies |
|--------|-----------|-------------|
| 01 - Docker & Terraform | `01-docker-terraform/` | Docker, PostgreSQL, Terraform, GCP |
| 02 - Workflow Orchestration | `02-workflow-orchestration/` | Kestra (YAML workflows), Docker Compose |
| 03 - Data Warehousing | `03-data-warehouse/` | Google BigQuery, SQL |
| 04 - Analytics Engineering | `04-analytics-engineering/` | dbt (BigQuery or DuckDB), SQL |
| 05 - Data Platforms | `05-data-platforms/` | Bruin CLI |
| 06 - Batch Processing | `06-batch/` | Apache Spark, PySpark, Jupyter |
| 07 - Stream Processing | `07-streaming/` | Kafka, PyFlink, ksqlDB, Java |

## Common Commands

### Docker Services (Module 2 - Kestra)
```bash
cd 02-workflow-orchestration/
docker compose up -d      # Start Kestra + PostgreSQL
docker compose down       # Stop services
```

### Python Data Pipeline (Module 1)
```bash
cd 01-docker-terraform/docker-sql/pipeline/
pip install -e .          # Install pipeline package
python pipeline.py --help # CLI options
```
Dependencies managed via `pyproject.toml` (Python ≥3.13, uses pandas, psycopg2, SQLAlchemy, click).

### dbt (Module 4)
```bash
cd 04-analytics-engineering/taxi_rides_ny/
dbt deps                  # Install packages (dbt_utils, codegen)
dbt run                   # Run all models
dbt test                  # Run tests
dbt run --select <model>  # Run specific model
dbt docs generate         # Generate documentation
```
Switch between BigQuery and DuckDB targets via profiles. The `GCP_PROJECT_ID` env var controls the BigQuery project. Dev environment samples data from 2019-01-01 to 2019-02-01 via `vars`.

### Bruin (Module 5)
```bash
bruin run ./pipelines/nyc-taxi/pipeline.yml
bruin run ./pipelines/nyc-taxi/pipeline.yml --start-date 2020-01-01 --end-date 2020-01-31
bruin run ./pipelines/nyc-taxi/pipeline.yml --asset staging.trips_summary
bruin validate ./pipelines/nyc-taxi/pipeline.yml
bruin run ./pipelines/nyc-taxi/pipeline.yml --full-refresh --environment default
```

### Streaming (Module 7)
```bash
cd 07-streaming/python/
# One-time network/volume setup
docker network create kafka-spark-network
docker volume create --name=hadoop-distributed-file-system
# Start Kafka/Spark clusters
cd docker && docker compose up -d
# Run Kafka producer/consumer
python3 producer.py
python3 consumer.py
```

## Architecture

The course teaches an end-to-end data pipeline flow:
1. **Ingest** — Extract from APIs/files (Python, Docker)
2. **Store** — Raw data in cloud storage (GCS)
3. **Warehouse** — Structured data (BigQuery)
4. **Transform** — Analytical models (dbt)
5. **Orchestrate** — Schedule/monitor (Kestra or Bruin)
6. **Stream** — Real-time processing (Kafka, Flink)

## dbt Model Layers (Module 4)

The dbt project at `04-analytics-engineering/taxi_rides_ny/` follows a three-layer pattern:

- **`models/staging/`** — materialized as views; clean and cast raw source tables (`stg_green_tripdata`, `stg_yellow_tripdata`). Sources defined in `sources.yml` with freshness checks.
- **`models/intermediate/`** — materialized as tables; joins and unions across staging models (`int_trips`, `int_trips_unioned`).
- **`models/marts/`** — materialized as tables; final analytical models (`fct_trips`, `dim_vendors`, `dim_zones`) plus a `reporting/` sub-layer.

Source data lives in BigQuery dataset `nytaxi` or DuckDB schema `prod`, controlled by `target.type` Jinja logic in `sources.yml`.

## Capstone Project (`00_Final_Project/`)

Personal capstone building an FDA FAERS (drug adverse event) analytics pipeline. Tech stack mirrors the course: Terraform → GCS → BigQuery → dbt → Looker Studio, orchestrated by Kestra. Key considerations:
- FAERS uses `$` as delimiter in older quarters
- `primaryid` is unique per report version; `caseid` links versions — use latest per `caseid` in marts
- dbt model layout: `stg_faers_*` → `int_reports_enriched` → `mart_drug_safety_signals`, `mart_reaction_trends`, `mart_patient_demographics`

## Dataset

NYC Taxi Trip Records (yellow and green cabs, 2019-2020). Schema documented in `dataset.md`. Sample DuckDB database: `taxi_rides_ny.duckdb`.

## Key Directories

- `00_Final_Project/` — Capstone project plan and data exploration notebooks
- `00_reference-notes/` — Personal notes per module (Kestra, dbt, BigQuery, Bruin, Terraform)
- `cohorts/` — Year-specific homework materials (2022–2026)
- `02-workflow-orchestration/flows/` — Kestra workflow YAML definitions (01–11, including GCP setup and RAG examples)
- `04-analytics-engineering/taxi_rides_ny/` — Complete dbt project
- `06-batch/code/` — PySpark Jupyter notebooks and scripts
- `07-streaming/python/` — Kafka producers/consumers (JSON/Avro examples, Redpanda)

## Development Notes

- Each module is self-contained with its own README and setup instructions
- Cloud setup uses GCP (BigQuery, GCS) but local alternatives exist (DuckDB, local Spark)
- dbt uses variable-based environment switching (dev/prod) and `target.type` for BigQuery vs DuckDB
- Kestra workflows in `02-workflow-orchestration/flows/` are numbered sequentially and build on each other
