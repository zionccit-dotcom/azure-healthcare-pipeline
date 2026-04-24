# azure-healthcare-pipeline
Azure ADF Medallion Architecture Healthcare Pipeline
# Azure Healthcare Data Pipeline
### Medallion Architecture (Bronze → Silver → Gold) with Power BI

![Azure](https://img.shields.io/badge/Azure-Data%20Factory-blue?logo=microsoftazure)
![PowerBI](https://img.shields.io/badge/Power%20BI-Dashboard-yellow?logo=powerbi)
![Status](https://img.shields.io/badge/Pipeline-Live%20%26%20Scheduled-brightgreen)

---

##  Table of Contents
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
- [Errors Faced & Fixes](#-errors-faced--fixes)
- [Pipeline Performance](#pipeline-performance)
- [Next Steps](#next-steps)

---

##  Project Overview

A fully automated, end-to-end healthcare data pipeline built on **Azure Data Factory** using the **Medallion Architecture** pattern. The pipeline ingests live COVID-19 statistics from a public REST API, cleans and transforms the data through three layers, stores aggregated results in Azure SQL Database, and visualizes insights in Power BI.

**Data Source:** [Disease.sh API](https://disease.sh/v3/covid-19/countries) — Free, no authentication required  
**Data:** Real-time COVID-19 statistics for ~200 countries  
**Schedule:** Runs automatically every day at 6:00 AM (Arizona UTC-7)  

---

##  Architecture

```
Disease.sh REST API  (Live COVID-19 Data)
           │
           ▼
┌─────────────────────────────────────────────────────┐
│              Azure Data Factory                      │
│                                                     │
│  PL_Master_HealthcarePipeline                       │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐        │
│  │RunBronze │──▶│RunSilver │──▶│ RunGold  │        │
│  └──────────┘   └──────────┘   └──────────┘        │
└──────────┬──────────────┬───────────────┬───────────┘
           │              │               │
           ▼              ▼               ▼
    ┌─────────────────────────────┐  ┌────────────────┐
    │   Azure Data Lake Gen2      │  │  Azure SQL DB  │
    │   (healthcaredatalakemk)    │  │ (db-healthcare │
    │                             │  │     -gold)     │
    │  bronze/                 │  │                │
    │     covid/countries/        │  │ gold_top_      │
    │     raw_2026-04-24.json     │  │ countries      │
    │                             │  │                │
    │   silver/                 │  │ gold_continent │
    │     covid/countries/        │  │ _summary       │
    │     covid_countries_        │  │                │
    │     cleaned.parquet         │  └────────┬───────┘
    └─────────────────────────────┘           │
                                              ▼
                                    ┌─────────────────┐
                                    │    Power BI     │
                                    │   Dashboard     │
                                    │   Live Data   │
                                    └─────────────────┘
```

---

##  Tech Stack

| Component | Service | Purpose |
|---|---|---|
| Orchestration | Azure Data Factory V2 | Pipeline orchestration & scheduling |
| Storage | Azure Data Lake Storage Gen2 | Bronze & Silver layer storage |
| Database | Azure SQL Database (Serverless) | Gold layer — reporting-ready data |
| Transformation | ADF Data Flows (Spark) | Silver layer cleaning & KPI calculation |
| Visualization | Power BI Desktop | Dashboard & reporting |
| Source API | Disease.sh REST API | Live COVID-19 healthcare data |
| Resource Group | rg-healthcare-pipeline | All Azure resources container |

---

## Phase 1: Azure Environment Setup

### Resources Created

| Resource | Name | Location |
|---|---|---|
| Resource Group | `rg-healthcare-pipeline` | East US |
| Data Factory | `adf-healthcare-pipelinemk` | East US |
| Storage Account | `healthcaredatalakemk` | East US |
| SQL Server | `sql-healthcare-mk2` | Central US |
| SQL Database | `db-healthcare-gold` | Central US |

### Storage Containers Created
```
healthcaredatalakemk/
├──  bronze/     ← Raw JSON files
├──  silver/     ← Cleaned Parquet files
└──  gold/       ← (Aggregated data in SQL instead)
```

### SQL Tables Created
```sql
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
```

---

## Phase 2: Linked Services

Three linked services created to connect ADF to all data sources and destinations:

### LS_REST_DiseaseSH
```
Type:             REST
Base URL:         https://disease.sh/v3/covid-19/
Authentication:   Anonymous (no API key needed)
```

### LS_ADLS_DataLake
```
Type:             Azure Data Lake Storage Gen2
Account:          healthcaredatalakemk
Authentication:   Account key
```

### LS_AzureSQL_Gold
```
Type:             Azure SQL Database
Server:           sql-healthcare-mk2.database.windows.net
Database:         db-healthcare-gold
Authentication:   SQL (sqladmin)
```

---

## Phase 3: Bronze Layer — Raw Ingestion

### Datasets
- **Source:** `DS_REST_CovidCountries` → REST API, relative URL: `countries`
- **Sink:** `DS_Bronze_CovidRaw` → ADLS Gen2, JSON format, path: `bronze/covid/countries/`

### Pipeline: `CopyCovidDataToBronze`
```
Activity:    Copy data
Source:      DS_REST_CovidCountries (GET request)
Sink:        DS_Bronze_CovidRaw
Output:      bronze/covid/countries/raw_2026-04-24.json
Duration:    ~17-26 seconds
```

### Output File Structure
```json
[
  {
    "country": "USA",
    "cases": 111820082,
    "deaths": 1219487,
    "recovered": 109381114,
    "active": 1219481,
    "critical": 1059,
    "tests": 1229065533,
    "population": 334805269,
    "continent": "North America",
    "updated": 1711152314040,
    "countryInfo": { "lat": 38, "long": -97, ... }
  },
  ...~200 countries
]
```

---

## Phase 4: Silver Layer — Cleaning & Transformation

### Dataset
- **Source:** `DS_Bronze_CovidRaw` → Bronze JSON files
- **Sink:** `DS_Silver_CovidCleaned` → ADLS Gen2, Parquet format

### Data Flow: `DF_Silver_CleanCovidData`

```
BronzeSource → SelectColumns → AddCalculatedColumns → FilterValidRows → SilverSink
```

#### Transformations Applied

**SelectColumns** — Kept only 10 core fields:
```
country, cases, deaths, recovered, active,
critical, tests, population, continent, updated
```
> Excluded nested object `countryInfo` (caused Parquet errors with dot notation)

**AddCalculatedColumns** — 4 new KPI fields:
```python
death_rate_pct    = round((deaths / cases) * 100, 2)
recovery_rate_pct = round((recovered / cases) * 100, 2)
tests_per_million = round(tests / (population / 1000000), 0)
ingestion_date    = currentDate()
```

**FilterValidRows** — Data quality filter:
```
cases > 0 && !isNull(country) && !isNull(continent)
```

**SilverSink** — Output settings:
```
Format:           Parquet
File name option: Output to single file
Output file:      covid_countries_cleaned.parquet
Final columns:    14 total
```

### Pipeline: `PL_Silver_TransformCovidData`
```
Activity:    Data Flow (DF_Silver_CleanCovidData)
Runtime:     debugpool-8Cores-General (East US)
Duration:    ~1 minute 41 seconds
Rows output: ~185-195 countries
```

---

## Phase 5: Gold Layer — Aggregation & SQL

### Dataset
- **Source:** `DS_Silver_CovidCleaned` → Silver Parquet (wildcard: `*.parquet`)
- **Sink:** `DS_Gold_TopCountries` → Azure SQL `dbo.gold_top_countries`

### Pipeline: `PL_Gold_LoadTopCountries`
```
Activity:          Copy data (CopyToGoldTopCountries)
Source:            DS_Silver_CovidCleaned (wildcard *.parquet)
Sink:              DS_Gold_TopCountries
Pre-copy script:   TRUNCATE TABLE gold_top_countries
Duration:          ~21 seconds
Rows loaded:       ~185-195 countries
```

### Sample Gold Layer Query
```sql
SELECT TOP 10
    country,
    continent,
    cases,
    deaths,
    death_rate_pct,
    recovery_rate_pct,
    tests_per_million
FROM gold_top_countries
ORDER BY cases DESC;
```

---

## Phase 6: Master Pipeline & Scheduling

### Pipeline: `PL_Master_HealthcarePipeline`

```
[RunBronze] ──▶ [RunSilver] ──▶ [RunGold]
   17s             ~2min           21s
```

| Activity | Invokes Pipeline | Duration |
|---|---|---|
| RunBronze | `CopyCovidDataToBronze` | ~17s |
| RunSilver | `PL_Silver_TransformCovidData` | ~2min |
| RunGold | `PL_Gold_LoadTopCountries` | ~21s |

### Schedule Trigger
```
Name:        Schedule
Type:        ScheduleTrigger
Recurrence:  Every 1 Day
Time:        6:00 AM
Timezone:    Arizona (UTC-7)
Status:      Started (Active)
```

---

## Phase 7: Power BI Dashboard

### Connection
```
Server:    sql-healthcare-mk2.database.windows.net
Database:  db-healthcare-gold
Table:     gold_top_countries
Mode:      Import
```

### Visuals Built

| Visual | Type | Fields Used |
|---|---|---|
| Total Cases | KPI Card | SUM(cases) |
| Total Deaths | KPI Card | SUM(deaths) |
| Avg Death Rate | KPI Card | AVG(death_rate_pct) |
| Avg Recovery Rate | KPI Card | AVG(recovery_rate_pct) |
| Cases by Continent | Clustered Bar Chart | continent, SUM(cases) |
| Top Countries | Table | country, cases, deaths, death_rate_pct, recovery_rate_pct, tests_per_million |
| Population by Continent | Treemap | continent, SUM(population) |
| Tests vs Death Rate | Scatter Chart | tests_per_million, death_rate_pct, country, continent |

### Dashboard Insights
- **USA** leads with 111M+ cases
- **Average death rate:** 1.39% globally
- **Average recovery rate:** 72.91%
- Countries with **more testing** tend to have **lower death rates** (visible in scatter chart)
- **Asia** has the largest population share (treemap)

---

##  Errors Faced & Fixes

### Error 1 — Data Factory Name Already Taken
```
Error: "The specified resource name 'adf-healthcare-pipeline' is already in use.
       Resource names must be globally unique."
```
**Fix:** Renamed to `adf-healthcare-pipelinemk` (added personal initials).  
**Lesson:** Always append initials or random suffix to globally-scoped Azure resources.

---

### Error 2 — ADLS Soft Delete Conflict
```
Error: "ADLS Gen2 operation failed. ErrorCode: EndpointUnsupportedAccountFeatures.
       Message: This endpoint does not support BlobStorageEvents or SoftDelete."
```
**Fix:**  
1. Went to Storage Account → Data Management → **Data Protection**
2. Disabled **"Enable soft delete for blobs"**
3. Disabled **"Enable soft delete for containers"**
4. Clicked Save → Retested connection → ✅

---

### Error 3 — SQL Server Region Capacity
```
Error: "Your subscription does not have access to create a server
       in the selected region (East US)."
```
**Fix:** Changed SQL Server region from `East US` to `Central US`.  
**Lesson:** Free/trial Azure subscriptions have regional capacity limits. Try `Central US` or `West US 2` as fallbacks.

---

### Error 4 — SQL Server Name Conflict
```
Error: "The resource 'sql-healthcare-mk' already exists in location 'eastus'.
       A resource with the same name cannot be created in location 'westus3'."
```
**Fix:** Created new server named `sql-healthcare-mk2` in `Central US`.

---

### Error 5 — Bronze Source Schema Import Failed
```
Error: "DF-JSON-WrongDocumentForm: Malformed records detected in schema inference.
       Parse Mode: FAILFAST."
```
**Fix:** Changed **Document form** in BronzeSource → Source Options from  
`Array of documents` → `Document per line`  
**Lesson:** Disease.sh API returns newline-delimited JSON, not a JSON array.

---

### Error 6 — Parquet Column Name with Special Characters
```
Error: "Column name cannot contain special characters or spaces
       when using Parquet format in 'SilverSink'"
```
**Root cause:** `countryInfo` is a nested JSON object with sub-fields like  
`countryInfo.lat`, `countryInfo.long` — the dot notation breaks Parquet.  
**Fix:** Turned off Auto Mapping in SilverSink and manually mapped only  
the 14 clean columns, explicitly excluding `countryInfo`.

---

### Error 7 — SelectColumns Type Mismatch Errors (×10)
```
Error: "Data flow expression has error" on SelectColumns (10 errors)
```
**Root cause:** Manual column mappings couldn't resolve types before  
schema was imported in Projection tab.  
**Fix:** Switched SelectColumns to **Auto mapping**, then imported  
projection schema in BronzeSource → Projection tab.

---

### Error 8 — Gold Pipeline Column Not Found
```
Error: "SqlColumnNameNotExist: Column 'active' does not exist
       in the target table '[dbo].[gold_top_countries]'"
```
**Root cause:** Original SQL table creation script was missing several  
columns that exist in the Silver Parquet file.  
**Fix:** Dropped and recreated the table with all 14 columns:
```sql
DROP TABLE IF EXISTS gold_top_countries;
CREATE TABLE gold_top_countries (
    country VARCHAR(100), continent VARCHAR(50),
    cases BIGINT, deaths BIGINT, recovered BIGINT,
    active BIGINT, critical BIGINT, tests BIGINT,
    population BIGINT, updated BIGINT,
    death_rate_pct FLOAT, recovery_rate_pct FLOAT,
    tests_per_million FLOAT, ingestion_date DATE
);
```

---

### Error 9 — ADF Linked Service Wrong Server Name
```
Error: "The remote name could not be resolved:
       'sql-healthcare-server-mk.database.windows.net'"
```
**Root cause:** Original linked service `LS_AzureSQL_Gold` had  
old server name from Phase 1 guide.  
**Fix:** Updated server in linked service to:  
`sql-healthcare-mk2.database.windows.net`

---

### Error 10 — Source/Sink Dataset Swapped
```
Symptom: Sink showing DS_REST_CovidCountries (the API dataset)
         instead of the storage/SQL dataset
```
**Root cause:** ADF auto-fills the last selected dataset into  
whichever tab you configure first.  
**Fix:** Always configure **Source first**, then **Sink**.  
Delete wrong dataset from sink and re-select correct one.

---

## 📊 Pipeline Performance

| Pipeline | Activity Type | Duration | Output |
|---|---|---|---|
| `CopyCovidDataToBronze` | Copy Data | ~17-26s | 1 JSON file |
| `PL_Silver_TransformCovidData` | Data Flow (Spark) | ~1m 41s | 1 Parquet file |
| `PL_Gold_LoadTopCountries` | Copy Data | ~21s | ~190 SQL rows |
| `PL_Master_HealthcarePipeline` | Execute Pipeline ×3 | ~3-4 min total | Full refresh |

---

## Next Steps

- [ ] **Azure Databricks** — Replace ADF Data Flows with PySpark notebooks for scalability
- [ ] **Delta Lake** — Add time-travel, schema evolution, and upsert (MERGE) support
- [ ] **Azure Key Vault** — Store connection strings and passwords securely
- [ ] **CI/CD with GitHub Actions** — Auto-deploy ADF changes on git push
- [ ] **Azure Monitor Alerts** — Email notifications on pipeline failure
- [ ] **Historical Trending** — Use `/v3/covid-19/historical` endpoint for time-series analysis
- [ ] **Power BI Service** — Publish dashboard online with scheduled refresh
- [ ] **Slowly Changing Dimensions** — Track how country stats change over time

---

## 📁 Repository Structure

```
azure-healthcare-pipeline/
├──  pipeline/
│   ├── CopyCovidDataToBronze.json
│   ├── PL_Silver_TransformCovidData.json
│   ├── PL_Gold_LoadTopCountries.json
│   └── PL_Master_HealthcarePipeline.json
├──  dataset/
│   ├── DS_REST_CovidCountries.json
│   ├── DS_Bronze_CovidRaw.json
│   ├── DS_Silver_CovidCleaned.json
│   └── DS_Gold_TopCountries.json
├──  linkedService/
│   ├── LS_REST_DiseaseSH.json
│   ├── LS_ADLS_DataLake.json
│   └── LS_AzureSQL_Gold.json
├──  dataflow/
│   └── DF_Silver_CleanCovidData.json
├──  trigger/
│   └── Schedule.json
├──  powerbi/
│   └── Healthcare_Pipeline_Dashboard.pbix
└──  README.md
```

---

##  Author

**Maxwell Kadzashie**  
Built with Azure Data Factory, ADLS Gen2, Azure SQL, and Power BI  
ThinkAnalytics437

---

*Pipeline runs daily at 6:00 AM Arizona time — data is always fresh! *
