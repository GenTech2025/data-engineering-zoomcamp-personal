# Docker Basics and Docker Compose

## What is Docker?

Docker is containerization software that isolates applications similar to virtual machines but much more lightweight. A Docker **image** is a snapshot of a container that can be exported and run anywhere Docker is installed.

### Why Use Docker?

- **Reproducibility** - Same environment everywhere (dev, CI, production)
- **Isolation** - Applications run independently without conflicts
- **Portability** - Run anywhere Docker is installed

### Common Use Cases

- CI/CD integration tests
- Running data pipelines on the cloud (AWS Batch, Kubernetes)
- Apache Spark clusters
- Serverless functions (AWS Lambda, Google Cloud Functions)

## Essential Docker Commands

### Running Containers

```bash
# Check Docker version
docker --version

# Run a simple test container
docker run hello-world

# Run interactively (-it = interactive + tty)
docker run -it ubuntu

# Run and auto-remove when stopped
docker run -it --rm ubuntu

# Run with custom entrypoint
docker run -it --rm --entrypoint=bash python:3.13-slim
```

### Container Management

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop a container
docker stop <container_id>

# Remove a container
docker rm <container_id>

# Remove all stopped containers
docker rm $(docker ps -aq)

# View container logs
docker logs <container_id>
```

### Image Management

```bash
# List images
docker images

# Pull an image
docker pull postgres:18

# Remove an image
docker rmi <image_name>

# Build an image from Dockerfile
docker build -t myimage:tag .
```

## Stateless Containers

Containers are **stateless** - changes made inside a container are lost when it stops. This is a feature, not a bug:

```bash
# Start container, make changes
docker run -it ubuntu
apt update && apt install python3
exit

# Start again - python3 is gone!
docker run -it ubuntu
python3  # command not found
```

To persist data, use **volumes**.

## Volumes

Volumes allow data to persist and be shared between host and container.

### Bind Mounts (Host Path)

Map a host directory to a container directory:

```bash
docker run -it --rm \
  -v $(pwd)/data:/app/data \
  python:3.13-slim
```

### Named Volumes (Docker Managed)

Docker manages the storage location:

```bash
docker run -it --rm \
  -v mydata:/app/data \
  python:3.13-slim
```

| Type | Syntax | Use Case |
|------|--------|----------|
| Bind mount | `/host/path:/container/path` | Development, sharing code |
| Named volume | `volume_name:/container/path` | Database storage, persistence |

## Environment Variables

Pass configuration to containers:

```bash
docker run -it --rm \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="secret" \
  -e POSTGRES_DB="mydb" \
  postgres:18
```

## Port Mapping

Expose container ports to the host:

```bash
# Map host port 5432 to container port 5432
docker run -it --rm -p 5432:5432 postgres:18

# Map to different host port
docker run -it --rm -p 8080:80 nginx
```

Format: `-p <host_port>:<container_port>`

## Docker Networks

Containers on the same network can communicate using container names:

```bash
# Create a network
docker network create my-network

# Run containers on the network
docker run -d --network=my-network --name db postgres:18
docker run -d --network=my-network --name app myapp:latest

# The app container can reach db at hostname "db"
```

```bash
# List networks
docker network ls

# Remove a network
docker network rm my-network
```

## Writing Dockerfiles

A Dockerfile defines how to build an image.

### Basic Dockerfile (with pip)

```dockerfile
# Base image
FROM python:3.13-slim

# Install dependencies
RUN pip install pandas pyarrow

# Set working directory
WORKDIR /app

# Copy files into image
COPY script.py script.py

# Default command
ENTRYPOINT ["python", "script.py"]
```

### Dockerfile with uv (Recommended)

```dockerfile
FROM python:3.13-slim

# Copy uv from official image
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/

WORKDIR /app

# Add venv to PATH
ENV PATH="/app/.venv/bin:$PATH"

# Copy dependency files first (better layer caching)
COPY pyproject.toml uv.lock .python-version ./

# Install dependencies
RUN uv sync --locked

# Copy application code
COPY . .

ENTRYPOINT ["uv", "run", "python", "main.py"]
```

### Dockerfile Instructions

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image to build on |
| `RUN` | Execute commands during build |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files from host to image |
| `ENV` | Set environment variables |
| `EXPOSE` | Document which ports the container listens on |
| `ENTRYPOINT` | Default executable |
| `CMD` | Default arguments to ENTRYPOINT |

### Building and Running

```bash
# Build image
docker build -t myapp:v1 .

# Run container
docker run -it --rm myapp:v1

# Pass arguments
docker run -it --rm myapp:v1 arg1 arg2
```

## Docker Compose

Docker Compose manages multi-container applications with a single YAML file.

### Basic docker-compose.yaml

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: "root"
      POSTGRES_PASSWORD: "root"
      POSTGRES_DB: "mydb"
    volumes:
      - db_data:/var/lib/postgresql
    ports:
      - "5432:5432"

  web:
    build: ./app
    ports:
      - "8080:80"
    depends_on:
      - db
    environment:
      DATABASE_URL: "postgresql://root:root@db:5432/mydb"

volumes:
  db_data:
```

### Key Features

- **Automatic networking** - All services share a network and can reach each other by service name
- **Volume management** - Define named volumes in the `volumes:` section
- **Build from Dockerfile** - Use `build:` instead of `image:`
- **Dependencies** - `depends_on:` controls startup order

### Docker Compose Commands

```bash
# Start all services (detached)
docker compose up -d

# Start in foreground (see logs)
docker compose up

# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v

# View logs
docker compose logs

# View logs for specific service
docker compose logs db

# Rebuild images
docker compose build

# Restart a service
docker compose restart web
```

### Connecting External Containers

When running a container outside Compose that needs to connect to Compose services:

```bash
# Find the network name
docker network ls
# Usually: <directory>_default

# Run container on that network
docker run -it --rm \
  --network=myproject_default \
  myimage:latest
```

## Cleanup Commands

```bash
# Remove all stopped containers
docker container prune

# Remove unused images
docker image prune

# Remove unused volumes
docker volume prune

# Remove everything unused
docker system prune

# Nuclear option: remove everything
docker system prune -a --volumes
```

## Quick Reference

| Task | Command |
|------|---------|
| Run interactive container | `docker run -it --rm <image>` |
| Build image | `docker build -t name:tag .` |
| List containers | `docker ps -a` |
| List images | `docker images` |
| View logs | `docker logs <container>` |
| Start Compose services | `docker compose up -d` |
| Stop Compose services | `docker compose down` |
| Connect to running container | `docker exec -it <container> bash` |
