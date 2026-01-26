# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is the **Data Engineering Zoomcamp** - a free 9-week online course teaching data engineering fundamentals through hands-on modules. The repository contains course materials, code examples, homework assignments, and infrastructure templates.

## Technology Stack by Module

| Module | Technologies |
|--------|-------------|
| 01 - Docker & Terraform | Docker, PostgreSQL, Terraform, GCP |
| 02 - Workflow Orchestration | Kestra (YAML workflows), Docker Compose |
| 03 - Data Warehousing | Google BigQuery, SQL |
| 04 - Analytics Engineering | dbt (BigQuery or DuckDB), SQL |
| 05 - Batch Processing | Apache Spark, PySpark, Jupyter |
| 06 - Stream Processing | Kafka, PyFlink, ksqlDB, Java |

## Common Commands

### Docker Services (Module 2 - Kestra)
```bash
cd 02-workflow-orchestration/
docker compose up -d      # Start Kestra + PostgreSQL
docker compose down       # Stop services
```

### dbt (Module 4)
```bash
cd 04-analytics-engineering/taxi_rides_ny/
dbt run                   # Run all models
dbt test                  # Run tests
dbt run --select <model>  # Run specific model
dbt docs generate         # Generate documentation
```

### Python Data Pipeline (Module 1)
```bash
cd 01-docker-terraform/docker-sql/pipeline/
pip install -e .          # Install pipeline package
python pipeline.py --help # CLI options
```

## Architecture

The course teaches an end-to-end data pipeline flow:
1. **Ingest** - Extract from APIs/files (Python, Docker)
2. **Store** - Raw data in cloud storage (GCS)
3. **Warehouse** - Structured data (BigQuery)
4. **Transform** - Analytical models (dbt)
5. **Orchestrate** - Schedule/monitor (Kestra)
6. **Stream** - Real-time processing (Kafka, Flink)

## Key Directories

- `01-docker-terraform/` - Docker + Terraform fundamentals
- `02-workflow-orchestration/flows/` - Kestra workflow YAML definitions
- `04-analytics-engineering/taxi_rides_ny/` - Complete dbt project with staging/marts
- `05-batch/code/` - PySpark processing examples
- `06-streaming/` - Kafka producers/consumers in Python and Java
- `cohorts/` - Year-specific homework and materials (2022-2026)

## Dataset

NYC Taxi Trip Records (yellow and green cabs, 2019-2020). Schema documented in `dataset.md`. Sample DuckDB database included: `taxi_rides_ny.duckdb`.

## Development Notes

- Each module is self-contained with its own README and setup instructions
- Cloud setup uses GCP (BigQuery, GCS) but local alternatives exist (DuckDB, local Spark)
- Kestra workflows defined in YAML at `02-workflow-orchestration/flows/`
- dbt project uses variable-based environment switching (dev/prod)
