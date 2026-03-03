# Local Environment Setup (Fully Offline, No Cloud)

This guide sets up the **entire Data Engineering Zoomcamp** to run locally with **zero cloud dependencies**. Every module uses a local alternative where a cloud service would otherwise be required.

We use **conda** as the environment manager to keep all installed tools isolated from your global system and avoid dependency conflicts.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Cloud-to-Local Alternatives](#cloud-to-local-alternatives)
- [Part 1: Install Conda](#part-1-install-conda)
- [Part 2: System-Level Tools](#part-2-system-level-tools)
- [Part 3: Conda Environment for Module 1 (Docker + PostgreSQL)](#part-3-conda-environment-for-module-1-docker--postgresql)
- [Part 4: Module 2 (Kestra Workflow Orchestration)](#part-4-module-2-kestra-workflow-orchestration)
- [Part 5: Conda Environment for Module 3 & 4 (DuckDB + dbt)](#part-5-conda-environment-for-module-3--4-duckdb--dbt)
- [Part 6: Conda Environment for Module 5 (Spark + PySpark)](#part-6-conda-environment-for-module-5-spark--pyspark)
- [Part 7: Module 6 (Kafka + Flink Streaming)](#part-7-module-6-kafka--flink-streaming)
- [Quick Reference: Conda Commands](#quick-reference-conda-commands)
- [Port Allocation Map](#port-allocation-map)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before starting, ensure you have:

- **Operating System**: Linux (Ubuntu 20.04+), macOS (10.14+), or Windows 10/11 with WSL2
- **RAM**: 8 GB minimum, 16 GB recommended
- **Disk Space**: ~50 GB free (Docker images, datasets, Spark, Java)
- **Internet**: Required only during setup (downloading tools, data, Docker images)

---

## Cloud-to-Local Alternatives

| Module | Cloud Service | Local Alternative | Notes |
|--------|--------------|-------------------|-------|
| 1 | GCP (Terraform) | PostgreSQL in Docker | Terraform section skipped; Docker + SQL fully local |
| 2 | None | Kestra in Docker | Already runs locally |
| 3 | BigQuery + GCS | DuckDB | Same SQL concepts, local analytical DB |
| 4 | BigQuery + dbt Cloud | DuckDB + dbt Core (CLI) | Full dbt project runs locally |
| 5 | GCP Dataproc | Spark standalone mode | Runs on your machine |
| 6 | None | Redpanda + Flink in Docker | All containers local |

---

## Part 1: Install Conda

We use **Miniforge** (a minimal conda installer that defaults to the `conda-forge` channel). It is lighter than full Anaconda and avoids licensing concerns.

### Linux / WSL2

```bash
# Download Miniforge installer
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh

# Run installer (follow prompts, accept defaults)
bash Miniforge3-Linux-x86_64.sh

# Restart your shell or source the config
source ~/.bashrc

# Verify installation
conda --version
```

### macOS

```bash
# Intel Mac
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-x86_64.sh

# Apple Silicon (M1/M2/M3/M4)
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh

# Run installer
bash Miniforge3-*.sh

source ~/.zshrc
conda --version
```

### Useful Conda Configuration

```bash
# Prevent conda from auto-activating the base environment on every new shell
conda config --set auto_activate_base false

# Ensure conda-forge is the default channel
conda config --add channels conda-forge
conda config --set channel_priority strict
```

---

## Part 2: System-Level Tools

These tools are required across multiple modules and are installed **outside** conda at the system level.

### Docker & Docker Compose

Docker runs containerized services (PostgreSQL, Kestra, Kafka, Flink). Install it at the system level since containers should be independent of any conda environment.

**Linux (Ubuntu/Debian):**

```bash
# Install Docker Engine
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Allow running Docker without sudo
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

**macOS:**

Install [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/) from the official website.

**Verify Docker works:**

```bash
docker run --rm hello-world
```

### Git

```bash
# Linux
sudo apt-get install -y git

# macOS (included with Xcode CLI tools)
xcode-select --install

git --version
```

### wget and curl

```bash
# Linux
sudo apt-get install -y wget curl

# macOS (curl is preinstalled, install wget)
brew install wget
```

### make (for Module 6)

```bash
# Linux
sudo apt-get install -y make

# macOS (included with Xcode CLI tools)
```

---

## Part 3: Conda Environment for Module 1 (Docker + PostgreSQL)

Module 1 uses Docker for PostgreSQL and pgAdmin, and Python for a data ingestion pipeline.

### Create the Environment

```bash
conda create -n de-zoomcamp-m1 python=3.13 -y
conda activate de-zoomcamp-m1
```

### Install Python Dependencies

```bash
pip install \
  click>=8.3.1 \
  pandas>=2.3.3 \
  psycopg2-binary>=2.9.11 \
  pyarrow>=22.0.0 \
  sqlalchemy>=2.0.44 \
  tqdm>=4.67.1

# Optional dev tools
pip install jupyter pgcli
```

### Start PostgreSQL + pgAdmin with Docker Compose

```bash
cd 01-docker-terraform/docker-sql/pipeline/
docker compose up -d
```

This starts:

| Service | Image | Port | Access |
|---------|-------|------|--------|
| PostgreSQL | `postgres:18` | `5432` | `psql -h localhost -U root -d ny_taxi` |
| pgAdmin | `dpage/pgadmin4` | `8085` | http://localhost:8085 (admin@admin.com / root) |

### Verify

```bash
# Test PostgreSQL connection via pgcli
pgcli -h localhost -p 5432 -U root -d ny_taxi
# Password: root
```

### Run the Ingestion Pipeline (Dockerized)

The pipeline itself runs inside a Docker container, so no additional host installs are needed:

```bash
# Build the pipeline image
docker build -t taxi-pipeline .

# Run the ingestion (example - check pipeline.py --help for options)
docker run --network=pipeline_default taxi-pipeline --help
```

### Stop Services When Done

```bash
docker compose down
conda deactivate
```

---

## Part 4: Module 2 (Kestra Workflow Orchestration)

Module 2 runs entirely in Docker. No conda environment is needed -- just Docker Compose.

### Start Kestra + PostgreSQL

```bash
cd 02-workflow-orchestration/
docker compose up -d
```

This starts:

| Service | Image | Port | Access |
|---------|-------|------|--------|
| Kestra | `kestra/kestra:v1.1` | `8080` | http://localhost:8080 (admin@kestra.io / Admin1234) |
| Kestra Postgres | `postgres:18` | Internal | Kestra metadata store |
| NY Taxi Postgres | `postgres:18` | `5432` | Data pipeline target |
| pgAdmin | `dpage/pgadmin4` | `8085` | http://localhost:8085 |

### Verify

Open http://localhost:8080 in your browser. Log in with:
- **Username**: admin@kestra.io
- **Password**: Admin1234

### Stop Services When Done

```bash
docker compose down
```

> **Note:** Kestra workflows are defined as YAML files in `02-workflow-orchestration/flows/`. You edit them in the Kestra UI or in your local editor and upload them.

---

## Part 5: Conda Environment for Module 3 & 4 (DuckDB + dbt)

Modules 3 and 4 share the same local stack: **DuckDB** replaces BigQuery and **dbt Core** replaces dbt Cloud.

### Create the Environment

```bash
conda create -n de-zoomcamp-dbt python=3.11 -y
conda activate de-zoomcamp-dbt
```

> We use Python 3.11 here because `dbt-core` and `dbt-duckdb` have the broadest compatibility with Python 3.9-3.12. Python 3.13 may cause issues with some dbt dependencies.

### Install DuckDB + dbt

```bash
# dbt-duckdb installs both dbt-core and the DuckDB adapter
pip install dbt-duckdb

# Also install standalone DuckDB CLI and Python library
conda install duckdb -y

# Install requests for the data ingestion script
pip install requests
```

### Verify Installed Versions

```bash
dbt --version
duckdb --version
```

### Configure dbt Profile

Create the dbt profile file that tells dbt how to connect to DuckDB:

```bash
mkdir -p ~/.dbt
```

Add the following to `~/.dbt/profiles.yml` (create the file if it doesn't exist):

```yaml
taxi_rides_ny:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: taxi_rides_ny.duckdb
      schema: dev
      threads: 1
      extensions:
        - parquet
      settings:
        memory_limit: '2GB'
        preserve_insertion_order: false

    prod:
      type: duckdb
      path: taxi_rides_ny.duckdb
      schema: prod
      threads: 1
      extensions:
        - parquet
      settings:
        memory_limit: '2GB'
        preserve_insertion_order: false
```

> **Tip:** If you have 16 GB+ RAM, increase `memory_limit` to `'4GB'` for faster builds. If you have less than 4 GB RAM, reduce to `'1GB'`.

### Download and Load NYC Taxi Data

Navigate to the dbt project and run the ingestion script:

```bash
cd 04-analytics-engineering/taxi_rides_ny/
```

Create and run the ingestion script (or use the one in `04-analytics-engineering/setup/` if it exists):

```python
# save as: load_data.py
import duckdb
import requests
from pathlib import Path

BASE_URL = "https://github.com/DataTalksClub/nyc-tlc-data/releases/download"

def download_and_convert_files(taxi_type):
    data_dir = Path("data") / taxi_type
    data_dir.mkdir(exist_ok=True, parents=True)

    for year in [2019, 2020]:
        for month in range(1, 13):
            parquet_filename = f"{taxi_type}_tripdata_{year}-{month:02d}.parquet"
            parquet_filepath = data_dir / parquet_filename

            if parquet_filepath.exists():
                print(f"Skipping {parquet_filename} (already exists)")
                continue

            csv_gz_filename = f"{taxi_type}_tripdata_{year}-{month:02d}.csv.gz"
            csv_gz_filepath = data_dir / csv_gz_filename

            print(f"Downloading {csv_gz_filename}...")
            response = requests.get(f"{BASE_URL}/{taxi_type}/{csv_gz_filename}", stream=True)
            response.raise_for_status()

            with open(csv_gz_filepath, 'wb') as f:
                for chunk in response.iter_content(chunk_size=8192):
                    f.write(chunk)

            print(f"Converting {csv_gz_filename} to Parquet...")
            con = duckdb.connect()
            con.execute(f"""
                COPY (SELECT * FROM read_csv_auto('{csv_gz_filepath}'))
                TO '{parquet_filepath}' (FORMAT PARQUET)
            """)
            con.close()

            csv_gz_filepath.unlink()
            print(f"Completed {parquet_filename}")

if __name__ == "__main__":
    for taxi_type in ["yellow", "green"]:
        download_and_convert_files(taxi_type)

    con = duckdb.connect("taxi_rides_ny.duckdb")
    con.execute("CREATE SCHEMA IF NOT EXISTS prod")

    for taxi_type in ["yellow", "green"]:
        con.execute(f"""
            CREATE OR REPLACE TABLE prod.{taxi_type}_tripdata AS
            SELECT * FROM read_parquet('data/{taxi_type}/*.parquet', union_by_name=true)
        """)

    con.close()
    print("Data loaded into DuckDB successfully.")
```

```bash
python load_data.py
```

### Verify dbt Connection

```bash
dbt debug
```

Expected output:

```
  Connection test: [OK connection ok]
  All checks passed!
```

### Build the dbt Project

```bash
# Install dbt packages (dbt_utils, codegen)
dbt deps

# Load seed reference data (payment types, taxi zones)
dbt seed

# Run all models (staging -> intermediate -> marts)
dbt run

# Run all tests
dbt test

# Or do it all in one command
dbt build
```

### Using DuckDB for Module 3 Exercises

Module 3 teaches BigQuery SQL concepts. You can practice the same queries locally with DuckDB:

```bash
# Launch DuckDB CLI against the database
duckdb taxi_rides_ny.duckdb
```

```sql
-- Example: count records by year
SELECT EXTRACT(YEAR FROM tpep_pickup_datetime) AS year, COUNT(*) AS trips
FROM prod.yellow_tripdata
GROUP BY 1
ORDER BY 1;
```

DuckDB supports nearly all the same SQL syntax as BigQuery (window functions, CTEs, `DATE_TRUNC`, `EXTRACT`, etc.) so Module 3 exercises translate directly.

### Stop When Done

```bash
conda deactivate
```

---

## Part 6: Conda Environment for Module 5 (Spark + PySpark)

Module 5 uses Apache Spark for batch processing. Spark requires Java 11, which conda can install for us, keeping it isolated from your system Java.

### Create the Environment

```bash
conda create -n de-zoomcamp-spark python=3.10 -y
conda activate de-zoomcamp-spark
```

> Python 3.10 is used because PySpark 3.3.x has best compatibility with Python 3.7-3.10.

### Install Java 11 via Conda

This is a key advantage of conda -- Java is installed **inside the environment** and won't affect your system:

```bash
conda install openjdk=11 -y
```

Verify:

```bash
java --version
# Should show: openjdk 11.x.x
```

`JAVA_HOME` is automatically set by conda within the activated environment. Verify:

```bash
echo $JAVA_HOME
# Should point to something like: /home/<user>/miniforge3/envs/de-zoomcamp-spark/
```

> If `JAVA_HOME` is not set automatically, add it to the environment's activation script:
> ```bash
> conda env config vars set JAVA_HOME=$CONDA_PREFIX
> conda deactivate && conda activate de-zoomcamp-spark
> ```

### Install Apache Spark

```bash
# Download Spark 3.3.2 (version used in the course)
wget -P /tmp https://archive.apache.org/dist/spark/spark-3.3.2/spark-3.3.2-bin-hadoop3.tgz

# Extract into the conda environment's share directory to keep it isolated
mkdir -p $CONDA_PREFIX/share/spark
tar xzf /tmp/spark-3.3.2-bin-hadoop3.tgz -C $CONDA_PREFIX/share/spark --strip-components=1

# Clean up
rm /tmp/spark-3.3.2-bin-hadoop3.tgz
```

### Set Environment Variables

Create activation/deactivation scripts so these variables are set automatically whenever you activate the environment:

```bash
# Create activation script
mkdir -p $CONDA_PREFIX/etc/conda/activate.d
cat > $CONDA_PREFIX/etc/conda/activate.d/spark_env.sh << 'EOF'
export SPARK_HOME="$CONDA_PREFIX/share/spark"
export PATH="$SPARK_HOME/bin:$PATH"
export PYTHONPATH="$SPARK_HOME/python/:$PYTHONPATH"
export PYTHONPATH="$SPARK_HOME/python/lib/py4j-0.10.9.5-src.zip:$PYTHONPATH"
EOF

# Create deactivation script (clean up on deactivate)
mkdir -p $CONDA_PREFIX/etc/conda/deactivate.d
cat > $CONDA_PREFIX/etc/conda/deactivate.d/spark_env.sh << 'EOF'
unset SPARK_HOME
# PATH and PYTHONPATH are restored automatically by conda
EOF
```

Reactivate the environment to load the new variables:

```bash
conda deactivate && conda activate de-zoomcamp-spark
```

### Install PySpark and Jupyter

```bash
pip install jupyter
```

> PySpark itself is bundled with the Spark installation at `$SPARK_HOME/python/`. The `PYTHONPATH` exports above make it importable. You do **not** need to `pip install pyspark` separately.

### Verify Spark

**Test the Scala shell:**

```bash
spark-shell
```

In the Spark shell:

```scala
val data = 1 to 10000
val distData = sc.parallelize(data)
distData.filter(_ < 10).collect()
// Expected: Array[Int] = Array(1, 2, 3, 4, 5, 6, 7, 8, 9)
```

Type `:quit` to exit.

**Test PySpark:**

```bash
cd 05-batch/code/

# Download test data
wget https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv
```

```python
# Run in ipython or jupyter notebook
import pyspark
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .master("local[*]") \
    .appName('test') \
    .getOrCreate()

df = spark.read \
    .option("header", "true") \
    .csv('taxi_zone_lookup.csv')

df.show()

# Test writing
df.write.parquet('zones')
```

### Stop When Done

```bash
conda deactivate
```

---

## Part 7: Module 6 (Kafka + Flink Streaming)

Module 6 runs Redpanda (Kafka-compatible), Apache Flink, and PostgreSQL entirely in Docker containers. The Python dependencies for interacting with Kafka from the host can be installed in a conda environment.

### Docker Services (No Conda Needed)

Start the full streaming stack:

```bash
cd 06-streaming/pyflink/

# Build the custom Flink image and start all services
make up
# Or equivalently: docker compose up --build --remove-orphans -d
```

This starts:

| Service | Image | Port | Access |
|---------|-------|------|--------|
| Redpanda (Kafka) | `redpandadata/redpanda:v24.2.18` | `9092` | Kafka broker |
| Redpanda REST | (same container) | `8082` | REST proxy API |
| Flink JobManager | `pyflink:1.16.0` (custom) | `8081` | http://localhost:8081 (Flink UI) |
| Flink TaskManager | `pyflink:1.16.0` (custom) | Internal | Worker node |
| PostgreSQL | `postgres:14` | `5432` | Sink database |

### Submit a Flink Job

```bash
make job
```

### Optional: Conda Environment for Host-Side Kafka Scripts

If you want to run Kafka producer/consumer scripts from your host machine (outside Docker):

```bash
conda create -n de-zoomcamp-streaming python=3.10 -y
conda activate de-zoomcamp-streaming

pip install kafka-python requests
```

> **Note:** `apache-flink==1.16.0` requires Python 3.6-3.8 and is installed **inside the Docker container**, not on the host. The Flink Dockerfile handles this with Python 3.7.9. You only need `kafka-python` on the host if you want to produce/consume messages from outside the containers.

### Verify

```bash
# Check that all containers are running
docker compose ps

# Open Flink UI
# Navigate to http://localhost:8081
```

### Stop Services

```bash
make down
# Or: docker compose down --remove-orphans

conda deactivate  # if using the streaming conda env
```

---

## Quick Reference: Conda Commands

```bash
# ── Environment Management ────────────────────────
conda env list                          # List all environments
conda activate <env-name>              # Activate an environment
conda deactivate                       # Deactivate current environment
conda remove -n <env-name> --all -y    # Delete an environment entirely

# ── Package Management ────────────────────────────
conda list                             # List installed packages in active env
conda install <package> -y             # Install a conda package
pip install <package>                  # Install a pip package (within active env)
pip list                               # List pip-installed packages

# ── Environments Created in This Guide ────────────
# de-zoomcamp-m1        -> Module 1 (Python 3.13, pandas, sqlalchemy, pgcli)
# de-zoomcamp-dbt       -> Module 3 & 4 (Python 3.11, dbt-duckdb, duckdb)
# de-zoomcamp-spark     -> Module 5 (Python 3.10, Java 11, Spark 3.3.2)
# de-zoomcamp-streaming -> Module 6 (Python 3.10, kafka-python) [optional]
```

---

## Port Allocation Map

Different modules use overlapping ports. **Do not run multiple modules simultaneously** unless you remap ports.

| Port | Module 1 | Module 2 | Module 4 | Module 5 | Module 6 |
|------|----------|----------|----------|----------|----------|
| `5432` | PostgreSQL | PostgreSQL (NY Taxi) | -- | -- | PostgreSQL (Flink sink) |
| `8080` | -- | Kestra UI | -- | -- | -- |
| `8081` | -- | Kestra internal | -- | -- | Flink UI |
| `8082` | -- | -- | -- | -- | Redpanda REST |
| `8085` | pgAdmin | pgAdmin | -- | -- | -- |
| `9092` | -- | -- | -- | -- | Kafka broker |

**Rule of thumb:** Run `docker compose down` in one module before starting another to free up ports.

---

## Troubleshooting

### Conda

| Problem | Solution |
|---------|----------|
| `conda: command not found` after install | Run `source ~/.bashrc` (Linux) or `source ~/.zshrc` (macOS) |
| Slow conda environment creation | Use `conda install mamba -y` then replace `conda create` with `mamba create` for faster solves |
| Package conflict between pip and conda | Install conda packages first, then pip packages. Avoid mixing both for the same package. |

### Docker

| Problem | Solution |
|---------|----------|
| `permission denied` when running Docker | Run `sudo usermod -aG docker $USER` then log out and back in |
| Port already in use | Run `docker compose down` in any other module directory, or check with `lsof -i :<port>` |
| Docker containers using too much disk | Run `docker system prune -a` to reclaim space (removes unused images/containers) |

### Module 5 (Spark)

| Problem | Solution |
|---------|----------|
| `JAVA_HOME is not set` | Run `conda env config vars set JAVA_HOME=$CONDA_PREFIX` then reactivate |
| `ModuleNotFoundError: No module named 'py4j'` | Check the py4j zip filename: `ls $SPARK_HOME/python/lib/py4j-*` and update the PYTHONPATH export in the activation script to match |
| Spark shell hangs or is very slow | Ensure you have at least 4 GB free RAM. Close other heavy applications. |

### Module 3 & 4 (dbt + DuckDB)

| Problem | Solution |
|---------|----------|
| `dbt debug` fails with connection error | Ensure you are in the `taxi_rides_ny/` directory and `~/.dbt/profiles.yml` exists with correct config |
| `dbt run` out of memory | Reduce `memory_limit` in profiles.yml, or run models one at a time with `dbt run --select <model>` |
| Data ingestion script fails | Ensure `requests` is installed (`pip install requests`) and you have internet access for the initial download |

---

## Summary

| What | Where |
|------|-------|
| Conda environments | Isolate Python + Java per module |
| Docker Compose | Runs PostgreSQL, Kestra, Kafka, Flink as containers |
| DuckDB | Replaces BigQuery for Modules 3 & 4 |
| Spark standalone | Replaces Dataproc for Module 5 |
| Redpanda | Replaces cloud Kafka for Module 6 |

With this setup, every module of the Data Engineering Zoomcamp runs on your local machine with no cloud accounts, no billing, and no external dependencies after initial installation.