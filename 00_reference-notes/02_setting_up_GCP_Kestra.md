# Setting Up GCP with Kestra

This guide explains how to configure Google Cloud Platform (GCP) to work with Kestra for workflow orchestration. You'll learn how to safely store credentials, create GCS buckets, and set up BigQuery datasets.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Creating a GCP Service Account](#creating-a-gcp-service-account)
3. [Downloading the Service Account Key](#downloading-the-service-account-key)
4. [Adding GCP Credentials to Kestra](#adding-gcp-credentials-to-kestra)
5. [Configuring GCP Settings in Kestra KV Store](#configuring-gcp-settings-in-kestra-kv-store)
6. [Creating GCS Buckets and BigQuery Datasets](#creating-gcs-buckets-and-bigquery-datasets)
7. [Using GCP Credentials in Workflows](#using-gcp-credentials-in-workflows)
8. [Security Best Practices](#security-best-practices)

---

## Prerequisites

Before you begin, ensure you have:

- A Google Cloud Platform account
- A GCP project created (note the Project ID)
- Kestra running locally via Docker Compose
- Basic familiarity with the GCP Console

---

## Creating a GCP Service Account

A service account allows Kestra to authenticate with GCP services without using your personal credentials.

### Step 1: Navigate to IAM & Admin

1. Go to the [GCP Console](https://console.cloud.google.com/)
2. Select your project from the dropdown
3. Navigate to **IAM & Admin** > **Service Accounts**

### Step 2: Create the Service Account

1. Click **+ CREATE SERVICE ACCOUNT**
2. Enter a name (e.g., `kestra-service-account`)
3. Enter a description (e.g., "Service account for Kestra workflow orchestration")
4. Click **CREATE AND CONTINUE**

### Step 3: Grant Required Roles

Assign the following roles to the service account:

| Role | Purpose |
|------|---------|
| `Storage Admin` | Create and manage GCS buckets, upload/download files |
| `BigQuery Admin` | Create datasets, tables, and run queries |
| `Storage Object Admin` | Full control over GCS objects |

To add roles:
1. Click **Select a role**
2. Search for and select each role
3. Click **+ ADD ANOTHER ROLE** to add more
4. Click **CONTINUE** when done

### Step 4: Skip User Access (Optional)

1. The "Grant users access" step is optional for this use case
2. Click **DONE**

---

## Downloading the Service Account Key

### Step 1: Generate the Key

1. In the Service Accounts list, find your newly created account
2. Click the three dots menu (⋮) on the right
3. Select **Manage keys**
4. Click **ADD KEY** > **Create new key**
5. Select **JSON** format
6. Click **CREATE**

The JSON key file will automatically download to your computer.

### Step 2: Secure the Key File

**IMPORTANT: This key file grants access to your GCP resources. Treat it like a password.**

- Never commit the key file to Git
- Never share the key file publicly
- Store it in a secure location
- Consider deleting the local file after adding it to Kestra

The JSON key file looks like this:

```json
{
  "type": "service_account",
  "project_id": "your-project-id",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n",
  "client_email": "kestra-service-account@your-project-id.iam.gserviceaccount.com",
  "client_id": "...",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "..."
}
```

---

## Adding GCP Credentials to Kestra

Kestra uses a Key-Value (KV) Store to securely store sensitive configuration values. We'll store the GCP service account credentials here.

### Method 1: Via Kestra UI (Recommended)

1. Open Kestra UI at [http://localhost:8080](http://localhost:8080)
2. Log in with your credentials (default: `admin@kestra.io` / `Admin1234`)
3. Navigate to **Namespaces** in the left sidebar
4. Select or create the namespace you'll use (e.g., `zoomcamp`)
5. Go to the **KV Store** tab
6. Click **+ Add a KV pair**
7. Set the key as `GCP_CREDS`
8. Paste the **entire contents** of your JSON key file as the value
9. Click **Save**

### Method 2: Via Kestra Flow

You can also set the credentials programmatically by creating a flow:

```yaml
id: set_gcp_credentials
namespace: zoomcamp

tasks:
  - id: set_gcp_creds
    type: io.kestra.plugin.core.kv.Set
    key: GCP_CREDS
    kvType: STRING
    value: |
      {
        "type": "service_account",
        "project_id": "your-project-id",
        ... (paste your entire JSON key content here)
      }
```

**Warning:** If using this method, delete the flow immediately after execution to avoid storing credentials in your flow history.

---

## Configuring GCP Settings in Kestra KV Store

Besides credentials, you need to configure other GCP settings. Use the following flow to set all required KV values:

### Flow: `06_gcp_kv.yaml`

```yaml
id: 06_gcp_kv
namespace: zoomcamp

tasks:
  - id: gcp_project_id
    type: io.kestra.plugin.core.kv.Set
    key: GCP_PROJECT_ID
    kvType: STRING
    value: your-project-id  # Replace with your GCP project ID

  - id: gcp_location
    type: io.kestra.plugin.core.kv.Set
    key: GCP_LOCATION
    kvType: STRING
    value: us-central1  # Choose your preferred region

  - id: gcp_bucket_name
    type: io.kestra.plugin.core.kv.Set
    key: GCP_BUCKET_NAME
    kvType: STRING
    value: your-unique-bucket-name  # Must be globally unique!

  - id: gcp_dataset
    type: io.kestra.plugin.core.kv.Set
    key: GCP_DATASET
    kvType: STRING
    value: zoomcamp  # Your BigQuery dataset name
```

### Configuration Values Explained

| Key | Description | Example |
|-----|-------------|---------|
| `GCP_PROJECT_ID` | Your GCP project identifier | `my-data-project-123` |
| `GCP_LOCATION` | Region for resources | `us-central1`, `europe-west2` |
| `GCP_BUCKET_NAME` | GCS bucket name (must be globally unique) | `yourname-kestra-data` |
| `GCP_DATASET` | BigQuery dataset name | `zoomcamp` |

### How to Execute

1. Copy the flow YAML to Kestra UI editor
2. Modify the values for your project
3. Save and execute the flow
4. Verify in the KV Store that all values are set

---

## Creating GCS Buckets and BigQuery Datasets

Once credentials and configuration are set, you can create the required GCP resources using Kestra.

### Flow: `07_gcp_setup.yaml`

```yaml
id: 07_gcp_setup
namespace: zoomcamp

tasks:
  - id: create_gcs_bucket
    type: io.kestra.plugin.gcp.gcs.CreateBucket
    ifExists: SKIP
    storageClass: REGIONAL
    name: "{{kv('GCP_BUCKET_NAME')}}"

  - id: create_bq_dataset
    type: io.kestra.plugin.gcp.bigquery.CreateDataset
    name: "{{kv('GCP_DATASET')}}"
    ifExists: SKIP

pluginDefaults:
  - type: io.kestra.plugin.gcp
    values:
      serviceAccount: "{{kv('GCP_CREDS')}}"
      projectId: "{{kv('GCP_PROJECT_ID')}}"
      location: "{{kv('GCP_LOCATION')}}"
      bucket: "{{kv('GCP_BUCKET_NAME')}}"
```

### What This Flow Does

1. **Creates a GCS Bucket**: Uses the `GCP_BUCKET_NAME` from KV Store with `REGIONAL` storage class
2. **Creates a BigQuery Dataset**: Uses the `GCP_DATASET` name from KV Store
3. **Uses Plugin Defaults**: Automatically applies credentials and project settings to all GCP tasks

### Execution

1. Save the flow in Kestra
2. Click **Execute**
3. Monitor the execution logs
4. Verify in GCP Console:
   - Go to **Cloud Storage** > **Buckets** to see your bucket
   - Go to **BigQuery** > **SQL Workspace** to see your dataset

---

## Using GCP Credentials in Workflows

When building workflows that interact with GCP, use the `pluginDefaults` section to apply credentials automatically.

### Example: Upload to GCS and Load to BigQuery

```yaml
id: gcp_data_pipeline
namespace: zoomcamp

tasks:
  - id: extract_data
    type: io.kestra.plugin.scripts.shell.Commands
    outputFiles:
      - "*.csv"
    commands:
      - wget -qO- https://example.com/data.csv.gz | gunzip > data.csv

  - id: upload_to_gcs
    type: io.kestra.plugin.gcp.gcs.Upload
    from: "{{outputs.extract_data.outputFiles['data.csv']}}"
    to: "gs://{{kv('GCP_BUCKET_NAME')}}/data/data.csv"

  - id: load_to_bigquery
    type: io.kestra.plugin.gcp.bigquery.Query
    sql: |
      CREATE OR REPLACE EXTERNAL TABLE `{{kv('GCP_PROJECT_ID')}}.{{kv('GCP_DATASET')}}.my_table`
      OPTIONS (
        format = 'CSV',
        uris = ['gs://{{kv('GCP_BUCKET_NAME')}}/data/data.csv'],
        skip_leading_rows = 1
      );

# Apply GCP credentials to all GCP plugin tasks
pluginDefaults:
  - type: io.kestra.plugin.gcp
    values:
      serviceAccount: "{{kv('GCP_CREDS')}}"
      projectId: "{{kv('GCP_PROJECT_ID')}}"
      location: "{{kv('GCP_LOCATION')}}"
      bucket: "{{kv('GCP_BUCKET_NAME')}}"
```

### Key Patterns

1. **Reference KV values with `{{kv('KEY_NAME')}}`**
2. **Use `pluginDefaults` to avoid repeating credentials** in every task
3. **GCS paths use the format**: `gs://bucket-name/path/to/file`
4. **BigQuery table references use**: `` `project_id.dataset.table` ``

---

## Security Best Practices

### Do's

- Store credentials in Kestra's KV Store, not in flow YAML
- Use the principle of least privilege when assigning IAM roles
- Regularly rotate service account keys
- Delete local copies of key files after adding to Kestra
- Use separate service accounts for different environments (dev/prod)

### Don'ts

- Never commit service account keys to Git
- Never hardcode credentials in flow YAML files
- Never share credentials via email or chat
- Never use your personal account for automation
- Never grant more permissions than necessary

### Checking for Exposed Credentials

If you accidentally commit credentials:

1. Immediately revoke the key in GCP Console
2. Generate a new key
3. Update Kestra KV Store with the new key
4. Remove the commit from Git history (use `git filter-branch` or BFG Repo-Cleaner)

---

## Troubleshooting

### Common Errors

**Error: "Permission denied" or "403 Forbidden"**
- Verify the service account has the required roles
- Check that `GCP_CREDS` contains the complete JSON key
- Ensure `GCP_PROJECT_ID` matches the project in your credentials

**Error: "Bucket name already exists"**
- GCS bucket names are globally unique
- Choose a different name that includes your username or organization

**Error: "Invalid credentials"**
- Re-download the JSON key from GCP Console
- Ensure no extra whitespace or formatting when pasting into KV Store
- Verify the service account hasn't been deleted or disabled

**Error: "Dataset not found" in BigQuery queries**
- Run the `07_gcp_setup.yaml` flow first
- Verify `GCP_DATASET` KV value matches your dataset name
- Check the dataset was created in the correct region

### Verifying Setup

Run this test flow to verify your GCP configuration:

```yaml
id: test_gcp_connection
namespace: zoomcamp

tasks:
  - id: list_buckets
    type: io.kestra.plugin.gcp.gcs.List
    from: "gs://{{kv('GCP_BUCKET_NAME')}}/"

  - id: test_bigquery
    type: io.kestra.plugin.gcp.bigquery.Query
    sql: SELECT 1 as test_value

pluginDefaults:
  - type: io.kestra.plugin.gcp
    values:
      serviceAccount: "{{kv('GCP_CREDS')}}"
      projectId: "{{kv('GCP_PROJECT_ID')}}"
      location: "{{kv('GCP_LOCATION')}}"
```

If both tasks succeed, your GCP integration is working correctly.

---

## Quick Reference

### KV Store Keys Required

| Key | Description |
|-----|-------------|
| `GCP_CREDS` | Full JSON content of service account key |
| `GCP_PROJECT_ID` | Your GCP project ID |
| `GCP_LOCATION` | Region (e.g., `us-central1`) |
| `GCP_BUCKET_NAME` | Globally unique bucket name |
| `GCP_DATASET` | BigQuery dataset name |

### Useful Links

- [GCP Console](https://console.cloud.google.com/)
- [Kestra GCP Plugin Docs](https://kestra.io/plugins/plugin-gcp)
- [GCP Service Account Docs](https://cloud.google.com/iam/docs/service-accounts)
- [BigQuery Regions](https://cloud.google.com/bigquery/docs/locations)
- [GCS Storage Classes](https://cloud.google.com/storage/docs/storage-classes)
