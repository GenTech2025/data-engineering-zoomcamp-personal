# Data Engineering Zoomcamp - All Homework Questions (2026 Cohort)

This document consolidates all homework questions from all modules of the Data Engineering Zoomcamp.

---

## Module 1: Docker & SQL

**Topics:** Docker, PostgreSQL, Terraform, GCP

### Question 1: Understanding Docker images

Run docker with the `python:3.13` image. Use an entrypoint `bash` to interact with the container.

What's the version of `pip` in the image?

- 25.3
- 24.3.1
- 24.2.1
- 23.3.1

### Question 2: Understanding Docker networking and docker-compose

Given the following `docker-compose.yaml`, what is the `hostname` and `port` that pgadmin should use to connect to the postgres database?

```yaml
services:
  db:
    container_name: postgres
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'postgres'
      POSTGRES_DB: 'ny_taxi'
    ports:
      - '5433:5432'
    volumes:
      - vol-pgdata:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: "pgadmin@pgadmin.com"
      PGADMIN_DEFAULT_PASSWORD: "pgadmin"
    ports:
      - "8080:80"
    volumes:
      - vol-pgadmin_data:/var/lib/pgadmin

volumes:
  vol-pgdata:
    name: vol-pgdata
  vol-pgadmin_data:
    name: vol-pgadmin_data
```

- postgres:5433
- localhost:5432
- db:5433
- postgres:5432
- db:5432

### Data Preparation

Download the green taxi trips data for November 2025:

```bash
wget https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet
```

You will also need the dataset with zones:

```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv
```

### Question 3: Counting short trips

For the trips in November 2025 (lpep_pickup_datetime between '2025-11-01' and '2025-12-01', exclusive of the upper bound), how many trips had a `trip_distance` of less than or equal to 1 mile?

- 7,853
- 8,007
- 8,254
- 8,421

### Question 4: Longest trip for each day

Which was the pick up day with the longest trip distance? Only consider trips with `trip_distance` less than 100 miles (to exclude data errors).

Use the pick up time for your calculations.

- 2025-11-14
- 2025-11-20
- 2025-11-23
- 2025-11-25

### Question 5: Biggest pickup zone

Which was the pickup zone with the largest `total_amount` (sum of all trips) on November 18th, 2025?

- East Harlem North
- East Harlem South
- Morningside Heights
- Forest Hills

### Question 6: Largest tip

For the passengers picked up in the zone named "East Harlem North" in November 2025, which was the drop off zone that had the largest tip?

Note: it's `tip`, not `trip`. We need the name of the zone, not the ID.

- JFK Airport
- Yorkville West
- East Harlem North
- LaGuardia Airport

### Question 7: Terraform Workflow

Which of the following sequences, respectively, describes the workflow for:
1. Downloading the provider plugins and setting up backend,
2. Generating proposed changes and auto-executing the plan
3. Remove all resources managed by terraform

Answers:
- terraform import, terraform apply -y, terraform destroy
- teraform init, terraform plan -auto-apply, terraform rm
- terraform init, terraform run -auto-approve, terraform destroy
- terraform init, terraform apply -auto-approve, terraform destroy
- terraform import, terraform apply -y, terraform rm

---

## Module 2: Workflow Orchestration (Kestra)

**Topics:** Kestra, YAML workflows, ETL pipelines, Backfill

### Assignment

Extend the existing flows to include data for the year 2021.

Options:
1. Use the backfill functionality in the scheduled flow to backfill data from `2021-01-01` to `2021-07-31` for both `yellow` and `green` taxi data.
2. Run the flow manually for each of the seven months of 2021 for both taxi types, or use `ForEach` task with `Subflow` task.

### Question 1

Within the execution for `Yellow` Taxi data for the year `2020` and month `12`: what is the uncompressed file size (i.e. the output file `yellow_tripdata_2020-12.csv` of the `extract` task)?

- 128.3 MiB
- 134.5 MiB
- 364.7 MiB
- 692.6 MiB

### Question 2

What is the rendered value of the variable `file` when the inputs `taxi` is set to `green`, `year` is set to `2020`, and `month` is set to `04` during execution?

- `{{inputs.taxi}}_tripdata_{{inputs.year}}-{{inputs.month}}.csv`
- `green_tripdata_2020-04.csv`
- `green_tripdata_04_2020.csv`
- `green_tripdata_2020.csv`

### Question 3

How many rows are there for the `Yellow` Taxi data for all CSV files in the year 2020?

- 13,537.299
- 24,648,499
- 18,324,219
- 29,430,127

### Question 4

How many rows are there for the `Green` Taxi data for all CSV files in the year 2020?

- 5,327,301
- 936,199
- 1,734,051
- 1,342,034

### Question 5

How many rows are there for the `Yellow` Taxi data for the March 2021 CSV file?

- 1,428,092
- 706,911
- 1,925,152
- 2,561,031

### Question 6

How would you configure the timezone to New York in a Schedule trigger?

- Add a `timezone` property set to `EST` in the `Schedule` trigger configuration
- Add a `timezone` property set to `America/New_York` in the `Schedule` trigger configuration
- Add a `timezone` property set to `UTC-5` in the `Schedule` trigger configuration
- Add a `location` property set to `New_York` in the `Schedule` trigger configuration

---

## Module 3: Data Warehouse (BigQuery)

**Topics:** BigQuery, External Tables, Partitioning, Clustering

### Data Preparation

Use Yellow Taxi Trip Records for **January 2024 - June 2024** (Parquet files).

1. Create an external table using the Yellow Taxi Trip Records
2. Create a regular/materialized table (do not partition or cluster)

### Question 1

What is count of records for the 2024 Yellow Taxi Data?

- 65,623
- 840,402
- 20,332,093
- 85,431,289

### Question 2

Write a query to count the distinct number of PULocationIDs for the entire dataset on both tables.

What is the **estimated amount** of data that will be read when this query is executed on the External Table and the Table?

- 18.82 MB for the External Table and 47.60 MB for the Materialized Table
- 0 MB for the External Table and 155.12 MB for the Materialized Table
- 2.14 GB for the External Table and 0MB for the Materialized Table
- 0 MB for the External Table and 0MB for the Materialized Table

### Question 3

Write a query to retrieve the PULocationID from the table (not the external table) in BigQuery. Now write a query to retrieve the PULocationID and DOLocationID on the same table. Why are the estimated number of Bytes different?

- BigQuery is a columnar database, and it only scans the specific columns requested in the query. Querying two columns (PULocationID, DOLocationID) requires reading more data than querying one column (PULocationID), leading to a higher estimated number of bytes processed.
- BigQuery duplicates data across multiple storage partitions, so selecting two columns instead of one requires scanning the table twice, doubling the estimated bytes processed.
- BigQuery automatically caches the first queried column, so adding a second column increases processing time but does not affect the estimated bytes scanned.
- When selecting multiple columns, BigQuery performs an implicit join operation between them, increasing the estimated bytes processed

### Question 4

How many records have a fare_amount of 0?

- 128,210
- 546,578
- 20,188,016
- 8,333

### Question 5

What is the best strategy to make an optimized table in Big Query if your query will always filter based on tpep_dropoff_datetime and order the results by VendorID?

- Partition by tpep_dropoff_datetime and Cluster on VendorID
- Cluster on by tpep_dropoff_datetime and Cluster on VendorID
- Cluster on tpep_dropoff_datetime Partition by VendorID
- Partition by tpep_dropoff_datetime and Partition by VendorID

### Question 6

Write a query to retrieve the distinct VendorIDs between tpep_dropoff_datetime 2024-03-01 and 2024-03-15 (inclusive).

Use the materialized table you created earlier in your from clause and note the estimated bytes. Now change the table in the from clause to the partitioned table you created for question 5 and note the estimated bytes processed. What are these values?

- 12.47 MB for non-partitioned table and 326.42 MB for the partitioned table
- 310.24 MB for non-partitioned table and 26.84 MB for the partitioned table
- 5.87 MB for non-partitioned table and 0 MB for the partitioned table
- 310.31 MB for non-partitioned table and 285.64 MB for the partitioned table

### Question 7

Where is the data stored in the External Table you created?

- Big Query
- Container Registry
- GCP Bucket
- Big Table

### Question 8

It is best practice in Big Query to always cluster your data:

- True
- False

### Bonus Question 9 (Not worth points)

Write a `SELECT count(*)` query FROM the materialized table you created. How many bytes does it estimate will be read? Why?

---

## Module 4: Analytics Engineering (dbt)

**Topics:** dbt, Data Modeling, Staging, Marts, Testing

### Setup

1. Set up your dbt project following the setup guide
2. Load the Green and Yellow taxi data for 2019-2020 into your warehouse
3. Run `dbt build` to create all models and run tests

### Question 1: dbt Lineage and Execution

Given a dbt project with the following structure:

```
models/
├── staging/
│   ├── stg_green_tripdata.sql
│   └── stg_yellow_tripdata.sql
└── intermediate/
    └── int_trips_unioned.sql (depends on stg_green_tripdata & stg_yellow_tripdata)
```

If you run `dbt run --select int_trips_unioned`, what models will be built?

- `stg_green_tripdata`, `stg_yellow_tripdata`, and `int_trips_unioned` (upstream dependencies)
- Any model with upstream and downstream dependencies to `int_trips_unioned`
- `int_trips_unioned` only
- `int_trips_unioned`, `int_trips`, and `fct_trips` (downstream dependencies)

### Question 2: dbt Tests

You've configured a generic test like this in your `schema.yml`:

```yaml
columns:
  - name: payment_type
    data_tests:
      - accepted_values:
          arguments:
            values: [1, 2, 3, 4, 5]
```

Your model `fct_trips` has been running successfully for months. A new value `6` now appears in the source data.

What happens when you run `dbt test --select fct_trips`?

- dbt will skip the test because the model didn't change
- dbt will fail the test, returning a non-zero exit code
- dbt will pass the test with a warning about the new value
- dbt will update the configuration to include the new value

### Question 3: Counting Records in `fct_monthly_zone_revenue`

After running your dbt project, query the `fct_monthly_zone_revenue` model.

What is the count of records in the `fct_monthly_zone_revenue` model?

- 12,998
- 14,120
- 12,184
- 15,421

### Question 4: Best Performing Zone for Green Taxis (2020)

Using the `fct_monthly_zone_revenue` table, find the pickup zone with the **highest total revenue** (`revenue_monthly_total_amount`) for **Green** taxi trips in 2020.

Which zone had the highest revenue?

- East Harlem North
- Morningside Heights
- East Harlem South
- Washington Heights South

### Question 5: Green Taxi Trip Counts (October 2019)

Using the `fct_monthly_zone_revenue` table, what is the **total number of trips** (`total_monthly_trips`) for Green taxis in October 2019?

- 500,234
- 350,891
- 384,624
- 421,509

### Question 6: Build a Staging Model for FHV Data

Create a staging model for the **For-Hire Vehicle (FHV)** trip data for 2019.

1. Load the FHV trip data for 2019 into your data warehouse
2. Create a staging model `stg_fhv_tripdata` with these requirements:
   - Filter out records where `dispatching_base_num IS NULL`
   - Rename fields to match your project's naming conventions (e.g., `PUlocationID` → `pickup_location_id`)

What is the count of records in `stg_fhv_tripdata`?

- 42,084,899
- 43,244,696
- 22,998,722
- 44,112,187

---

## Module 5: Batch Processing (Spark)

**Topics:** Apache Spark, PySpark, DataFrames, Partitioning

### Data Preparation

```bash
wget https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-10.parquet
```

### Question 1: Install Spark and PySpark

- Install Spark
- Run PySpark
- Create a local spark session
- Execute `spark.version`

What's the output?

### Question 2: Yellow October 2024

Read the October 2024 Yellow into a Spark Dataframe.

Repartition the Dataframe to 4 partitions and save it to parquet.

What is the average size of the Parquet (ending with .parquet extension) Files that were created (in MB)?

- 6MB
- 25MB
- 75MB
- 100MB

### Question 3: Count records

How many taxi trips were there on the 15th of October?

Consider only trips that started on the 15th of October.

- 85,567
- 105,567
- 125,567
- 145,567

### Question 4: Longest trip

What is the length of the longest trip in the dataset in hours?

- 122
- 142
- 162
- 182

### Question 5: User Interface

Spark's User Interface which shows the application's dashboard runs on which local port?

- 80
- 443
- 4040
- 8080

### Question 6: Least frequent pickup location zone

Load the zone lookup data into a temp view in Spark:

```bash
wget https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv
```

Using the zone lookup data and the Yellow October 2024 data, what is the name of the LEAST frequent pickup location Zone?

- Governor's Island/Ellis Island/Liberty Island
- Arden Heights
- Rikers Island
- Jamaica Bay

---

## Module 6: Stream Processing (Kafka/PyFlink)

**Topics:** Kafka, Red Panda, PyFlink, Streaming, Session Windows

### Setup

Start the required services:

```bash
cd 06-streaming/pyflink/
docker-compose up
```

Services needed:
- Red Panda (Kafka replacement)
- Flink Job Manager
- Flink Task Manager
- Postgres

Postgres connection:
- Username: `postgres`
- Password: `postgres`
- Database: `postgres`
- Host: `localhost`
- Port: `5432`

Create the landing tables:

```sql
CREATE TABLE processed_events (
    test_data INTEGER,
    event_timestamp TIMESTAMP
);

CREATE TABLE processed_events_aggregated (
    event_hour TIMESTAMP,
    test_data INTEGER,
    num_hits INTEGER
);
```

### Question 1: Redpanda version

Check the output of the command `rpk help` inside the container (container name: `redpanda-1`).

What's the version, based on the output of the command you executed? (copy the entire version)

### Question 2: Creating a topic

Read the output of `help` and create a topic with name `green-trips`.

What's the output of the command for creating a topic? Include the entire output in your answer.

### Question 3: Connecting to the Kafka server

Install the kafka connector:

```bash
pip install kafka-python
```

Connect to the server:

```python
import json
from kafka import KafkaProducer

def json_serializer(data):
    return json.dumps(data).encode('utf-8')

server = 'localhost:9092'

producer = KafkaProducer(
    bootstrap_servers=[server],
    value_serializer=json_serializer
)

producer.bootstrap_connected()
```

What's the output of the last command?

### Question 4: Sending the Trip Data

Send the Green taxi data (October 2019) to the `green-trips` topic.

Keep only these columns:
- `lpep_pickup_datetime`
- `lpep_dropoff_datetime`
- `PULocationID`
- `DOLocationID`
- `passenger_count`
- `trip_distance`
- `tip_amount`

Send all data using:

```python
producer.send(topic_name, value=message)
```

After sending all messages, flush:

```python
producer.flush()
```

Measure the time:

```python
from time import time

t0 = time()
# ... your code
t1 = time()
took = t1 - t0
```

How much time did it take to send the entire dataset and flush?

### Question 5: Build a Sessionization Window (2 points)

Process the data from the Kafka stream:

1. Copy `aggregation_job.py` and rename it to `session_job.py`
2. Have it read from `green-trips` fixing the schema
3. Use a session window with a gap of 5 minutes
4. Use `lpep_dropoff_datetime` time as your watermark with a 5 second tolerance

Which pickup and drop off locations have the longest unbroken streak of taxi trips?

---

## Submission Links

| Module | Form URL |
|--------|----------|
| Module 1 | https://courses.datatalks.club/de-zoomcamp-2026/homework/hw1 |
| Module 2 | https://courses.datatalks.club/de-zoomcamp-2026/homework/hw2 |
| Module 3 | https://courses.datatalks.club/de-zoomcamp-2026/homework/hw3 |
| Module 4 | https://courses.datatalks.club/de-zoomcamp-2026/homework/hw4 |
| Module 5 | https://courses.datatalks.club/de-zoomcamp-2026/homework/hw5 |
| Module 6 | https://courses.datatalks.club/de-zoomcamp-2026/homework/hw6 |
