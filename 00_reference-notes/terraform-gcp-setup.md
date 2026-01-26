# Terraform and GCP Setup

## Module 1 Summary

Module 1 covers the foundational tools for data engineering:

| Topic | Technologies | Purpose |
|-------|--------------|---------|
| Containerization | Docker, Docker Compose | Reproducible environments, isolated applications |
| Database | PostgreSQL, pgAdmin | Relational database for data storage |
| Infrastructure as Code | Terraform | Automated cloud resource provisioning |
| Cloud Platform | GCP (GCS, BigQuery) | Data lake and data warehouse |

**Learning Path:**
1. Docker basics and containerization concepts
2. Running PostgreSQL in Docker
3. Data ingestion with Python (pandas, SQLAlchemy)
4. pgAdmin for database management
5. Docker Compose for multi-container setups
6. GCP account setup and service accounts
7. Terraform for provisioning GCS buckets and BigQuery datasets

---

## What is Terraform?

Terraform is an open-source Infrastructure-as-Code (IaC) tool by HashiCorp that provisions and manages cloud infrastructure.

### Why Use Terraform?

- **Infrastructure lifecycle management** - Create, update, destroy resources
- **Version control** - Track infrastructure changes in git
- **State-based** - Tracks resource state to detect drift
- **Multi-cloud** - Works with AWS, GCP, Azure, Kubernetes, etc.
- **Reproducible** - Same configuration deploys identical infrastructure

### Core Concepts

| Concept | Description |
|---------|-------------|
| Provider | Plugin that interfaces with cloud APIs (google, aws, azurerm) |
| Resource | Infrastructure component to create (bucket, database, VM) |
| State | Terraform's record of managed infrastructure |
| Plan | Preview of changes before applying |
| Module | Reusable group of resources |

---

## Terraform Files

| File | Purpose |
|------|---------|
| `main.tf` | Primary configuration (providers, resources) |
| `variables.tf` | Variable declarations |
| `terraform.tfvars` | Variable values (don't commit secrets!) |
| `outputs.tf` | Output values after apply |
| `.tfstate` | State file (auto-generated, don't edit) |

---

## Terraform Commands

```bash
# Initialize - download providers, setup backend
terraform init

# Preview changes
terraform plan

# Apply changes (creates/updates infrastructure)
terraform apply

# Destroy all resources
terraform destroy

# Format configuration files
terraform fmt

# Validate configuration
terraform validate
```

---

## GCP Initial Setup

### 1. Create GCP Account and Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project (e.g., "DE-Zoomcamp")
3. Note your **Project ID** (used in Terraform configs)

### 2. Create Service Account

1. Go to **IAM & Admin** → **Service Accounts**
2. Click **Create Service Account**
3. Name it (e.g., "terraform-runner")
4. Grant these roles:
   - **Viewer** (basic access)
   - **Storage Admin** (manage GCS buckets)
   - **Storage Object Admin** (manage objects in buckets)
   - **BigQuery Admin** (manage BigQuery datasets/tables)
5. Click **Done**

### 3. Generate Service Account Key

1. Click on your service account
2. Go to **Keys** tab
3. Click **Add Key** → **Create new key**
4. Select **JSON** format
5. Download and save securely (e.g., `~/.gc/my-creds.json`)

### 4. Enable Required APIs

Enable these APIs in your project:
- [IAM API](https://console.cloud.google.com/apis/library/iam.googleapis.com)
- [IAM Credentials API](https://console.cloud.google.com/apis/library/iamcredentials.googleapis.com)

### 5. Install Google Cloud SDK

```bash
# Linux/macOS - using install script
curl https://sdk.cloud.google.com | bash

# Or download from: https://cloud.google.com/sdk/docs/quickstart
```

### 6. Configure Authentication

```bash
# Set credentials environment variable
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.gc/my-creds.json"

# Add to ~/.bashrc or ~/.zshrc to persist

# Authenticate and verify
gcloud auth application-default login
```

---

## Terraform Configuration for GCP

### Basic Configuration (Single File)

```hcl
# main.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "5.6.0"
    }
  }
}

provider "google" {
  # Uses GOOGLE_APPLICATION_CREDENTIALS env var
  project = "<Your-Project-ID>"
  region  = "us-central1"
}

# Google Cloud Storage bucket (Data Lake)
resource "google_storage_bucket" "data-lake-bucket" {
  name          = "<globally-unique-bucket-name>"
  location      = "US"
  storage_class = "STANDARD"

  uniform_bucket_level_access = true

  versioning {
    enabled = true
  }

  lifecycle_rule {
    action {
      type = "Delete"
    }
    condition {
      age = 30  # days
    }
  }

  force_destroy = true
}

# BigQuery dataset (Data Warehouse)
resource "google_bigquery_dataset" "dataset" {
  dataset_id = "demo_dataset"
  project    = "<Your-Project-ID>"
  location   = "US"
}
```

### Configuration with Variables (Recommended)

**main.tf:**
```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "5.6.0"
    }
  }
}

provider "google" {
  credentials = file(var.credentials)
  project     = var.project
  region      = var.region
}

resource "google_storage_bucket" "data-lake-bucket" {
  name          = var.gcs_bucket_name
  location      = var.location
  force_destroy = true

  lifecycle_rule {
    condition {
      age = 1
    }
    action {
      type = "AbortIncompleteMultipartUpload"
    }
  }
}

resource "google_bigquery_dataset" "dataset" {
  dataset_id = var.bq_dataset_name
  location   = var.location
}
```

**variables.tf:**
```hcl
variable "credentials" {
  description = "Path to service account JSON key"
  default     = "~/.gc/my-creds.json"
}

variable "project" {
  description = "GCP Project ID"
  default     = "<Your-Project-ID>"
}

variable "region" {
  description = "Region for resources"
  default     = "us-central1"
}

variable "location" {
  description = "Location for GCS and BigQuery"
  default     = "US"
}

variable "bq_dataset_name" {
  description = "BigQuery dataset name"
  default     = "demo_dataset"
}

variable "gcs_bucket_name" {
  description = "GCS bucket name (must be globally unique)"
  default     = "my-unique-bucket-name-12345"
}
```

---

## Deploying Infrastructure

```bash
# 1. Navigate to terraform directory
cd 01-docker-terraform/terraform/terraform/terraform_with_variables

# 2. Update variables.tf with your project ID and unique bucket name

# 3. Initialize Terraform
terraform init

# 4. Preview changes
terraform plan

# 5. Apply (type 'yes' to confirm)
terraform apply

# 6. When done, destroy resources to avoid charges
terraform destroy
```

---

## Windows Setup (Without WSL)

If not using WSL, you need additional setup:

### Google Cloud SDK on Windows

1. Use GitBash, MinGW, or Cygwin for Linux-like environment
2. Download SDK: https://dl.google.com/dl/cloudsdk/channels/rapid/google-cloud-sdk.zip
3. Unzip and run `install.sh`
4. Add to PATH: `C:\tools\google-cloud-sdk\bin`
5. Set Python path (if using Anaconda):
   ```bash
   export CLOUDSDK_PYTHON=~/Anaconda3/python
   ```

### Authentication on Windows

```bash
# Set credentials path
export GOOGLE_APPLICATION_CREDENTIALS=~/.gc/my-creds.json

# Authenticate with service account
gcloud auth activate-service-account --key-file $GOOGLE_APPLICATION_CREDENTIALS

# Or use OAuth
gcloud auth application-default login
```

---

## Common Errors and Fixes

### "quota exceeded" or "API not enabled"

```bash
gcloud auth application-default set-quota-project <Your-Project-ID>
```

### "does not have storage.buckets.create access"

Your service account needs **Storage Admin** role. Check IAM permissions in GCP Console.

### "bucket name already exists"

GCS bucket names are globally unique. Choose a different name.

### State Lock Error

Someone else is running Terraform, or a previous run crashed. Wait or manually unlock:
```bash
terraform force-unlock <lock-id>
```

---

## GCP Resources Created

| Resource | Purpose | Terraform Resource Type |
|----------|---------|------------------------|
| GCS Bucket | Data Lake - store raw files | `google_storage_bucket` |
| BigQuery Dataset | Data Warehouse - store structured data | `google_bigquery_dataset` |

---

## Quick Reference

```bash
# Set GCP credentials
export GOOGLE_APPLICATION_CREDENTIALS="$HOME/.gc/my-creds.json"

# Terraform workflow
terraform init      # Initialize
terraform plan      # Preview
terraform apply     # Deploy
terraform destroy   # Cleanup

# Check GCP authentication
gcloud auth list
gcloud config list project
```
