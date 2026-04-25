# 🏥 Azure Healthcare Data Pipeline
### Medallion Architecture (Bronze → Silver → Gold) + Security + CI/CD

![Azure](https://img.shields.io/badge/Azure-Data%20Factory-blue?logo=microsoftazure)
![PowerBI](https://img.shields.io/badge/Power%20BI-Dashboard-yellow?logo=powerbi)
![KeyVault](https://img.shields.io/badge/Azure-Key%20Vault-orange?logo=microsoftazure)
![CI/CD](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-green?logo=githubactions)
![Status](https://img.shields.io/badge/Pipeline-Live%20%26%20Scheduled-brightgreen)

---

## 📋 Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Phase 1: Azure Environment Setup](#phase-1-azure-environment-setup)
- [Phase 2: Linked Services](#phase-2-linked-services)
- [Phase 3: Bronze Layer](#phase-3-bronze-layer--raw-ingestion)
- [Phase 4: Silver Layer](#phase-4-silver-layer--cleaning--transformation)
- [Phase 5: Gold Layer](#phase-5-gold-layer--aggregation--sql)
- [Phase 6: Master Pipeline & Scheduling](#phase-6-master-pipeline--scheduling)
- [Phase 7: Power BI Dashboard](#phase-7-power-bi-dashboard)
- [Phase 8: Azure Key Vault](#phase-8-azure-key-vault--security)
- [Phase 9: CI/CD with GitHub Actions](#phase-9-cicd-with-github-actions)
- [Errors Faced & Fixes](#-errors-faced--fixes)
- [Pipeline Performance](#pipeline-performance)
- [Repository Structure](#repository-structure)
- [Next Steps](#next-steps)

---

## 🎯 Project Overview

A fully automated, end-to-end healthcare data pipeline built on **Azure Data Factory** using the **Medallion Architecture** pattern. The pipeline ingests live COVID-19 statistics from a public REST API, cleans and transforms the data through three layers, stores aggregated results in Azure SQL Database, and visualizes insights in Power BI — all secured with Azure Key Vault and automatically deployed via GitHub Actions CI/CD.

**Data Source:** [Disease.sh API](https://disease.sh/v3/covid-19/countries) — Free, no authentication required  
**Data:** Real-time COVID-19 statistics for ~200 countries  
**Schedule:** Runs automatically every day at 6:00 AM (Arizona UTC-7)  
**Security:** Zero hardcoded credentials — all secrets stored in Azure Key Vault  
**Deployment:** Fully automated via GitHub Actions CI/CD  

---

## 🏗️ Architecture

```
🌐 Disease.sh REST API  (Live COVID-19 Data)
           │
           ▼
┌──────────────────────────────────────────────────────┐
│               Azure Data Factory                      │
│                                                      │
│  PL_Master_HealthcarePipeline                        │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│  │RunBronze │──▶│RunSilver │──▶│ RunGold  │         │
│  └──────────┘   └──────────┘   └──────────┘         │
│                                                      │
│  🔐 All credentials read from Azure Key Vault        │
└──────────┬──────────────┬────────────────┬───────────┘
           │              │                │
           ▼              ▼                ▼
    ┌───────────────────────────┐  ┌────────────────┐
    │   Azure Data Lake Gen2    │  │  Azure SQL DB  │
    │   (healthcaredatalakemk)  │  │ (db-healthcare │
    │                           │  │     -gold)     │
    │  📁 bronze/               │  │                │
    │     covid/countries/      │  │ gold_top_      │
    │     raw_2026-04-24.json   │  │ countries      │
    │                           │  │                │
    │  📁 silver/               │  └────────┬───────┘
    │     covid/countries/      │           │
    │     covid_countries_      │           ▼
    │     cleaned.parquet       │  ┌─────────────────┐
    └───────────────────────────┘  │    Power BI     │
                                   │   Dashboard     │
                                   │  📊 Live Data   │
                                   └─────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│              GitHub Actions CI/CD                     │
│                                                      │
│  Push to adf_publish → Auto Deploy to Production    │
│  Stop Triggers → Deploy ARM → Restart Triggers       │
└──────────────────────────────────────────────────────┘
```

---

## 🛠️ Tech Stack

| Component | Service | Purpose |
|---|---|---|
| Orchestration | Azure Data Factory V2 | Pipeline orchestration & scheduling |
| Storage | Azure Data Lake Storage Gen2 | Bronze & Silver layer storage |
| Database | Azure SQL Database (Serverless) | Gold layer — reporting-ready data |
| Transformation | ADF Data Flows (Spark) | Silver layer cleaning & KPI calculation |
| Security | Azure Key Vault | Secrets & credentials management |
| Visualization | Power BI Desktop | Dashboard & reporting |
| Source API | Disease.sh REST API | Live COVID-19 healthcare data |
| Version Control | GitHub | Source control for ADF code |
| CI/CD | GitHub Actions | Automated deployment pipeline |
| Resource Group | rg-healthcare-pipeline | All Azure resources container |

---

## Phase 1: Azure Environment Setup

### Resources Created

| Resource | Name | Location | Purpose |
|---|---|---|---|
| Resource Group | `rg-healthcare-pipeline` | East US | Container for all resources |
| Data Factory | `adf-healthcare-pipelinemk` | East US | Pipeline orchestration |
| Storage Account | `healthcaredatalakemk` | East US | Bronze + Silver data lake |
| SQL Server | `sql-healthcare-mk2` | Central US | Gold layer database server |
| SQL Database | `db-healthcare-gold` | Central US | Gold layer tables |
| Key Vault | `kv-healthcare-mk` | East US | Secrets management |

> **Note:** SQL Server is in Central US due to East US regional capacity limits on free subscriptions.

### Storage Containers
```
healthcaredatalakemk/
├── 📁 bronze/     ← Raw JSON files from API
├── 📁 silver/     ← Cleaned Parquet files
└── 📁 gold/       ← (Aggregated data stored in SQL instead)
```

### SQL Tables
```sql
-- Gold Table 1: All countries with KPIs
CREATE TABLE gold_top_countries (
    country           VARCHAR(100),
    continent         VARCHAR(50),
    cases             BIGINT,
    deaths            BIGINT,
    recovered         BIGINT,
    active            BIGINT,
    critical          BIGINT,
    tests             BIGINT,
    population        BIGINT,
    updated           BIGINT,
    death_rate_pct    FLOAT,
    recovery_rate_pct FLOAT,
    tests_per_million FLOAT,
    ingestion_date    DATE
);

-- Gold Table 2: Continent-level aggregations
CREATE TABLE gold_continent_summary (
    continent         VARCHAR(50),
    total_cases       BIGINT,
    total_deaths      BIGINT,
    total_recovered   BIGINT,
    total_active      BIGINT,
    avg_death_rate    FLOAT,
    avg_recovery_rate FLOAT,
    country_count     INT,
    ingestion_date    DATE
);

-- Helper: Wake up serverless database before pipeline runs
CREATE PROCEDURE WakeUp
AS
BEGIN
    SELECT 1 AS WakeUp
END
```

---

## Phase 2: Linked Services

Four linked services connecting ADF to all data sources and destinations:

### LS_REST_DiseaseSH
```
Type:             REST
Base URL:         https://disease.sh/v3/covid-19/
Authentication:   Anonymous
```

### LS_ADLS_DataLake
```
Type:             Azure Data Lake Storage Gen2
Account:          healthcaredatalakemk
Authentication:   🔐 Azure Key Vault → secret: storage-account-key
```

### LS_AzureSQL_Gold
```
Type:             Azure SQL Database
Server:           sql-healthcare-mk2.database.windows.net
Database:         db-healthcare-gold
Authentication:   SQL (sqladmin)
Password:         🔐 Azure Key Vault → secret: sql-password
```

### LS_KeyVault
```
Type:             Azure Key Vault
Vault:            kv-healthcare-mk
Authentication:   System Assigned Managed Identity
```

---

## Phase 3: Bronze Layer — Raw Ingestion

### Datasets
- **Source:** `DS_REST_CovidCountries` → REST API, relative URL: `countries`
- **Sink:** `DS_Bronze_CovidRaw` → ADLS Gen2, JSON format

### Pipeline: `CopyCovidDataToBronze`
```
Activity:    Copy data
Source:      DS_REST_CovidCountries (GET request)
Sink:        DS_Bronze_CovidRaw
Output:      bronze/covid/countries/raw_2026-04-24.json
Duration:    ~17-26 seconds
```

### Output Sample
```json
[
  {
    "country": "USA",
    "cases": 111820082,
    "deaths": 1219487,
    "recovered": 109381114,
    "active": 1219481,
    "continent": "North America",
    "updated": 1711152314040
  }
]
```

---

## Phase 4: Silver Layer — Cleaning & Transformation

### Data Flow: `DF_Silver_CleanCovidData`

```
BronzeSource → SelectColumns → AddCalculatedColumns → FilterValidRows → SilverSink
```

**AddCalculatedColumns — 4 KPI fields:**
```python
death_rate_pct    = round((deaths / cases) * 100, 2)
recovery_rate_pct = round((recovered / cases) * 100, 2)
tests_per_million = round(tests / (population / 1000000), 0)
ingestion_date    = currentDate()
```

**FilterValidRows:**
```
cases > 0 && !isNull(country) && !isNull(continent)
```

**Output:** `covid_countries_cleaned.parquet` — 14 columns, ~190 rows

### Pipeline: `PL_Silver_TransformCovidData`
```
Duration:    ~1m 41s - 2m 55s
Rows:        ~185-195 countries
```

---

## Phase 5: Gold Layer — Aggregation & SQL

### Pipeline: `PL_Gold_LoadTopCountries`
```
Activities:
  1. WakeUpDatabase        → Stored procedure (wakes serverless DB)
  2. CopyToGoldTopCountries → Silver Parquet → SQL Table

Pre-copy script:   TRUNCATE TABLE gold_top_countries
Duration:          ~21-31 seconds
```

---

## Phase 6: Master Pipeline & Scheduling

### Pipeline: `PL_Master_HealthcarePipeline`

```
[RunBronze] ──▶ [RunSilver] ──▶ [RunGold]
   ~19s            ~2-3min         ~31s
   Total: ~3-4 minutes end-to-end
```

### Schedule Trigger
```
Recurrence:  Every 1 Day at 6:00 AM Arizona (UTC-7)
Status:      Active
```

---

## Phase 7: Power BI Dashboard

### Visuals Built

| Visual | Type | Fields |
|---|---|---|
| Total Cases | KPI Card | SUM(cases) |
| Total Deaths | KPI Card | SUM(deaths) |
| Avg Death Rate | KPI Card | AVG(death_rate_pct) |
| Avg Recovery Rate | KPI Card | AVG(recovery_rate_pct) |
| Cases by Continent | Clustered Bar | continent, SUM(cases) |
| Top Countries | Table | country, cases, deaths, rates |
| Population Share | Treemap | continent, SUM(population) |
| Tests vs Death Rate | Scatter Chart | tests_per_million, death_rate_pct |

### Key Insights
- **USA** leads with 111M+ cases
- **Average global death rate:** 1.39%
- **Average recovery rate:** 72.91%
- Countries with more testing have lower death rates (scatter chart)

---

## Phase 8: Azure Key Vault — Security

### Secrets Stored
| Secret Name | Used By |
|---|---|
| `sql-password` | `LS_AzureSQL_Gold` |
| `sql-server-name` | Reference |
| `storage-account-key` | `LS_ADLS_DataLake` |

### RBAC Roles
| Principal | Role |
|---|---|
| maxwell Kadzashie | Key Vault Secrets Officer |
| adf-healthcare-pipelinemk (Managed Identity) | Key Vault Secrets User |

```
BEFORE:  Password visible in ADF linked services ❌
AFTER:   All secrets in Key Vault, read via Managed Identity ✅
```

---

## Phase 9: CI/CD with GitHub Actions

### GitHub Secrets
| Secret | Value |
|---|---|
| `AZURE_CREDENTIALS` | Service principal JSON |
| `AZURE_RESOURCE_GROUP` | `rg-healthcare-pipeline` |
| `AZURE_DATA_FACTORY_NAME` | `adf-healthcare-pipelinemk` |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |

### Workflow Steps
```
1. ✅ Checkout repository
2. ✅ Login to Azure
3. ✅ Fetch ARM templates from adf_publish/adf-healthcare-pipelinemk/
4. ✅ Stop all ADF triggers
5. ✅ Deploy ARM template (Incremental mode)
6. ✅ Restart all ADF triggers
7. ✅ Print deployment summary
Duration: ~1 minute 8 seconds
```

### Daily Developer Workflow
```
Make changes in ADF → Click Publish → GitHub Actions auto-deploys ✅
Zero manual deployment steps!
```

---

## 🐛 Errors Faced & Fixes

### Error 1 — Data Factory Name Already Taken
```
"The specified resource name 'adf-healthcare-pipeline' is already in use."
```
**Fix:** Added initials suffix → `adf-healthcare-pipelinemk`  
**Lesson:** Always suffix globally-scoped Azure resources with initials or random string.

---

### Error 2 — ADLS Soft Delete Conflict
```
"EndpointUnsupportedAccountFeatures: This endpoint does not support SoftDelete."
```
**Fix:** Storage Account → Data Protection → Disabled soft delete for blobs and containers.

---

### Error 3 — SQL Server Region Capacity
```
"Your subscription does not have access to create a server in East US."
```
**Fix:** Changed region to `Central US`.  
**Lesson:** Free subscriptions have regional capacity limits — try Central US or West US 2.

---

### Error 4 — SQL Server Name Conflict Across Regions
```
"sql-healthcare-mk already exists in eastus, cannot create in westus3."
```
**Fix:** Created `sql-healthcare-mk2` in Central US.

---

### Error 5 — Bronze Source Wrong Document Form
```
"DF-JSON-WrongDocumentForm: Malformed records detected in schema inference."
```
**Fix:** Changed Document form → `Document per line`.  
**Lesson:** Disease.sh returns newline-delimited JSON, not a wrapped JSON array.

---

### Error 6 — Parquet Column Name Special Characters
```
"Column name cannot contain special characters when using Parquet format."
```
**Fix:** Excluded nested `countryInfo` object from mappings (dot notation breaks Parquet).

---

### Error 7 — SelectColumns Type Mismatch (×10)
```
"Data flow expression has error" on SelectColumns
```
**Fix:** Switched to Auto mapping + imported projection schema from BronzeSource.

---

### Error 8 — Gold SQL Table Missing Columns
```
"SqlColumnNameNotExist: Column 'active' does not exist in gold_top_countries"
```
**Fix:** Dropped and recreated table with all 14 columns matching Silver schema.

---

### Error 9 — Wrong SQL Server Name in Linked Service
```
"The remote name could not be resolved: sql-healthcare-server-mk.database.windows.net"
```
**Fix:** Updated server name in `LS_AzureSQL_Gold` to `sql-healthcare-mk2.database.windows.net`.

---

### Error 10 — Source/Sink Datasets Swapped
```
Sink showing the API dataset instead of the SQL/storage dataset
```
**Fix:** Always configure Source first, then Sink to prevent ADF auto-fill swapping them.

---

### Error 11 — Key Vault RBAC Unauthorized
```
"The operation is not allowed by RBAC. You are unauthorized to view these contents."
```
**Fix:** Assigned "Key Vault Secrets Officer" role via IAM → Add role assignment.

---

### Error 12 — Serverless SQL Database Auto-Paused
```
"Database 'db-healthcare-gold' is not currently available."
```
**Root cause:** Serverless tier pauses after 1hr inactivity. Silver takes ~3min → DB pauses.  
**Fix:** Added `WakeUpDatabase` stored procedure activity before copy to force a connection:
```sql
CREATE PROCEDURE WakeUp AS BEGIN SELECT 1 AS WakeUp END
```

---

### Error 13 — CI/CD ARM Templates Not Found
```
"pathspec 'ARMTemplateForFactory.json' did not match any file(s) known to git"
```
**Root cause:** ADF stores ARM templates in subfolder `adf-healthcare-pipelinemk/`  
**Fix:** Updated workflow git checkout path to include subfolder:
```yaml
adf-healthcare-pipelinemk/ARMTemplateForFactory.json
```

---

### Error 14 — CI/CD Cannot Update Enabled Trigger
```
"TriggerEnabledCannotUpdate: Cannot update enabled Trigger; disable it first."
```
### Session 3 additions:
- Azure Monitor diagnostic settings
- Log Analytics workspace  
- Action group with email notifications
- 2 alert rules (pipeline failure + success)
  
**Fix:** Added pre-deploy step to stop triggers and post-deploy step to restart:
```yaml
az datafactory trigger stop --name "$trigger"
# ... deploy ...
az datafactory trigger start --name "$trigger"
```

---

## 📊 Pipeline Performance

| Pipeline | Type | Duration | Output |
|---|---|---|---|
| `CopyCovidDataToBronze` | Copy Data | ~17-26s | 1 JSON file |
| `PL_Silver_TransformCovidData` | Data Flow | ~1m 41s - 2m 55s | 1 Parquet file |
| `PL_Gold_LoadTopCountries` | SP + Copy | ~21-31s | ~190 SQL rows |
| `PL_Master_HealthcarePipeline` | Execute ×3 | ~3-4 min | Full refresh |
| GitHub Actions CI/CD | ARM Deploy | ~1m 8s | Production deploy |

---

## 📁 Repository Structure

```
azure-healthcare-pipeline/
├── 📁 .github/workflows/
│   └── 📄 adf-cicd.yml              ← GitHub Actions CI/CD
├── 📁 pipeline/
│   ├── 📄 CopyCovidDataToBronze.json
│   ├── 📄 PL_Silver_TransformCovidData.json
│   ├── 📄 PL_Gold_LoadTopCountries.json
│   └── 📄 PL_Master_HealthcarePipeline.json
├── 📁 dataset/
│   ├── 📄 DS_REST_CovidCountries.json
│   ├── 📄 DS_Bronze_CovidRaw.json
│   ├── 📄 DS_Silver_CovidCleaned.json
│   └── 📄 DS_Gold_TopCountries.json
├── 📁 linkedService/
│   ├── 📄 LS_REST_DiseaseSH.json
│   ├── 📄 LS_ADLS_DataLake.json      ← Uses Key Vault
│   ├── 📄 LS_AzureSQL_Gold.json      ← Uses Key Vault
│   └── 📄 LS_KeyVault.json
├── 📁 dataflow/
│   └── 📄 DF_Silver_CleanCovidData.json
├── 📁 trigger/
│   └── 📄 Schedule.json
├── 📁 factory/
│   └── 📄 adf-healthcare-pipelinemk.json
├── 📁 powerbi/
│   └── 📄 Healthcare_Pipeline_Dashboard.pbix
└── 📄 README.md
```

---

## 🚀 Next Steps

- [ ] **Azure Monitor & Alerts** — Email notifications on pipeline failure
- [ ] **Power BI Service** — Publish dashboard online with scheduled refresh
- [ ] **Delta Lake** — Time-travel, schema evolution, and MERGE support
- [ ] **Azure Databricks + PySpark** — Enterprise-scale Spark transformations
- [ ] **Update CI/CD** — Upgrade Node.js 20 → 24 in GitHub Actions
- [ ] **Environment separation** — DEV / STAGING / PROD ADF instances

---

## 👤 Author

**Maxwell Kadzashie**  
Built with Azure Data Factory, ADLS Gen2, Azure SQL, Key Vault, Power BI & GitHub Actions  
ThinkAnalytics437 | zionccit-dotcom

---

*Pipeline runs daily at 6:00 AM Arizona time 🌍 | Auto-deployed via GitHub Actions 🤖*
