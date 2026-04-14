# 5-Day Execution Plan — FDA FAERS Analytics Pipeline

**Cohort:** 2026 | **Start:** 2026-04-14 | **Deadline:** 2026-04-18 (end of day)

---

## Overview

| Day | Focus | Deliverable |
|-----|-------|-------------|
| 1 (Apr 14) | Repo scaffolding + GCP infrastructure | Terraform applies cleanly; GCS bucket + BQ datasets exist |
| 2 (Apr 15) | Data ingestion — FAERS download → GCS → BigQuery raw | Kestra flow runs end-to-end for at least one quarter |
| 3 (Apr 16) | dbt project — staging + intermediate layers | `dbt run` + `dbt test` pass on staging/intermediate |
| 4 (Apr 17) | dbt marts + data validation | All mart models run; dashboard data is queryable |
| 5 (Apr 18) | Looker Studio dashboard + README + submission polish | 2+ dashboard tiles live; repo is fully reproducible |

---

## Day 1 — Infrastructure & Repo Scaffold (Apr 14)

**Goal:** Stand up all cloud infrastructure and establish the project repo structure.

### Authentication approach

This project uses **Application Default Credentials (ADC)** throughout — no service account key files are created or stored anywhere. This is required because the GCP org policy `iam.disableServiceAccountKeyCreation` is enforced.

| Context | Auth method |
|---------|-------------|
| Local (Terraform, dbt, gcloud CLI) | ADC via `gcloud auth application-default login` |
| Kestra in local Docker | ADC file mounted read-only into the container |
| Kestra on GCE (cloud deployment) | SA attached to VM via Workload Identity — no key needed |

The `faers-pipeline-sa` service account created by Terraform is still the pipeline's identity. On GCP compute it is attached to the instance; locally the ADC file is used directly.

### Tasks

- [x] Scaffold directory structure inside `00_Final_Project/`:
  ```
  terraform/          ← main.tf, variables.tf, outputs.tf, terraform.tfvars (gitignored)
  kestra/flows/
  dbt/faers_analytics/models/{staging,intermediate,marts}/
  docker-compose.yml
  .gitignore
  README.md (stub)
  ```
- [x] Write `terraform/main.tf` — GCS bucket, 3 BigQuery datasets, `faers-pipeline-sa` with IAM roles
- [x] Write `terraform/variables.tf` and `terraform/outputs.tf`
- [ ] Authenticate with GCP:
  ```bash
  gcloud auth login
  gcloud auth application-default login
  gcloud config set project YOUR_PROJECT_ID
  ```
- [ ] Fill in `terraform/terraform.tfvars` (copied from `terraform.tfvars.example`):
  ```hcl
  project         = "your-gcp-project-id"
  gcs_bucket_name = "de-zoomcamp-faers-lake-your-gcp-project-id"
  ```
- [ ] Run Terraform:
  ```bash
  cd 00_Final_Project/terraform
  terraform init
  terraform plan    # expect: 1 bucket, 3 BQ datasets, 1 SA, 3 IAM bindings
  terraform apply
  ```

### Verification
```bash
# Find your project ID if unsure
gcloud projects list
gcloud config get-value project

# Confirm bucket exists
gcloud storage ls gs://de-zoomcamp-faers-lake-YOUR_PROJECT_ID

# Confirm all 3 datasets exist
bq ls --project_id=YOUR_PROJECT_ID
# Expected: faers_raw  faers_staging  faers_mart
```

### Watch out for
- `terraform.tfvars` and `.env` are gitignored — never commit them
- GCS bucket names are globally unique across all GCP accounts — suffix with your project ID to be safe
- Terraform state is local for now (`terraform.tfstate`) — gitignored, don't lose it
- Billing must be enabled on the GCP project before `terraform apply`

---

## Day 2 — Data Ingestion: Kestra Flow (Apr 15)

**Goal:** Parameterised Kestra flow that downloads one FAERS quarter, uploads to GCS, and loads raw tables into BigQuery.

### Tasks

- [ ] Write `docker-compose.yml` for local Kestra (reuse pattern from `02-workflow-orchestration/`), mounting ADC credentials:
  ```yaml
  volumes:
    - ~/.config/gcloud/application_default_credentials.json:/tmp/gcp-adc.json:ro
  environment:
    - GOOGLE_APPLICATION_CREDENTIALS=/tmp/gcp-adc.json
    - GCP_PROJECT_ID=your-project-id
    - GCS_BUCKET=de-zoomcamp-faers-lake-your-project-id
  ```
- [ ] Start local Kestra: `docker compose up -d`
- [ ] Write `kestra/flows/faers_quarterly_ingest.yaml` with these tasks in order:
  1. `http.Download` — fetch ZIP from FDA HTTPS endpoint, parameterised by `year` + `quarter`
  2. `io.UnzipFiles` — extract to temp directory; validate presence of DEMO, DRUG, REAC, OUTC, RPSR, THER, INDI files
  3. `gcs.Upload` — upload each `.txt` to `gs://de-zoomcamp-faers-lake/raw/faers/{year}{quarter}/`
  4. `bigquery.Load` — load each `.txt` into `faers_raw.{filename}_{year}{quarter}` as append-only, auto-detect schema
  5. `notifications` — log success/failure (use Kestra `io.kestra.core.tasks.log.Log` for simplicity)
- [ ] Test flow manually in Kestra UI with inputs `year=2023, quarter=Q4`
- [ ] Run backfill for 2019 Q1 → 2024 Q4 (trigger each quarter manually or via a loop flow)

### FDA Download URL pattern
```
https://fis.fda.gov/content/Exports/faers_ascii_{year}q{N}.zip
# Example: https://fis.fda.gov/content/Exports/faers_ascii_2023q4.zip
```

### BigQuery raw table naming
```
faers_raw.demo_2023q4
faers_raw.drug_2023q4
faers_raw.reac_2023q4
faers_raw.outc_2023q4
faers_raw.rpsr_2023q4
faers_raw.ther_2023q4
faers_raw.indi_2023q4
```

### Verification
```bash
# Check files landed in GCS
gcloud storage ls gs://de-zoomcamp-faers-lake/raw/faers/2023Q4/

# Check row counts in BQ
bq query --use_legacy_sql=false \
  'SELECT COUNT(*) FROM faers_raw.demo_2023q4'
```

### Watch out for
- FAERS ZIP files use `$` as delimiter in some quarters — set `fieldDelimiter: "$"` in the BigQuery Load task; test against an older quarter
- File names inside the ZIP vary (`DEMO23Q4.txt` vs `demo23q4.txt`) — normalise to lowercase before upload
- BigQuery auto-detect schema can mis-type numeric fields as strings — acceptable at raw layer; fix in dbt staging
- FDA server can be slow; set an HTTP timeout of at least 120s in Kestra

---

## Day 3 — dbt Staging + Intermediate Layers (Apr 16)

**Goal:** `dbt run` and `dbt test` pass for all staging and intermediate models.

### Tasks

- [ ] Initialise dbt project: `dbt init faers_analytics` inside `dbt/`
- [ ] Configure `profiles.yml` for BigQuery using ADC (oauth method — no key file):
  ```yaml
  faers_analytics:
    target: dev
    outputs:
      dev:
        type: bigquery
        method: oauth                 # uses ADC / gcloud auth application-default login
        project: "{{ env_var('GCP_PROJECT_ID') }}"
        dataset: faers_staging
        location: US
        threads: 4
      prod:
        type: bigquery
        method: oauth
        project: "{{ env_var('GCP_PROJECT_ID') }}"
        dataset: faers_staging
        location: US
        threads: 8
  ```
- [ ] Write `dbt_project.yml`:
  - staging → `+materialized: view`
  - intermediate → `+materialized: table`
  - marts → `+materialized: table`
- [ ] Write `models/staging/sources.yml` — point to `faers_raw` dataset; define all 7 source tables
- [ ] Write staging models (one per FAERS file):

  **`stg_faers_demographics.sql`**
  - Cast `age` to FLOAT64, `receipt_date` to DATE
  - Standardise `sex` → `'M'`, `'F'`, `'UNK'`
  - Filter to latest version per `caseid`: `ROW_NUMBER() OVER (PARTITION BY caseid ORDER BY primaryid DESC) = 1`

  **`stg_faers_drugs.sql`**
  - `UPPER(TRIM(drugname))` for normalisation
  - Keep `role_cod` (PS = primary suspect, SS = secondary suspect, C = concomitant, I = interacting)

  **`stg_faers_reactions.sql`**
  - Deduplicate: `DISTINCT primaryid, UPPER(pt)` (MedDRA preferred term)

  **`stg_faers_outcomes.sql`**
  - Map raw outcome codes to labels:
    ```
    DE = 'Death', LT = 'Life-Threatening', HO = 'Hospitalisation',
    DS = 'Disability', CA = 'Congenital Anomaly', RI = 'Required Intervention', OT = 'Other'
    ```

- [ ] Write `models/intermediate/int_reports_enriched.sql`
  - Join `stg_faers_demographics` + `stg_faers_drugs` + `stg_faers_reactions` + `stg_faers_outcomes` on `primaryid`
  - Add boolean column `is_serious` (true if any outcome in DE, LT, HO, DS, CA, RI)

- [ ] Write `models/staging/schema.yml` with tests:
  - `not_null` + `unique` on `primaryid` in demographics
  - `accepted_values` on `sex`: `['M', 'F', 'UNK']`
  - `accepted_values` on `role_cod`: `['PS', 'SS', 'C', 'I']`

- [ ] Run and verify:
  ```bash
  cd dbt/faers_analytics
  dbt deps
  dbt run --select staging
  dbt test --select staging
  dbt run --select intermediate
  ```

### Watch out for
- `caseid` deduplication is critical — without it, FAERS inflates counts because amended reports exist alongside originals
- Some `receipt_date` values are malformed (nulls, future dates) — coerce with `SAFE.PARSE_DATE`
- Drug name normalisation won't be perfect; accept ~80% quality at this stage, document limitations in README

---

## Day 4 — dbt Marts + End-to-End Validation (Apr 17)

**Goal:** All mart models run cleanly; analytical queries return sensible results.

### Tasks

- [ ] Write `models/marts/mart_drug_safety_signals.sql`
  ```sql
  -- Per drug: total reports, serious count, % serious
  SELECT
    drugname,
    COUNT(DISTINCT primaryid)                          AS total_reports,
    COUNTIF(is_serious)                                AS serious_reports,
    ROUND(COUNTIF(is_serious) / COUNT(DISTINCT primaryid) * 100, 2) AS pct_serious
  FROM {{ ref('int_reports_enriched') }}
  WHERE role_cod = 'PS'   -- primary suspect only
  GROUP BY 1
  ORDER BY total_reports DESC
  ```

- [ ] Write `models/marts/mart_reaction_trends.sql`
  ```sql
  -- Monthly report volume by MedDRA reaction term
  SELECT
    DATE_TRUNC(receipt_date, MONTH)  AS report_month,
    pt                               AS reaction_term,
    COUNT(DISTINCT primaryid)        AS report_count
  FROM {{ ref('int_reports_enriched') }}
  GROUP BY 1, 2
  ```

- [ ] Write `models/marts/mart_patient_demographics.sql`
  ```sql
  -- Breakdown by age group + sex
  SELECT
    CASE
      WHEN age < 18  THEN 'Pediatric (<18)'
      WHEN age < 45  THEN 'Adult (18-44)'
      WHEN age < 65  THEN 'Middle-aged (45-64)'
      WHEN age >= 65 THEN 'Senior (65+)'
      ELSE 'Unknown'
    END AS age_group,
    sex,
    pt  AS top_reaction,
    COUNT(DISTINCT primaryid) AS report_count
  FROM {{ ref('int_reports_enriched') }}
  GROUP BY 1, 2, 3
  ```

- [ ] Add partitioning/clustering config to mart models via `dbt_project.yml` or model-level config blocks:
  ```sql
  {{ config(
      partition_by = {'field': 'report_month', 'data_type': 'date', 'granularity': 'month'},
      cluster_by  = ['reaction_term']
  ) }}
  ```

- [ ] Run full pipeline:
  ```bash
  dbt run
  dbt test
  dbt docs generate && dbt docs serve  # sanity check lineage
  ```

- [ ] Spot-check mart outputs:
  ```sql
  -- Top 10 drugs by serious adverse events
  SELECT drugname, total_reports, pct_serious
  FROM faers_mart.mart_drug_safety_signals
  ORDER BY serious_reports DESC
  LIMIT 10;

  -- Report volume trend (last 12 months)
  SELECT report_month, SUM(report_count) AS monthly_total
  FROM faers_mart.mart_reaction_trends
  WHERE report_month >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
  GROUP BY 1 ORDER BY 1;
  ```

### Watch out for
- If `dbt run` is slow on the full backfill (~20M rows), use `--vars '{"year_filter": 2023}'` to limit scope during dev, then remove the filter for the final run
- BigQuery partition limits: each `INSERT` creating >4000 partitions will fail — the mart queries aggregate to monthly granularity, which keeps partition count manageable

---

## Day 5 — Dashboard + README + Submission Polish (Apr 18)

**Goal:** Looker Studio dashboard live, repo fully reproducible, submission-ready.

### Tasks

#### Morning: Dashboard (Looker Studio)
- [ ] Connect Looker Studio to BigQuery project `faers_mart` dataset
- [ ] **Tile 1 — Temporal Trend (line chart)**
  - Source: `mart_reaction_trends`
  - Dimension: `report_month`, Breakdown: `is_serious` (add this flag to mart or filter at source)
  - Metric: `SUM(report_count)`
  - Date range control: default last 5 years
- [ ] **Tile 2 — Top 20 Drugs by Adverse Events (bar chart)**
  - Source: `mart_drug_safety_signals`
  - Dimension: `drugname` (top 20 by `total_reports`)
  - Metric: `total_reports`, secondary metric `pct_serious`
  - Add dropdown filter: Year
- [ ] **Tile 3 (bonus) — Demographics Heatmap**
  - Source: `mart_patient_demographics`
  - Dimensions: `age_group` × `top_reaction` (filter to top 10 reactions)
  - Metric: `report_count`
- [ ] Set dashboard to "public, anyone with the link can view"
- [ ] Copy shareable URL for README

#### Afternoon: README + Reproducibility
- [ ] Write the final `README.md` with these sections:
  1. **Problem description** — 2–3 paragraphs, reference the 4 business questions
  2. **Architecture diagram** — copy the ASCII diagram from the project plan
  3. **Tech stack** — table from the project plan
  4. **Reproducibility — step-by-step:**
     ```
     Step 1: Prerequisites (gcloud CLI, Terraform, Docker, dbt, Python 3.11+)
     Step 2: Authenticate — gcloud auth login && gcloud auth application-default login
     Step 3: cd terraform && terraform init && terraform apply
     Step 4: docker compose up -d  (starts local Kestra)
     Step 5: Trigger Kestra flow for desired quarters
     Step 6: cd dbt/faers_analytics && dbt deps && dbt run && dbt test
     Step 7: Open Looker Studio dashboard link
     ```
  5. **Dashboard link**
  6. **Data considerations** — deduplication note, delimiter note, scope (2019–present)

- [ ] Final checklist before submission:
  - [ ] `terraform destroy` leaves no orphaned resources (test in a scratch project if time allows)
  - [ ] `terraform.tfvars` and `.env` are in `.gitignore`; confirm no JSON key files exist in the repo
  - [ ] `dbt test` passes with zero failures
  - [ ] Looker Studio dashboard URL is accessible in an incognito window
  - [ ] All 7 evaluation criteria covered in README

---

## Evaluation Criteria Checklist

| # | Criterion (4 pts each) | Where it lives | Status |
|---|------------------------|----------------|--------|
| 1 | Problem description | README intro section | |
| 2 | Cloud (GCP, IaC) | `terraform/` | |
| 3 | Data ingestion (batch pipeline) | `kestra/flows/` | |
| 4 | Data warehouse (partition + cluster) | dbt model configs + BQ | |
| 5 | Transformations (3 dbt layers) | `dbt/faers_analytics/models/` | |
| 6 | Dashboard (≥2 tiles) | Looker Studio URL in README | |
| 7 | Reproducibility (step-by-step README) | README reproducibility section | |

**Target: 28/28**

---

## Risk Register

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| FDA site slow / ZIP download fails | Medium | Pre-download 2–3 quarters manually on Day 2 before writing the Kestra flow; store locally as fallback |
| ADC token expiry during long Kestra runs | Low | Re-run `gcloud auth application-default login` to refresh; token lasts ~1 hour by default |
| Kestra flow debugging takes too long | Medium | Test each task in isolation before wiring them together |
| BigQuery costs spike during backfill | Low | Scope backfill to 2022–2024 (3 years) first; full history is bonus |
| dbt tests fail on data quality issues | High | Use `severity: warn` for `accepted_values` tests — don't block the run |
| Looker Studio permissions issue | Low | Test incognito window immediately after setting public access |

---

## Key Technical Notes (from data exploration)

- **Delimiter**: FAERS ASCII files use `$` as field delimiter and `\n` as row delimiter
- **Deduplication**: Always `ROW_NUMBER() OVER (PARTITION BY caseid ORDER BY primaryid DESC) = 1` in staging to keep latest report version only
- **Scope**: Limit to 2019 Q1 onward — pre-2012 files use a legacy format, and older data has lower data quality
- **Drug name normalisation**: `UPPER(TRIM(drugname))` catches most duplicates; `REGEXP_REPLACE` to strip trailing dosage info is a bonus
- **Age outliers**: FAERS contains ages in years, decades, and sometimes months — filter `age < 150` to remove obviously corrupt values; cast `age_cod = 'DEC'` (decades) by multiplying by 10
