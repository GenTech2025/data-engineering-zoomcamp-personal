# Module 4: Analytics Engineering with dbt

## Table of Contents
- [Module Syllabus](#module-syllabus)
- [Theoretical Concepts](#theoretical-concepts)
- [Practical Tools Covered](#practical-tools-covered)
- [Step-by-Step Tutorial / Follow-Along Guide](#step-by-step-tutorial--follow-along-guide)
- [Documentation Links](#documentation-links)

---

## Module Syllabus

This module covers **Analytics Engineering** using **dbt (data build tool)** to transform raw data in a Data Warehouse into clean, tested, documented, analytics-ready models. By the end of this module, you will understand:

- What Analytics Engineering is and the role of an Analytics Engineer
- Data modeling concepts (Kimball dimensional modeling, Star Schema)
- What dbt is and how it fits in the modern data stack
- dbt Core (CLI) vs dbt Cloud (web IDE)
- Building a complete dbt project from scratch with layered architecture
- Sources, staging models, intermediate models, and marts
- Seeds (CSV reference data) and Macros (reusable SQL functions)
- Testing strategies and data documentation
- Packages and dependencies in dbt
- Incremental materializations for performance
- Cross-database compatibility (BigQuery and DuckDB)

**Dataset:** NYC Taxi Trip Records (Yellow and Green cabs, 2019-2020)

---

## Theoretical Concepts

### 1. What is Analytics Engineering?

Analytics Engineering sits at the intersection of **Data Engineering** and **Data Analytics**. An Analytics Engineer transforms raw data that has already been loaded into a warehouse into clean, reliable, well-documented datasets that analysts and business users can trust.

**Traditional Roles vs Analytics Engineering:**

| Aspect | Data Engineer | Analytics Engineer | Data Analyst |
|--------|--------------|-------------------|--------------|
| **Focus** | Infrastructure & pipelines | Data transformation & modeling | Insights & reporting |
| **Tools** | Python, Spark, Airflow | dbt, SQL, Jinja | Looker, Tableau, SQL |
| **Output** | Raw data in warehouse | Clean, tested models | Dashboards & reports |
| **Skills** | Software engineering | SQL + software engineering practices | Business domain + SQL |

**Key Responsibilities:**
- Define and maintain transformation logic (SQL models)
- Implement data quality tests
- Write documentation for datasets
- Build dimensional models (fact & dimension tables)
- Apply software engineering best practices (version control, CI/CD, code review) to data transformations

### 2. Data Modeling Concepts

#### Kimball Dimensional Modeling

The dbt project in this module follows **Ralph Kimball's dimensional modeling** approach, which organizes data into:

- **Fact Tables**: Store measurable events (transactions, trips, clicks). Each row represents a business event.
- **Dimension Tables**: Store descriptive context about the events (who, what, where, when).

**Star Schema:**

```
              ┌──────────────┐
              │  dim_vendors  │
              │──────────────│
              │ vendor_id    │
              │ vendor_name  │
              └──────┬───────┘
                     │
┌──────────────┐     │     ┌──────────────────────────┐
│  dim_zones   │     │     │        fct_trips          │
│──────────────│     │     │──────────────────────────│
│ location_id  ├─────┼─────┤ trip_id                  │
│ borough      │     │     │ vendor_id (FK)            │
│ zone         │     │     │ pickup_location_id (FK)   │
│ service_zone │     │     │ dropoff_location_id (FK)  │
└──────────────┘     │     │ pickup_datetime           │
                     │     │ trip_distance              │
                     │     │ total_amount               │
                     │     │ service_type               │
                     │     │ ...                        │
                     │     └──────────────────────────┘
```

#### Medallion / Layered Architecture

The project uses a **layered architecture** (also known as Medallion Architecture) to progressively transform data:

| Layer | dbt Folder | Purpose | Materialization |
|-------|-----------|---------|----------------|
| **Staging** (Bronze) | `models/staging/` | Rename columns, cast types, basic filters | `VIEW` |
| **Intermediate** (Silver) | `models/intermediate/` | Union datasets, enrich, deduplicate, apply business logic | `TABLE` |
| **Marts** (Gold) | `models/marts/` | Final analytics tables (facts & dimensions) | `TABLE` / `INCREMENTAL` |

**Why layers matter:**
- Each layer has a single responsibility
- Changes in raw data only affect staging, not downstream models
- Testing can happen at each layer
- Developers can debug issues layer by layer

### 3. What is dbt?

**dbt (data build tool)** is an open-source transformation tool that lets you write SQL SELECT statements and dbt handles turning them into tables/views in your data warehouse. It brings software engineering best practices to data transformation.

**How dbt works:**

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  SQL + Jinja │───>│   dbt       │───>│  Data       │
│  Models      │    │  (compile   │    │  Warehouse  │
│  (.sql files)│    │   & run)    │    │  (tables/   │
│              │    │             │    │   views)    │
└─────────────┘    └─────────────┘    └─────────────┘
```

1. You write SQL SELECT statements (models)
2. dbt compiles Jinja templates into pure SQL
3. dbt wraps your SELECT in CREATE TABLE/VIEW and executes it
4. dbt builds a dependency graph (DAG) and runs models in order

**Key Features:**
- **SQL-first**: Write transformations in SQL (the language analysts already know)
- **Jinja templating**: Add logic, loops, and variables to SQL
- **Dependency management**: Automatically determines build order via `ref()` and `source()`
- **Testing framework**: Validate data quality with schema and custom tests
- **Documentation**: Auto-generate a data dictionary website
- **Version control**: All models are code, stored in Git
- **Packages**: Extend functionality with community packages

### 4. dbt Core vs dbt Cloud

| Feature | dbt Core | dbt Cloud |
|---------|---------|-----------|
| **Interface** | Command-line (CLI) | Web-based IDE |
| **Cost** | Free & open-source | Free Developer plan available |
| **Hosting** | Your machine | Managed by dbt Labs |
| **Scheduling** | Need external scheduler (Airflow, cron) | Built-in job scheduler |
| **IDE** | Use any editor (VS Code, etc.) | Browser-based editor |
| **Collaboration** | Via Git | Built-in team features |
| **Best for** | Developers comfortable with CLI | Quick setup, team collaboration |

**This module supports both paths:**
- **Cloud path**: BigQuery + dbt Cloud
- **Local path**: DuckDB + dbt Core

### 5. Jinja Templating in dbt

dbt uses **Jinja** (a Python templating language) to add dynamic behavior to SQL:

```sql
-- Variables
{{ var('dev_start_date') }}

-- References to other models
{{ ref('stg_yellow_tripdata') }}

-- References to source tables
{{ source('raw', 'yellow_tripdata') }}

-- Conditional logic
{% if target.name == 'dev' %}
  WHERE pickup_datetime >= '{{ var("dev_start_date") }}'
{% endif %}

-- Macros (reusable functions)
{{ get_trip_duration_minutes('pickup_datetime', 'dropoff_datetime') }}

-- Incremental logic
{% if is_incremental() %}
  WHERE pickup_datetime > (SELECT MAX(pickup_datetime) FROM {{ this }})
{% endif %}
```

### 6. Materializations in dbt

Materializations determine **how** dbt persists your model in the warehouse:

| Materialization | Description | When to Use |
|----------------|-------------|-------------|
| **view** | Creates a SQL view (no data stored) | Staging models, lightweight transformations |
| **table** | Creates a full table (rebuilt each run) | Intermediate models, dimension tables |
| **incremental** | Appends/merges only new data | Large fact tables that grow over time |
| **ephemeral** | Inlined as CTE (not materialized) | Helper logic reused in other models |

### 7. Testing in dbt

dbt provides two categories of tests:

**Schema Tests** (defined in YAML):
- `unique`: No duplicate values in a column
- `not_null`: No NULL values in a column
- `accepted_values`: Column values must be in a defined list
- `relationships`: Foreign key references must exist in another table

**Custom Tests** (defined as SQL):
- SQL queries that return rows that **fail** the test
- If the query returns 0 rows, the test passes

### 8. Seeds, Sources, and Packages

- **Seeds**: Small CSV files (reference/lookup data) loaded into the warehouse with `dbt seed`. Example: payment type codes, taxi zone lookup tables.
- **Sources**: Declarations of raw tables that exist in the warehouse but are not managed by dbt. Defined in YAML, referenced via `{{ source() }}`. Support freshness checks.
- **Packages**: Reusable dbt code from the community. Installed via `packages.yml` and `dbt deps`. Example: `dbt_utils` provides utility macros like `generate_surrogate_key()`.

---

## Practical Tools Covered

### 1. dbt (data build tool)
- **What it does**: Transforms raw data in a warehouse into analytics-ready models using SQL and Jinja
- **Use case**: Building staging, intermediate, and marts layers; testing data quality; generating documentation
- **Variants used**: dbt Core (CLI) and/or dbt Cloud (web IDE)

### 2. Google BigQuery (Cloud Path)
- **What it does**: Fully-managed, serverless cloud data warehouse by Google
- **Use case**: Storing raw taxi trip data and running dbt transformations at scale
- **Note**: Requires GCP account (free tier available)

### 3. DuckDB (Local Path)
- **What it does**: In-process analytical SQL database (like SQLite but for analytics)
- **Use case**: Running the entire dbt project locally without cloud costs
- **Key advantage**: Zero setup, runs on your laptop, perfect for learning

### 4. Jinja
- **What it does**: Python-based templating language embedded in dbt SQL files
- **Use case**: Adding variables, conditional logic, loops, and macros to SQL models

### 5. dbt Packages (dbt_utils, codegen)
- **dbt_utils**: Provides utility macros like `generate_surrogate_key()`, `unique_combination_of_columns` test
- **codegen**: Helps auto-generate base model SQL and schema YAML files

### 6. Git / Version Control
- **What it does**: Tracks changes to dbt project files
- **Use case**: Versioning all SQL models, collaborating with team, code review for data transformations

### 7. VS Code + dbt Power User Extension (Optional)
- **What it does**: Enhanced dbt development experience in VS Code
- **Use case**: Syntax highlighting, model navigation, inline compilation, lineage visualization

---

## Step-by-Step Tutorial / Follow-Along Guide

This tutorial walks you through the complete Module 4 project. Choose **either** the Cloud path (BigQuery + dbt Cloud) or the Local path (DuckDB + dbt Core).

---

### Part 1: Environment Setup

#### Option A: Local Setup (DuckDB + dbt Core) - Recommended for Beginners

**Step 1: Install DuckDB CLI**

```bash
# macOS (Homebrew)
brew install duckdb

# Ubuntu/Debian
wget https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-linux-amd64.zip
unzip duckdb_cli-linux-amd64.zip
sudo mv duckdb /usr/local/bin/

# Verify installation
duckdb --version
```

**Step 2: Install dbt-duckdb**

```bash
# Create a virtual environment (recommended)
python -m venv dbt-env
source dbt-env/bin/activate  # Linux/macOS
# dbt-env\Scripts\activate   # Windows

# Install dbt with DuckDB adapter
pip install dbt-duckdb
```

**Step 3: Configure dbt Profile**

Create or edit `~/.dbt/profiles.yml`:

```yaml
taxi_rides_ny:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: "taxi_rides_ny.duckdb"
      schema: dev
      threads: 4
    prod:
      type: duckdb
      path: "taxi_rides_ny.duckdb"
      schema: prod
      threads: 4
```

> **Note:** The `path` should point to where you want the DuckDB database file to live. If you run dbt from the `taxi_rides_ny/` project directory, this creates the file there.

**Step 4: Load Source Data**

Navigate to the setup directory and run the ingestion script:

```bash
cd 04-analytics-engineering/setup/
python web_to_local_duckdb.py
```

This script:
- Downloads yellow and green taxi CSV.gz files (2019-2020) from GitHub releases
- Converts them to Parquet format
- Creates a `prod` schema in the DuckDB database
- Loads data into `prod.yellow_tripdata` and `prod.green_tripdata` tables

**Step 5: Verify Connection**

```bash
cd ../taxi_rides_ny/
dbt debug
```

Expected output:
```
  Connection test: [OK connection ok]
  All checks passed!
```

#### Option B: Cloud Setup (BigQuery + dbt Cloud)

**Step 1: Verify BigQuery Data**

Ensure you have completed Module 3 and have the NYC taxi data loaded in BigQuery under a `nytaxi` dataset.

**Step 2: Create a dbt Cloud Account**

1. Go to [cloud.getdbt.com](https://cloud.getdbt.com) and sign up (free Developer plan)
2. Create a new project

**Step 3: Configure BigQuery Connection**

1. In dbt Cloud, go to **Account Settings > Projects > Connection**
2. Select **BigQuery** as the connection type
3. Upload your GCP service account JSON key
4. Set the dataset (schema) to your development dataset (e.g., `dbt_yourname`)

**Step 4: Set Up Repository**

1. In dbt Cloud, connect to a Git repository (GitHub, GitLab, or managed repo)
2. Initialize the project or clone the existing `taxi_rides_ny` dbt project

**Step 5: Test Connection**

In the dbt Cloud IDE, open the terminal and run:

```bash
dbt debug
```

---

### Part 2: Understanding the dbt Project Structure

Navigate to the project directory:

```bash
cd 04-analytics-engineering/taxi_rides_ny/
```

The project is organized as follows:

```
taxi_rides_ny/
├── dbt_project.yml          # Project configuration (name, version, materializations)
├── packages.yml             # External package dependencies
├── models/
│   ├── staging/             # Layer 1: Standardize raw data
│   │   ├── sources.yml      # Define raw source tables
│   │   ├── schema.yml       # Tests & docs for staging models
│   │   ├── stg_green_tripdata.sql
│   │   └── stg_yellow_tripdata.sql
│   ├── intermediate/        # Layer 2: Business logic
│   │   ├── schema.yml       # Tests & docs for intermediate models
│   │   ├── int_trips_unioned.sql
│   │   └── int_trips.sql
│   └── marts/               # Layer 3: Final analytics tables
│       ├── schema.yml       # Tests & docs for marts models
│       ├── fct_trips.sql
│       ├── dim_zones.sql
│       ├── dim_vendors.sql
│       └── reporting/
│           ├── schema.yml
│           └── fct_monthly_zone_revenue.sql
├── seeds/                   # CSV reference data
│   ├── seeds_properties.yml
│   ├── payment_type_lookup.csv
│   └── taxi_zone_lookup.csv
├── macros/                  # Reusable SQL functions
│   ├── macros_properties.yml
│   ├── get_trip_duration_minutes.sql
│   └── get_vendor_data.sql
├── tests/                   # Custom SQL tests
├── snapshots/               # Slowly changing dimensions
└── analyses/                # Ad-hoc queries
```

#### Key Configuration: `dbt_project.yml`

```yaml
name: 'taxi_rides_ny'
version: '1.0.0'
require-dbt-version: '>=1.7.0,<2.0.0'
profile: 'taxi_rides_ny'

model-paths: ["models"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
test-paths: ["tests"]
snapshot-paths: ["snapshots"]
analysis-paths: ["analyses"]

models:
  taxi_rides_ny:
    staging:
      +materialized: view       # Staging = views (lightweight)
    intermediate:
      +materialized: table      # Intermediate = tables
    marts:
      +materialized: table      # Marts = tables

vars:
  dev_start_date: '2019-01-01'  # Dev environment: only load Jan 2019
  dev_end_date: '2019-02-01'
```

**Key points:**
- `profile` links to the connection defined in `~/.dbt/profiles.yml`
- `+materialized` sets the default materialization per folder
- `vars` define project-level variables accessible via `{{ var('dev_start_date') }}`

---

### Part 3: Install Packages

Before building models, install dbt packages:

```bash
dbt deps
```

This reads `packages.yml` and installs:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: 1.3.3
  - package: dbt-labs/codegen
    version: 0.14.0
```

- **dbt_utils**: Provides `generate_surrogate_key()` macro and `unique_combination_of_columns` test
- **codegen**: Helps generate boilerplate schema YAML and base models

Packages are installed into the `dbt_packages/` directory (gitignored).

---

### Part 4: Load Seeds

Seeds are CSV files that dbt loads as tables in your warehouse. Load them first since models depend on them:

```bash
dbt seed
```

This loads two reference tables:

**`payment_type_lookup.csv`** - Maps payment codes to descriptions:

| payment_type | description |
|-------------|-------------|
| 0 | Unknown |
| 1 | Credit card |
| 2 | Cash |
| 3 | No charge |
| 4 | Dispute |
| 5 | Unknown |
| 6 | Voided trip |

**`taxi_zone_lookup.csv`** - Maps location IDs to NYC zones (265 zones):

| locationid | borough | zone | service_zone |
|-----------|---------|------|-------------|
| 1 | EWR | Newark Airport | EWR |
| 132 | Queens | JFK Airport | Airports |
| 138 | Queens | LaGuardia Airport | Airports |
| 230 | Manhattan | Times Sq/Theatre District | Yellow Zone |
| 261 | Manhattan | World Trade Center | Yellow Zone |

---

### Part 5: Understanding Sources

Sources declare the raw tables that exist in your warehouse. They are defined in `models/staging/sources.yml`:

```yaml
sources:
  - name: raw
    database: |
      {%- if target.type == 'bigquery' -%}
        {{ env_var('GCP_PROJECT_ID', 'your-project-id') }}
      {%- else -%}
        taxi_rides_ny
      {%- endif -%}
    schema: |
      {%- if target.type == 'bigquery' -%}
        nytaxi
      {%- else -%}
        prod
      {%- endif -%}
    loaded_at_field: |
      {%- if target.type == 'bigquery' -%}
        tpep_pickup_datetime
      {%- else -%}
        lpep_pickup_datetime
      {%- endif -%}
    freshness:
      warn_after: { count: 24, period: hour }
      error_after: { count: 48, period: hour }
    tables:
      - name: green_tripdata
        description: "Raw green taxi trip records"
      - name: yellow_tripdata
        description: "Raw yellow taxi trip records"
```

**Key points:**
- The Jinja `if/else` blocks make this work on both BigQuery and DuckDB
- `freshness` checks warn if data hasn't been updated in 24 hours
- In models, reference sources with `{{ source('raw', 'green_tripdata') }}`

---

### Part 6: Building Staging Models (Layer 1)

Staging models standardize raw data: rename columns, cast types, and apply basic filters. They are materialized as **views** for efficiency.

#### `stg_green_tripdata.sql`

```sql
{{ config(materialized='view') }}

with source as (
    select * from {{ source('raw', 'green_tripdata') }}
),

renamed as (
    select
        -- identifiers
        cast(vendorid as integer) as vendor_id,
        cast(ratecodeid as integer) as rate_code_id,
        cast(pulocationid as integer) as pickup_location_id,
        cast(dolocationid as integer) as dropoff_location_id,

        -- timestamps
        cast(lpep_pickup_datetime as timestamp) as pickup_datetime,
        cast(lpep_dropoff_datetime as timestamp) as dropoff_datetime,

        -- trip info
        store_and_fwd_flag,
        cast(passenger_count as integer) as passenger_count,
        cast(trip_distance as numeric) as trip_distance,
        cast(trip_type as integer) as trip_type,

        -- payment info
        cast(fare_amount as numeric) as fare_amount,
        cast(extra as numeric) as extra,
        cast(mta_tax as numeric) as mta_tax,
        cast(tip_amount as numeric) as tip_amount,
        cast(tolls_amount as numeric) as tolls_amount,
        cast(ehail_fee as numeric) as ehail_fee,
        cast(improvement_surcharge as numeric) as improvement_surcharge,
        cast(total_amount as numeric) as total_amount,
        cast(payment_type as integer) as payment_type,
        cast(congestion_surcharge as numeric) as congestion_surcharge

    from source
    where vendorid is not null

    {% if target.name == 'dev' %}
      and lpep_pickup_datetime >= '{{ var("dev_start_date") }}'
      and lpep_pickup_datetime < '{{ var("dev_end_date") }}'
    {% endif %}
)

select * from renamed
```

**What this model does:**
1. **Reads raw data** from the `green_tripdata` source
2. **Renames columns** for consistency (e.g., `vendorid` -> `vendor_id`, `pulocationid` -> `pickup_location_id`)
3. **Casts types** to ensure correct data types (integer, timestamp, numeric)
4. **Filters out nulls** where `vendorid` is null
5. **Dev sampling**: In the dev environment, only processes January 2019 data for faster builds

#### `stg_yellow_tripdata.sql`

Nearly identical to green, but with these differences:
- Uses `tpep_pickup_datetime` / `tpep_dropoff_datetime` (TPEP = Taxicab Passenger Enhancement Program) instead of `lpep_*` (LPEP = Licensed Passenger Enhancement Program)
- Does **not** include `trip_type` or `ehail_fee` columns (yellow taxis don't have these)

**Run staging models:**

```bash
dbt run --select staging
```

---

### Part 7: Building Intermediate Models (Layer 2)

Intermediate models apply business logic: union datasets, enrich with reference data, and deduplicate. Materialized as **tables**.

#### `int_trips_unioned.sql` - Combine Yellow and Green Trips

```sql
with green_trips as (
    select
        vendor_id,
        pickup_datetime,
        dropoff_datetime,
        store_and_fwd_flag,
        rate_code_id,
        pickup_location_id,
        dropoff_location_id,
        passenger_count,
        trip_distance,
        fare_amount,
        extra,
        mta_tax,
        tip_amount,
        tolls_amount,
        ehail_fee,
        improvement_surcharge,
        total_amount,
        payment_type,
        congestion_surcharge,
        trip_type,
        'Green' as service_type
    from {{ ref('stg_green_tripdata') }}
),

yellow_trips as (
    select
        vendor_id,
        pickup_datetime,
        dropoff_datetime,
        store_and_fwd_flag,
        rate_code_id,
        pickup_location_id,
        dropoff_location_id,
        passenger_count,
        trip_distance,
        fare_amount,
        extra,
        mta_tax,
        tip_amount,
        tolls_amount,
        cast(0 as numeric) as ehail_fee,       -- Yellow taxis have no ehail
        improvement_surcharge,
        total_amount,
        payment_type,
        congestion_surcharge,
        1 as trip_type,                          -- Yellow = street-hail only
        'Yellow' as service_type
    from {{ ref('stg_yellow_tripdata') }}
),

all_trips as (
    select * from green_trips
    union all
    select * from yellow_trips
)

select * from all_trips
```

**What this model does:**
1. Selects green trips and adds `'Green' as service_type`
2. Selects yellow trips, fills missing columns (`ehail_fee = 0`, `trip_type = 1`), and adds `'Yellow' as service_type`
3. Unions both into a single dataset with `UNION ALL`

#### `int_trips.sql` - Enrich and Deduplicate

```sql
with unioned as (
    select * from {{ ref('int_trips_unioned') }}
),

payment as (
    select * from {{ ref('payment_type_lookup') }}
),

trips_with_payment as (
    select
        {{ dbt_utils.generate_surrogate_key([
            'u.vendor_id',
            'u.pickup_datetime',
            'u.pickup_location_id',
            'u.service_type'
        ]) }} as trip_id,

        u.vendor_id,
        u.service_type,
        u.rate_code_id,
        u.pickup_location_id,
        u.dropoff_location_id,
        u.pickup_datetime,
        u.dropoff_datetime,
        u.store_and_fwd_flag,
        u.passenger_count,
        u.trip_distance,
        u.trip_type,
        u.fare_amount,
        u.extra,
        u.mta_tax,
        u.tip_amount,
        u.tolls_amount,
        u.ehail_fee,
        u.improvement_surcharge,
        u.total_amount,
        coalesce(u.payment_type, 0) as payment_type,
        p.description as payment_type_description,
        u.congestion_surcharge

    from unioned u
    left join payment p
        on coalesce(u.payment_type, 0) = p.payment_type

    qualify row_number() over(
        partition by u.vendor_id, u.pickup_datetime, u.pickup_location_id, u.service_type
        order by u.dropoff_datetime
    ) = 1
)

select * from trips_with_payment
```

**What this model does:**
1. **Generates a surrogate key** (`trip_id`) using `dbt_utils.generate_surrogate_key()` - creates an MD5 hash from natural key columns
2. **Joins payment descriptions** from the `payment_type_lookup` seed using `COALESCE` to handle nulls
3. **Deduplicates** using `QUALIFY ROW_NUMBER()` - if multiple trips share the same natural key, only the first (by dropoff time) is kept

**Run intermediate models:**

```bash
dbt run --select intermediate
```

---

### Part 8: Building Macros (Reusable SQL Functions)

Macros are Jinja templates that generate SQL at compile time. They enable code reuse and cross-database compatibility.

#### `get_trip_duration_minutes.sql`

```sql
{% macro get_trip_duration_minutes(pickup_datetime, dropoff_datetime) %}
    {{ dbt.datediff(pickup_datetime, dropoff_datetime, 'minute') }}
{% endmacro %}
```

**What it does:** Calculates trip duration in minutes. Uses dbt's built-in `datediff()` which works on BigQuery, DuckDB, Snowflake, PostgreSQL, etc.

**Usage in a model:**
```sql
{{ get_trip_duration_minutes('trips.pickup_datetime', 'trips.dropoff_datetime') }} as trip_duration_minutes
```

#### `get_vendor_data.sql`

```sql
{% macro get_vendor_data(vendor_id_column) %}
{% set vendors = {
    1: 'Creative Mobile Technologies',
    2: 'VeriFone Inc.',
    4: 'Unknown/Other'
} %}

case {{ vendor_id_column }}
    {% for vendor_id, vendor_name in vendors.items() %}
    when {{ vendor_id }} then '{{ vendor_name }}'
    {% endfor %}
end
{% endmacro %}
```

**What it does:** Generates a SQL `CASE` statement mapping vendor IDs to company names at compile time.

**At compile time, this generates:**
```sql
case vendor_id
    when 1 then 'Creative Mobile Technologies'
    when 2 then 'VeriFone Inc.'
    when 4 then 'Unknown/Other'
end
```

---

### Part 9: Building Marts Models (Layer 3)

Marts are the final, business-ready tables. These are what analysts and dashboards consume.

#### `dim_zones.sql` - Zone Dimension Table

```sql
select
    locationid as location_id,
    borough,
    zone,
    service_zone
from {{ ref('taxi_zone_lookup') }}
```

A simple pass-through from the seed, but having it as a model:
- Provides a consistent interface for fact tables
- Allows future enhancements without breaking downstream models

#### `dim_vendors.sql` - Vendor Dimension Table

```sql
with trips as (
    select * from {{ ref('fct_trips') }}
),

vendors as (
    select distinct
        vendor_id,
        {{ get_vendor_data('vendor_id') }} as vendor_name
    from trips
)

select * from vendors
```

Extracts distinct vendors from the fact table and enriches with names using the custom macro.

#### `fct_trips.sql` - Core Fact Table (Incremental)

```sql
{{
  config(
    materialized='incremental',
    unique_key='trip_id',
    on_schema_change='fail'
  )
}}

with trips as (
    select * from {{ ref('int_trips') }}
),

dim_zones as (
    select * from {{ ref('dim_zones') }}
)

select
    trips.trip_id,
    trips.vendor_id,
    trips.service_type,
    trips.rate_code_id,
    trips.pickup_location_id,
    pz.borough as pickup_borough,
    pz.zone as pickup_zone,
    trips.dropoff_location_id,
    dz.borough as dropoff_borough,
    dz.zone as dropoff_zone,
    trips.pickup_datetime,
    trips.dropoff_datetime,
    trips.store_and_fwd_flag,
    trips.passenger_count,
    trips.trip_distance,
    trips.trip_type,
    {{ get_trip_duration_minutes('trips.pickup_datetime', 'trips.dropoff_datetime') }} as trip_duration_minutes,
    trips.fare_amount,
    trips.extra,
    trips.mta_tax,
    trips.tip_amount,
    trips.tolls_amount,
    trips.ehail_fee,
    trips.improvement_surcharge,
    trips.total_amount,
    trips.payment_type,
    trips.payment_type_description,
    trips.congestion_surcharge

from trips
left join dim_zones as pz
    on trips.pickup_location_id = pz.location_id
left join dim_zones as dz
    on trips.dropoff_location_id = dz.location_id

{% if is_incremental() %}
  where trips.pickup_datetime > (select max(pickup_datetime) from {{ this }})
{% endif %}
```

**What this model does:**
1. **Joins zone data** to add pickup/dropoff borough and zone names via LEFT JOINs
2. **Calculates trip duration** using the custom macro
3. **Incremental logic**: On the first run, loads all data. On subsequent runs, only loads trips newer than the latest `pickup_datetime` already in the table
4. **`on_schema_change='fail'`**: If source schema changes, dbt fails rather than silently dropping/adding columns

#### `fct_monthly_zone_revenue.sql` - Reporting Aggregation

```sql
select
    coalesce(pickup_zone, 'Unknown Zone') as pickup_zone,
    {{ dbt.date_trunc('month', 'pickup_datetime') }} as revenue_month,
    service_type,

    -- Revenue metrics
    sum(fare_amount) as revenue_monthly_fare,
    sum(extra) as revenue_monthly_extra,
    sum(mta_tax) as revenue_monthly_mta_tax,
    sum(tip_amount) as revenue_monthly_tip_amount,
    sum(tolls_amount) as revenue_monthly_tolls_amount,
    sum(ehail_fee) as revenue_monthly_ehail_fee,
    sum(improvement_surcharge) as revenue_monthly_improvement_surcharge,
    sum(total_amount) as revenue_monthly_total_amount,

    -- Operational metrics
    count(trip_id) as total_monthly_trips,
    avg(passenger_count) as avg_monthly_passenger_count,
    avg(trip_distance) as avg_monthly_trip_distance

from {{ ref('fct_trips') }}
group by pickup_zone, revenue_month, service_type
```

**What this model does:** Aggregates trip data by pickup zone, month, and service type for business dashboards - providing monthly revenue breakdowns and operational metrics.

**Run all marts models:**

```bash
dbt run --select marts
```

---

### Part 10: Run the Full Project

Now that you understand each layer, build the entire project:

```bash
# Build everything: seeds + models + tests (in dependency order)
dbt build
```

Or run each step separately:

```bash
# Step 1: Load seed data
dbt seed

# Step 2: Run all models (staging → intermediate → marts)
dbt run

# Step 3: Run all tests
dbt test
```

**Useful `dbt run` selection syntax:**

```bash
# Run a specific model
dbt run --select fct_trips

# Run a model and all its upstream dependencies (+ prefix)
dbt run --select +fct_trips

# Run a model and all its downstream dependents (+ suffix)
dbt run --select fct_trips+

# Run all models in a folder
dbt run --select staging

# Full refresh of incremental models (rebuild from scratch)
dbt run --full-refresh --select fct_trips
```

---

### Part 11: Testing Your Models

Run all tests to validate data quality:

```bash
dbt test
```

The project includes tests at every layer:

**Staging tests:**
- `vendor_id` is not null
- `pickup_datetime` is not null

**Intermediate tests (`int_trips`):**
- `trip_id` is unique and not null
- `service_type` is in `['Green', 'Yellow']`
- `total_amount` is not null

**Marts tests (`fct_trips`):**
- `trip_id` is unique and not null
- `service_type` accepted values: `['Green', 'Yellow']`
- `payment_type` accepted values: `[0, 1, 2, 3, 4, 5, 6]`
- `pickup_location_id` has a valid relationship to `dim_zones.location_id`
- `dropoff_location_id` has a valid relationship to `dim_zones.location_id`

**Reporting tests (`fct_monthly_zone_revenue`):**
- `dbt_utils.unique_combination_of_columns` for `(pickup_zone, revenue_month, service_type)`
- `pickup_zone`, `revenue_month`, `service_type` are not null

**Run tests for a specific model:**

```bash
dbt test --select fct_trips
```

---

### Part 12: Generate and Serve Documentation

dbt can auto-generate a documentation website from your schema YAML files:

```bash
# Generate documentation
dbt docs generate

# Serve locally (opens browser at localhost:8080)
dbt docs serve
```

The documentation site includes:
- **Model descriptions** and column-level documentation
- **Lineage graph (DAG)**: Visual representation of how models depend on each other
- **Test results**: Which tests are defined on each model
- **Source freshness**: Status of raw data freshness checks

---

### Part 13: Dev vs Prod Environments

The project uses dbt variables to limit data in dev for faster iteration:

**In dev** (`target.name == 'dev'`):
- Staging models filter to January 2019 only
- Fast builds (~seconds instead of minutes)
- Uses `dev` schema

**In prod** (`target.name == 'prod'`):
- Processes all 2019-2020 data
- Full dataset builds
- Uses `prod` schema

**Switch targets:**

```bash
# Run with dev target (default)
dbt run

# Run with prod target
dbt run --target prod
```

---

### Part 14: Model Dependency Graph (DAG)

The complete model flow:

```
Sources (Raw Data)
    │
    ├── raw.green_tripdata
    │       │
    │       ▼
    │   stg_green_tripdata (view)
    │       │
    ├── raw.yellow_tripdata
    │       │
    │       ▼
    │   stg_yellow_tripdata (view)
    │       │
    │       └───────┬───────┘
    │               │
    │               ▼
    │       int_trips_unioned (table)
    │               │
    │               │  ◄── payment_type_lookup (seed)
    │               │
    │               ▼
    │          int_trips (table)
    │               │
    │               │  ◄── dim_zones (table) ◄── taxi_zone_lookup (seed)
    │               │
    │               ▼
    │          fct_trips (incremental)
    │           │          │
    │           │          ▼
    │           │    dim_vendors (table)
    │           │
    │           ▼
    │   fct_monthly_zone_revenue (table)
```

---

### SQL Refresher: Key Concepts Used in This Module

The module includes a SQL refresher covering advanced concepts you will encounter:

#### Window Functions

```sql
-- ROW_NUMBER: Assign unique row numbers within partitions (used for deduplication)
ROW_NUMBER() OVER(
    PARTITION BY vendor_id, pickup_datetime
    ORDER BY dropoff_datetime
) as row_num

-- RANK: Like ROW_NUMBER but assigns same rank for ties
RANK() OVER(
    PARTITION BY pickup_location_id
    ORDER BY total_amount DESC
) as amount_rank

-- LAG/LEAD: Access previous/next row values
LAG(total_amount) OVER(ORDER BY pickup_datetime) as prev_amount
LEAD(total_amount) OVER(ORDER BY pickup_datetime) as next_amount

-- PERCENTILE_CONT: Calculate percentiles
PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY trip_distance) as p90_distance
```

#### Common Table Expressions (CTEs)

```sql
-- CTEs make complex queries readable and modular
WITH source AS (
    SELECT * FROM raw_table
),

cleaned AS (
    SELECT
        id,
        UPPER(name) as name
    FROM source
    WHERE id IS NOT NULL
),

final AS (
    SELECT * FROM cleaned
    WHERE name != 'UNKNOWN'
)

SELECT * FROM final
```

#### QUALIFY Clause (Used for Deduplication)

```sql
-- QUALIFY filters window function results (like HAVING for aggregates)
SELECT *
FROM trips
QUALIFY ROW_NUMBER() OVER(
    PARTITION BY vendor_id, pickup_datetime
    ORDER BY dropoff_datetime
) = 1
-- Keeps only the first row per partition
```

---

### Essential dbt Commands Reference

```bash
# ── Setup ──────────────────────────────────────────
dbt deps              # Install packages from packages.yml
dbt debug             # Test warehouse connection
dbt clean             # Delete target/ and dbt_packages/ directories

# ── Building ───────────────────────────────────────
dbt seed              # Load CSV seeds into warehouse
dbt run               # Run all models
dbt test              # Run all tests
dbt build             # Seed + run + test (in dependency order)

# ── Selection ──────────────────────────────────────
dbt run --select <model_name>       # Run one model
dbt run --select +<model_name>      # Run model + upstream deps
dbt run --select <model_name>+      # Run model + downstream deps
dbt run --select staging            # Run all models in folder
dbt run --full-refresh              # Rebuild incremental models

# ── Documentation ──────────────────────────────────
dbt docs generate     # Generate docs site
dbt docs serve        # Serve docs at localhost:8080

# ── Debugging ──────────────────────────────────────
dbt compile           # Compile Jinja to SQL (without running)
dbt compile --select fct_trips   # Compile specific model
dbt source freshness  # Check source data freshness

# ── Targets ────────────────────────────────────────
dbt run --target dev   # Run against dev environment (default)
dbt run --target prod  # Run against prod environment
```

---

## Documentation Links

### dbt (data build tool)
- [dbt Documentation (Official)](https://docs.getdbt.com/)
- [dbt Fundamentals Course (Free)](https://learn.getdbt.com/)
- [dbt Core GitHub Repository](https://github.com/dbt-labs/dbt-core)
- [dbt Cloud](https://cloud.getdbt.com/)
- [dbt Jinja Documentation](https://docs.getdbt.com/docs/build/jinja-macros)
- [dbt Materializations](https://docs.getdbt.com/docs/build/materializations)
- [dbt Tests](https://docs.getdbt.com/docs/build/data-tests)
- [dbt Seeds](https://docs.getdbt.com/docs/build/seeds)
- [dbt Sources](https://docs.getdbt.com/docs/build/sources)
- [dbt Packages (Package Hub)](https://hub.getdbt.com/)
- [dbt_utils Package](https://hub.getdbt.com/dbt-labs/dbt_utils/latest/)

### DuckDB
- [DuckDB Documentation](https://duckdb.org/docs/)
- [DuckDB Installation](https://duckdb.org/docs/installation/)
- [dbt-duckdb Adapter](https://github.com/duckdb/dbt-duckdb)

### Google BigQuery
- [BigQuery Documentation](https://cloud.google.com/bigquery/docs)
- [BigQuery SQL Reference](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax)
- [BigQuery Free Tier](https://cloud.google.com/bigquery/pricing#free-tier)

### Jinja
- [Jinja Documentation](https://jinja.palletsprojects.com/)
- [dbt Jinja Context](https://docs.getdbt.com/reference/dbt-jinja-functions)

### Data Modeling
- [Kimball Group - Dimensional Modeling](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/dimensional-modeling-techniques/)
- [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
- [NYC TLC Data Dictionary (PDF)](https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf)

### Data Engineering Zoomcamp
- [Module 4 Course Materials](https://github.com/DataTalksClub/data-engineering-zoomcamp/tree/main/04-analytics-engineering)
- [Data Engineering Zoomcamp Main Repository](https://github.com/DataTalksClub/data-engineering-zoomcamp)