# Module 2: Workflow Orchestration with Kestra - Follow-Through Tutorial

This hands-on tutorial guides you through building data pipelines with Kestra, from basic concepts to production-ready cloud workflows.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Environment Setup](#environment-setup)
3. [Part 1: Hello World - Core Concepts](#part-1-hello-world---core-concepts)
4. [Part 2: Python Integration](#part-2-python-integration)
5. [Part 3: Your First ETL Pipeline](#part-3-your-first-etl-pipeline)
6. [Part 4: Loading Data to PostgreSQL](#part-4-loading-data-to-postgresql)
7. [Part 5: Scheduling and Backfills](#part-5-scheduling-and-backfills)
8. [Part 6: GCP Setup and Configuration](#part-6-gcp-setup-and-configuration)
9. [Part 7: ELT Pipeline to BigQuery](#part-7-elt-pipeline-to-bigquery)
10. [Part 8: Scheduled Cloud Pipelines](#part-8-scheduled-cloud-pipelines)
11. [Part 9: AI Integration with RAG](#part-9-ai-integration-with-rag)
12. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before starting, ensure you have:

- **Docker** and **Docker Compose** installed
- **Git** for cloning the repository
- A **text editor** (VS Code recommended)
- **GCP account** (free tier works) for cloud sections
- Basic familiarity with YAML syntax

---

## Environment Setup

### Step 1: Navigate to the Module Directory

```bash
cd 02-workflow-orchestration/
```

### Step 2: Start the Docker Services

```bash
docker compose up -d
```

This starts four services:
| Service | Port | Purpose |
|---------|------|---------|
| Kestra | 8080 | Orchestration UI |
| pgdatabase | 5432 | NYC taxi data storage |
| pgAdmin | 8085 | Database management UI |
| kestra_postgres | - | Kestra backend storage |

### Step 3: Access Kestra UI

Open your browser and navigate to: **http://localhost:8080**

Login credentials:
- **Email**: `admin@kestra.io`
- **Password**: `Admin1234!`

### Step 4: Verify Services Are Running

```bash
docker compose ps
```

You should see all four containers with status `running`.

---

## Part 1: Hello World - Core Concepts

**File**: `flows/01_hello_world.yaml`

This flow demonstrates every core Kestra concept in one place.

### Understanding the Flow Structure

Open `flows/01_hello_world.yaml` and examine its sections:

```yaml
id: hello_world
namespace: zoomcamp
```

- **id**: Unique identifier for the flow
- **namespace**: Logical grouping (like a folder)

### Key Concepts Demonstrated

#### 1. Inputs - Dynamic Parameters

```yaml
inputs:
  - id: name
    type: STRING
    defaults: Will
```

Inputs allow you to pass different values each time you run the flow.

#### 2. Variables - Reusable Values

```yaml
variables:
  welcome_message: Hello {{inputs.name}}! Welcome to the Data Engineering course!
```

Variables use Pebble templating (`{{ }}`) to interpolate values.

#### 3. Tasks - The Work Units

```yaml
tasks:
  - id: hello
    type: io.kestra.plugin.core.log.Log
    message: "{{ vars.welcome_message }}"
```

Tasks are individual steps executed in sequence.

#### 4. Outputs - Passing Data Between Tasks

```yaml
  - id: return_data
    type: io.kestra.plugin.core.debug.Return
    format: "Cheers, {{ inputs.name }}!"
```

Outputs can be referenced by downstream tasks.

#### 5. Triggers - Automatic Execution

```yaml
triggers:
  - id: daily_at_10am
    type: io.kestra.plugin.core.trigger.Schedule
    disabled: true
    cron: "0 10 * * *"
```

Triggers start flows automatically (schedule, events, webhooks).

### Hands-On: Run the Flow

1. In Kestra UI, go to **Flows** > **Create**
2. Copy the contents of `01_hello_world.yaml`
3. Click **Save**
4. Click **Execute** (top right)
5. Try changing the `name` input to your name
6. Click **Execute** and observe the logs

### What to Observe

- The **Gantt** view shows task execution timeline
- The **Logs** tab shows output from each task
- The **Outputs** tab shows data produced by the flow

---

## Part 2: Python Integration

**File**: `flows/02_python.yaml`

Learn how to run Python code within Kestra workflows.

### Key Concepts

#### Docker Task Runner

```yaml
taskRunner:
  type: io.kestra.plugin.scripts.runner.docker.Docker
```

Kestra runs Python in isolated Docker containers, ensuring reproducibility.

#### Installing Dependencies

```yaml
beforeCommands:
  - pip install requests kestra
```

Dependencies are installed fresh in each container.

#### Returning Data to Kestra

```python
from kestra import Kestra

# ... your code ...

Kestra.outputs({"count": image_count})
```

The `kestra` Python package lets you send structured data back to the workflow.

### Hands-On: Create the Python Flow

1. Create a new flow with the contents of `02_python.yaml`
2. Execute the flow
3. Check the **Outputs** tab - you'll see the Docker Hub download count

### Exercise: Modify the Python Script

Try modifying the script to:
- Query a different API
- Return multiple output values
- Add error handling

---

## Part 3: Your First ETL Pipeline

**File**: `flows/03_getting_started_data_pipeline.yaml`

Build a complete Extract-Transform-Load pipeline.

### Pipeline Architecture

```
Extract (HTTP) → Transform (Python) → Query (DuckDB)
```

### Step-by-Step Breakdown

#### Extract: Fetch Data from API

```yaml
- id: extract
  type: io.kestra.plugin.core.http.Request
  uri: https://dummyjson.com/products
```

Downloads JSON data from a REST API.

#### Transform: Process with Python

```yaml
- id: transform
  type: io.kestra.plugin.scripts.python.Script
  script: |
    import json
    # Filter and transform the data
```

The Python task:
1. Reads the extracted JSON
2. Filters to keep only needed columns
3. Writes a cleaned CSV file

#### Load/Query: Analyze with DuckDB

```yaml
- id: query
  type: io.kestra.plugin.jdbc.duckdb.Query
  sql: |
    SELECT brand, avg(price) as avg_price
    FROM read_csv_auto('{{ outputs.transform.outputFiles["products.csv"] }}')
    GROUP BY brand
    ORDER BY avg_price DESC
```

DuckDB queries the transformed data directly from the CSV file.

### Hands-On: Build the Pipeline

1. Create the flow from `03_getting_started_data_pipeline.yaml`
2. Execute and observe each task
3. Check the **Outputs** tab for query results

### Understanding Data Flow

Notice how data passes between tasks:
- `{{ outputs.extract.body }}` - Body from HTTP request
- `{{ outputs.transform.outputFiles["products.csv"] }}` - File from Python task

---

## Part 4: Loading Data to PostgreSQL

**File**: `flows/04_postgres_taxi.yaml`

Load NYC taxi trip data into a local PostgreSQL database.

### Pipeline Overview

```
Input Selection → Download CSV → Create Table → Load Data → Transform → Merge
```

### Key Features

#### Dynamic Inputs

```yaml
inputs:
  - id: taxi
    type: SELECT
    values: [yellow, green]
  - id: year
    type: SELECT
    values: ["2019", "2020"]
  - id: month
    type: SELECT
    values: ["01", "02", ..., "12"]
```

Users select the taxi type, year, and month at runtime.

#### Conditional Logic (If/Switch)

```yaml
- id: if_yellow
  type: io.kestra.plugin.core.flow.If
  condition: "{{ inputs.taxi == 'yellow' }}"
  then:
    # Tasks for yellow taxi data
```

Different schemas require different processing logic.

#### Plugin Defaults

```yaml
pluginDefaults:
  - type: io.kestra.plugin.jdbc.postgresql
    values:
      url: jdbc:postgresql://pgdatabase:5432/ny_taxi
      username: root
      password: root
```

Apply default configuration to all PostgreSQL tasks.

#### Merge for Idempotency

```yaml
MERGE INTO {{ render(vars.table) }} AS T
USING {{ render(vars.staging_table) }} AS S
ON T.unique_key = S.unique_key
WHEN NOT MATCHED THEN INSERT ...
```

MERGE ensures running the same data twice doesn't create duplicates.

### Hands-On: Load Taxi Data

1. Create the flow from `04_postgres_taxi.yaml`
2. Execute with inputs:
   - taxi: `yellow`
   - year: `2019`
   - month: `01`
3. Wait for completion (this downloads ~100MB)

### Verify the Data

Access pgAdmin at **http://localhost:8085**:
- Email: `admin@admin.com`
- Password: `root`

Add a new server:
- Host: `pgdatabase`
- Port: `5432`
- Database: `ny_taxi`
- Username: `root`
- Password: `root`

Run this query:
```sql
SELECT COUNT(*) FROM yellow_tripdata;
```

---

## Part 5: Scheduling and Backfills

**File**: `flows/05_postgres_taxi_scheduled.yaml`

Automate pipeline execution with schedules and historical data loading.

### Schedule Configuration

```yaml
triggers:
  - id: monthly_trigger
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 9 1 * *"  # 9 AM on the 1st of each month
```

Cron expression format: `minute hour day month weekday`

### Dynamic Date from Trigger

```yaml
variables:
  year: "{{ trigger.date | date('yyyy') }}"
  month: "{{ trigger.date | date('MM') }}"
```

When triggered by schedule, `trigger.date` contains the execution timestamp.

### Backfilling Historical Data

Backfilling loads past data that wasn't processed when it was first available.

#### How to Backfill in Kestra

1. Go to the flow's **Triggers** tab
2. Click on your schedule trigger
3. Click **Backfill executions**
4. Select the date range (e.g., 2019-01-01 to 2020-12-31)
5. Click **Execute backfill**

Kestra will create one execution for each scheduled occurrence in that range.

### Concurrency Control

```yaml
concurrency:
  limit: 1
```

Ensures only one execution runs at a time, preventing database conflicts.

### Hands-On: Set Up Scheduled Pipeline

1. Create the flow from `05_postgres_taxi_scheduled.yaml`
2. Try a manual backfill for January 2019
3. Observe the execution in the **Executions** tab

---

## Part 6: GCP Setup and Configuration

**Files**: `flows/06_gcp_kv.yaml`, `flows/07_gcp_setup.yaml`

Prepare your Google Cloud Platform environment.

### Prerequisites

1. Create a GCP project (or use existing)
2. Enable BigQuery and Cloud Storage APIs
3. Create a service account with roles:
   - BigQuery Admin
   - Storage Admin
4. Download the service account JSON key

### Step 1: Store GCP Credentials

**File**: `flows/06_gcp_kv.yaml`

This flow stores your GCP configuration in Kestra's Key-Value store.

1. Open `06_gcp_kv.yaml`
2. Replace placeholder values:
   ```yaml
   - id: gcp_project_id
     value: YOUR_PROJECT_ID

   - id: gcp_location
     value: us-central1  # or your preferred region

   - id: gcp_bucket_name
     value: YOUR_UNIQUE_BUCKET_NAME

   - id: gcp_dataset
     value: zoomcamp
   ```
3. Create and execute the flow

### Step 2: Add Service Account Key

In Kestra UI:
1. Go to **Namespaces** > **zoomcamp**
2. Click **KV Store**
3. Add a new key:
   - Key: `GCP_CREDS`
   - Value: (paste your entire service account JSON)

### Step 3: Create GCP Resources

**File**: `flows/07_gcp_setup.yaml`

This flow creates the GCS bucket and BigQuery dataset.

```yaml
- id: create_gcs_bucket
  type: io.kestra.plugin.gcp.gcs.CreateBucket
  name: "{{ kv('GCP_BUCKET_NAME') }}"

- id: create_bq_dataset
  type: io.kestra.plugin.gcp.bigquery.CreateDataset
  name: "{{ kv('GCP_DATASET') }}"
```

1. Create and execute the flow
2. Verify in GCP Console that resources were created

---

## Part 7: ELT Pipeline to BigQuery

**File**: `flows/08_gcp_taxi.yaml`

Build a cloud-native ELT (Extract-Load-Transform) pipeline.

### ETL vs ELT

| ETL | ELT |
|-----|-----|
| Transform before loading | Load first, transform in warehouse |
| Limited by local compute | Leverage cloud compute power |
| Good for small data | Better for large data |

### Pipeline Architecture

```
Extract CSV → Upload to GCS → Create External Table → Transform in BigQuery → Merge
```

### Key Components

#### Upload to Cloud Storage

```yaml
- id: upload_to_gcs
  type: io.kestra.plugin.gcp.gcs.Upload
  from: "{{ outputs.extract.uri }}"
  to: "gs://{{ kv('GCP_BUCKET_NAME') }}/{{ render(vars.file) }}"
```

#### External Tables

```yaml
CREATE EXTERNAL TABLE IF NOT EXISTS `{{ kv('GCP_DATASET') }}.{{ render(vars.file) }}`
OPTIONS (
  format = 'CSV',
  uris = ['gs://{{ kv('GCP_BUCKET_NAME') }}/{{ render(vars.file) }}']
);
```

External tables query data directly from GCS without loading it into BigQuery storage.

#### BigQuery Merge

```yaml
MERGE INTO `{{ kv('GCP_DATASET') }}.{{ render(vars.table) }}` AS T
USING (SELECT *, MD5(...) as unique_key FROM staging) AS S
ON T.unique_key = S.unique_key
WHEN NOT MATCHED THEN INSERT ...
```

### Hands-On: Run the GCP Pipeline

1. Create the flow from `08_gcp_taxi.yaml`
2. Execute with:
   - taxi: `yellow`
   - year: `2019`
   - month: `01`
3. Monitor progress in Kestra UI
4. Verify data in BigQuery Console

### Verify in BigQuery

```sql
SELECT COUNT(*) as trip_count,
       AVG(total_amount) as avg_fare
FROM `your_project.zoomcamp.yellow_tripdata`
WHERE DATE(tpep_pickup_datetime) = '2019-01-15';
```

---

## Part 8: Scheduled Cloud Pipelines

**File**: `flows/09_gcp_taxi_scheduled.yaml`

Production-ready scheduled pipelines for continuous data ingestion.

### Schedule Configuration

```yaml
triggers:
  - id: green_schedule
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "0 9 * * *"  # Daily at 9 AM UTC
```

### Backfilling the Full Dataset

To load all historical data (2019-2020):

1. Go to the flow's **Triggers** tab
2. Click **Backfill executions**
3. Set range: `2019-01-01` to `2020-12-31`
4. Execute

This creates 24 executions (one per month for 2 years).

### Monitoring Backfills

- **Executions** tab shows all runs
- **Gantt** view shows task timing
- **Logs** contain detailed information
- Use **Labels** to filter backfill vs regular runs

### Production Considerations

1. **Concurrency**: Limit parallel executions to avoid API throttling
2. **Retries**: Configure automatic retry on failure
3. **Alerts**: Set up notifications for failed executions
4. **Labels**: Tag backfills for easy identification

---

## Part 9: AI Integration with RAG

**Files**: `flows/10_chat_without_rag.yaml`, `flows/11_chat_with_rag.yaml`

Explore AI integration and Retrieval Augmented Generation (RAG).

### The Problem: LLM Hallucinations

**File**: `flows/10_chat_without_rag.yaml`

Run this flow to see what happens when you ask an LLM about recent releases:

```yaml
- id: ai_response
  type: io.kestra.plugin.ai.ChatCompletion
  prompt: "Which features were released in Kestra 1.1?"
```

The LLM may hallucinate or give outdated information because:
- Training data has a cutoff date
- No access to current documentation

### The Solution: RAG (Retrieval Augmented Generation)

**File**: `flows/11_chat_with_rag.yaml`

RAG provides context to the LLM:

```
Ingest Docs → Create Embeddings → Store Vectors → Retrieve Context → Generate Response
```

#### Step 1: Ingest Documentation

```yaml
- id: ingest
  type: io.kestra.plugin.ai.IngestDocument
  source: "https://github.com/kestra-io/kestra/releases"
```

#### Step 2: Create and Store Embeddings

```yaml
- id: embed
  type: io.kestra.plugin.ai.Embed
  documents: "{{ outputs.ingest.documents }}"
```

Embeddings are vector representations that capture semantic meaning.

#### Step 3: Query with Context

```yaml
- id: query
  type: io.kestra.plugin.ai.RAGQuery
  prompt: "Which features were released in Kestra 1.1?"
  embeddingStore: "{{ outputs.embed.store }}"
```

The LLM now has access to actual release notes and provides accurate answers.

### Hands-On: Compare Results

1. Run `10_chat_without_rag.yaml` - note the response
2. Run `11_chat_with_rag.yaml` - compare the accuracy
3. Try different questions about Kestra features

---

## Troubleshooting

### Port Conflicts

If port 8080 is in use:

```yaml
# In docker-compose.yml, change:
ports:
  - "18080:8080"
```

Then access Kestra at http://localhost:18080

### Docker Socket Permission Denied

On Linux, you may need to add your user to the docker group:

```bash
sudo usermod -aG docker $USER
# Then log out and back in
```

### PostgreSQL Connection Failed

Ensure the database container is running:

```bash
docker compose logs pgdatabase
```

Check if the port is accessible:

```bash
docker compose exec pgdatabase pg_isready
```

### BigQuery CSV Column Mismatch

If you see schema errors:
1. Delete the existing file in GCS
2. Re-run the flow to re-upload

### Flow Import via API

To import flows programmatically:

```bash
curl -X POST "http://localhost:8080/api/v1/flows/import" \
  -u "admin@kestra.io:Admin1234" \
  -H "Content-Type: multipart/form-data" \
  -F "fileUpload=@flows/01_hello_world.yaml"
```

### Kestra Version Issues

Always pin to a specific version in production:

```yaml
image: kestra/kestra:v1.1  # Not :latest or :develop
```

### Cleaning Up

To stop all services and remove data:

```bash
docker compose down -v  # -v removes volumes
```

---

## Summary

You've learned how to:

- Set up Kestra with Docker Compose
- Understand core concepts: flows, tasks, inputs, outputs, triggers
- Run Python code within workflows
- Build ETL pipelines with PostgreSQL
- Schedule pipelines and run backfills
- Create cloud ELT pipelines with GCP
- Use AI and RAG in data workflows

### Next Steps

- Complete the [Module 2 Homework](https://github.com/DataTalksClub/data-engineering-zoomcamp/blob/main/cohorts/2025/02-workflow-orchestration/homework.md)
- Explore Kestra's [plugin library](https://kestra.io/plugins)
- Join the DataTalks.Club Slack for help

---

## Quick Reference

### Docker Commands

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker compose logs -f kestra

# Restart a service
docker compose restart kestra
```

### Kestra CLI (via Docker)

```bash
# Validate a flow
docker compose exec kestra kestra flow validate flows/

# List flows
docker compose exec kestra kestra flow list --namespace=zoomcamp
```

### URLs

| Service | URL |
|---------|-----|
| Kestra UI | http://localhost:8080 |
| pgAdmin | http://localhost:8085 |

### Credentials

| Service | Username | Password |
|---------|----------|----------|
| Kestra | admin@kestra.io | Admin1234 |
| pgAdmin | admin@admin.com | root |
| PostgreSQL | root | root |
