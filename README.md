# Spotify End-to-End Azure Data Engineering Project

Build an end-to-end lakehouse pipeline for Spotify data.  
Source data lands in Azure SQL Database, Azure Data Factory ingests it into ADLS Gen2 (bronze) using incremental loads and watermarking, Databricks (PySpark) transforms it into curated star schema tables (silver), then Lakeflow Declarative Pipelines applies SCD logic and publishes serving tables (gold).

---

## Architecture

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/4faf7d34-6d38-4efe-b3c4-dd2a458eb327" />

### High-level flow:
- Azure SQL Database (source tables)
- Azure Data Factory (incremental ingestion, watermarking, orchestration)
- ADLS Gen2 (bronze, silver, gold containers)
- Azure Databricks + Unity Catalog (PySpark transformations, Auto Loader)
- Lakeflow Declarative Pipelines (SCD to gold, serving layer)
- Logic Apps (pipeline failure alerting)

---

## Tech Stack

- Azure SQL Database
- Azure Data Factory
- Azure Data Lake Storage Gen2 (ADLS)
- Azure Databricks (PySpark)
- Unity Catalog (metastore, external locations)
- Lakeflow Declarative Pipelines (gold layer)
- Azure Logic Apps (alerts)
- GitHub (version control, PR workflow)

---

## Data Lake Layout

- bronze/
  - Raw ingested Parquet files from Azure SQL
  - cdc/ folders per table for watermark JSON
- silver/
  - Cleaned, standardized Delta tables (star schema ready)
  - Auto Loader checkpoints and schema locations
- gold/
  - Serving tables with SCD logic (dimensions + fact)

<img width="949" height="432" alt="image" src="https://github.com/user-attachments/assets/ee0cb336-3ed6-477e-b783-cf3eac653efb" />

---

## Bronze Layer (ADF Ingestion)

Goal: ingest data from Azure SQL to ADLS Gen2 as Parquet using incremental loading.

What you built:
- Parameterized pipeline to ingest any table:
  - schema
  - table
  - cdc_col
  - from_date (optional backfill)
- Watermarking with JSON files stored in ADLS:
  - `empty.json`
  - `cdc.json` with initial value like `{"cdc":"1900-01-01"}`
- Query pattern:
  - Load only records where `cdc_col` is greater than the stored watermark
  - If `from_date` is provided, backfill from that date instead
- File naming strategy:
  - `table_currentTimestamp.parquet` so silver can identify latest loads
- MAX(cdc_col) step after ingestion:
  - fetch `MAX(cdc_col)` from Azure SQL
  - overwrite the watermark JSON with the latest value
- ForEach looping:
  - pipeline reads a `loop_input` JSON to ingest multiple tables
- Cleanup:
  - delete empty parquet file if no incremental rows are returned

<img width="847" height="616" alt="image" src="https://github.com/user-attachments/assets/0b33f0ad-977e-4562-a057-9e16818c1ed3" />


### Watermark Files

<img width="959" height="665" alt="image" src="https://github.com/user-attachments/assets/a68118b1-ed1a-4907-a92b-b084cd99acc8" />

---

## Alerts (Logic Apps + ADF Web Activity)

Goal: notify you when the pipeline fails or is skipped.

What you built:
- Logic App trigger: "When an HTTP request is received"
- Email action: Outlook "Send an email (V2)"
- ADF Web Activity:
  - connected on fail and on skip
  - sends a JSON payload using ADF system variables

<img width="221" height="337" alt="image" src="https://github.com/user-attachments/assets/fd42607d-4500-4107-8cda-a7593ee42231" />


<img width="848" height="412" alt="image" src="https://github.com/user-attachments/assets/f50bda56-c4b8-4394-88c7-319a0a592808" />

---

## Silver Layer (Databricks + PySpark)

Goal: read bronze Parquet incrementally and build curated Delta tables for analytics.

What you built:
- Access to ADLS using:
  - Access Connector (managed identity)
  - Unity Catalog metastore
  - External locations for bronze, silver, gold
- Auto Loader ingestion:
  - checkpointing (idempotent loads)
  - schema location (schema evolution support)
- Transformations:
  - data type casting
  - null handling
  - standard column naming
  - deduplication for dimension keys
  - optional handling of `_rescued_data` if schema evolves

<img width="948" height="671" alt="image" src="https://github.com/user-attachments/assets/8d196961-1982-49e0-b290-50723d32252a" />


<img width="841" height="760" alt="image" src="https://github.com/user-attachments/assets/aa846d20-f914-41ec-8def-4e502e3cfa03" />


### Utilities (Reusable PySpark Functions)

You created a utilities module to reuse logic across notebooks:
- deduplication by primary key
- dropping technical columns like `_rescued_data` (when safe)
- common cleaning steps

<img width="563" height="257" alt="image" src="https://github.com/user-attachments/assets/6283b5b3-06b0-43da-b45f-c7ee90ab0f37" />

---

## Gold Layer (Lakeflow Declarative Pipelines)

Goal: serve analytics-ready tables with SCD logic.

Gold tables:
- Dimensions:
  - dimdate
  - dimuser
  - dimtrack
- Fact:
  - factstream
- Staging tables:
  - dimdate_stg
  - dimuser_stg
  - dimtrack_stg
  - factstream_stg

SCD approach:
- Dimensions use SCD (Type 1 or Type 2 depending on your implementation)
- Facts are typically append-only unless you have event-level corrections

<img width="656" height="687" alt="image" src="https://github.com/user-attachments/assets/265cb003-10fd-445e-87be-80d42b49b2ee" />

---

## Data Model

Star schema (serving layer):
- factstream
  - references dimuser, dimtrack, dimdate
- dimuser
- dimtrack
- dimdate

<img width="662" height="553" alt="image" src="https://github.com/user-attachments/assets/2c286a12-60a1-4185-ae9e-347fbb7db0a5" />

---

## How to Run

1) Source setup
- Create Azure SQL Database
- Run the source SQL script to create and populate tables

2) Bronze ingestion
- Run ADF pipeline
- Confirm Parquet files land in `bronze/<TableName>/`

3) Silver transformations
- Run Databricks notebooks or jobs for silver ingestion
- Confirm Delta tables are created in `spotify_cata.silver`

4) Gold serving
- Run Lakeflow Declarative Pipeline
- Confirm gold tables exist and update correctly

---

## Notes

- Watermarking is stored in ADLS per table, so each table increments independently.
- Auto Loader checkpoints ensure you only process new files in silver.
- Git workflow:
  - work in dev branch
  - create PR to main
  - publish ADF changes after merge

---

## Future Improvements

- Add data quality checks (null keys, duplicate keys, referential integrity)
- Add unit tests for PySpark utilities
- Add job scheduling for Databricks and Lakeflow pipelines
- Add monitoring dashboards (Log Analytics, Azure Monitor)

---

## Credits

This project follows a learning path inspired by Ansh Lamba’s Spotify Azure project and tutorials.
