# PostgreSQL and pgAdmin Setup with Docker Compose

This guide covers setting up PostgreSQL database and pgAdmin using Docker Compose, plus configuring pgAdmin to manage your database.

## Docker Compose Configuration

Create a `docker-compose.yaml` file:

```yaml
services:
  pgdatabase:
    image: postgres:18
    environment:
      POSTGRES_USER: "root"
      POSTGRES_PASSWORD: "root"
      POSTGRES_DB: "ny_taxi"
    volumes:
      - ny_taxi_postgres_data:/var/lib/postgresql
    ports:
      - "5432:5432"

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: "admin@admin.com"
      PGADMIN_DEFAULT_PASSWORD: "root"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    ports:
      - "8085:80"

volumes:
  ny_taxi_postgres_data:
  pgadmin_data:
```

### Configuration Explained

**PostgreSQL Service (`pgdatabase`):**
- `POSTGRES_USER` - Database superuser name
- `POSTGRES_PASSWORD` - Superuser password
- `POSTGRES_DB` - Default database created on startup
- Volume persists data across container restarts
- Port 5432 exposed to host

**pgAdmin Service:**
- `PGADMIN_DEFAULT_EMAIL` - Login email for web interface
- `PGADMIN_DEFAULT_PASSWORD` - Login password
- Volume persists settings and server connections
- Port 8085 on host maps to port 80 in container

**Networking:**
Docker Compose automatically creates a shared network for all services. Containers can reach each other using their service names (e.g., `pgdatabase`).

## Starting and Stopping Services

```bash
# Start services in background
docker compose up -d

# Start services in foreground (see logs)
docker compose up

# Stop services
docker compose down

# Stop services and remove volumes (deletes all data)
docker compose down -v

# View logs
docker compose logs
```

## Configuring pgAdmin to Connect to PostgreSQL

### Step 1: Access pgAdmin

1. Open your browser and navigate to `http://localhost:8085`
2. Login with:
   - Email: `admin@admin.com`
   - Password: `root`

### Step 2: Register the PostgreSQL Server

1. In the left sidebar, right-click on **Servers**
2. Select **Register** → **Server...**

### Step 3: Configure the Connection

**General Tab:**
- Name: `Local Docker` (or any descriptive name)

**Connection Tab:**
- Host name/address: `pgdatabase` (the Docker service name)
- Port: `5432`
- Maintenance database: `postgres`
- Username: `root`
- Password: `root`
- Toggle "Save password" if desired

Click **Save**.

### Step 4: Explore Your Database

After connecting, you can:
- Expand **Servers** → **Local Docker** → **Databases** → **ny_taxi**
- View tables under **Schemas** → **public** → **Tables**
- Right-click tables to view/edit data
- Use **Query Tool** (right-click database → Query Tool) to run SQL

## pgAdmin Dashboard Features

### Query Tool
- Right-click any database → **Query Tool**
- Write and execute SQL queries
- View results in tabular format
- Export results to CSV

### Dashboard
- Click on a server to see real-time metrics
- Monitor active sessions, transactions per second
- View server activity and locks

### Backup and Restore
- Right-click database → **Backup** to create a dump
- Right-click **Databases** → **Create** → **Database** to create new databases
- Right-click database → **Restore** to restore from backup

### Table Management
- Right-click **Tables** → **Create** → **Table** to create tables
- Right-click a table → **View/Edit Data** to browse records
- Right-click a table → **Properties** to modify schema

## Alternative: Running Containers Manually

If you prefer not to use Docker Compose, run containers manually with a shared network:

```bash
# Create network
docker network create pg-network

# Run PostgreSQL
docker run -it \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="ny_taxi" \
  -v ny_taxi_postgres_data:/var/lib/postgresql \
  -p 5432:5432 \
  --network=pg-network \
  --name pgdatabase \
  postgres:18

# Run pgAdmin (in another terminal)
docker run -it \
  -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
  -e PGADMIN_DEFAULT_PASSWORD="root" \
  -v pgadmin_data:/var/lib/pgadmin \
  -p 8085:80 \
  --network=pg-network \
  --name pgadmin \
  dpage/pgadmin4
```

## Connecting from Host Machine

To connect to PostgreSQL from your host machine (outside Docker):

```bash
# Using psql
psql -h localhost -p 5432 -U root -d ny_taxi

# Using pgcli
pgcli -h localhost -p 5432 -u root -d ny_taxi
```

When connecting from the host, use `localhost`. When connecting from another container (like pgAdmin), use the service name `pgdatabase`.
