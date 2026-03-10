# Azure End-to-End Data Engineering Project

An end-to-end data engineering solution that migrates an on-premises SQL Server database (AdventureWorksLT2022) to the Azure cloud using a **Medallion Lakehouse architecture** (Bronze → Silver → Gold). The pipeline ingests raw data, applies staged transformations, and delivers analytics-ready output — all orchestrated through a single automated pipeline.

## Architecture

**Source** → Azure Data Factory → **Bronze** (ADLS Gen2) → Azure Databricks → **Silver** → Azure Databricks → **Gold** → Power BI

| Layer | Purpose | Format |
|-------|---------|--------|
| Bronze | Raw, unmodified copy of source tables | Parquet |
| Silver | Light transformation (datetime → date conversion) | Delta |
| Gold | Fully curated (PascalCase → snake_case column renaming) | Delta |

## Tech Stack

- **Azure Data Factory** — Dynamic pipeline with Lookup + ForEach to ingest all 10 SalesLT tables in a single run
- **Azure Data Lake Storage Gen2** — Three-container lakehouse (bronze / silver / gold)
- **Azure Databricks (PySpark)** — Two-stage transformation notebooks triggered by ADF
- **Azure Key Vault** — Secrets management for SQL credentials and Databricks access tokens
- **Microsoft Entra ID** — Identity, access management, and RBAC
- **Self-Hosted Integration Runtime** — Bridges ADF to the on-premises SQL Server
- **Unity Catalog External Locations** — Databricks-to-ADLS authentication (replaced deprecated credential passthrough)

## Pipeline Design

The ADF pipeline runs four activities in sequence:

1. **Lookup** — Queries `sys.tables` / `sys.schemas` to dynamically discover all SalesLT tables
2. **ForEach** — Iterates over the Lookup output and copies each table to the Bronze layer as Parquet
3. **Bronze to Silver (Databricks Notebook)** — Converts all datetime columns to date-only format
4. **Silver to Gold (Databricks Notebook)** — Renames columns from PascalCase to snake_case

Adding a new row to any source table and triggering the pipeline propagates the change through all three layers automatically.

## Key Implementation Details

- **Dynamic ingestion**: A single parameterized pipeline handles all 10 tables — no hardcoded table names
- **Unity Catalog integration**: ADLS access uses an Access Connector with managed identity and External Locations (credential passthrough was deprecated on Databricks Runtime 17.3+)
- **Delta Lake**: Silver and Gold layers use Delta format for ACID transactions and version tracking
- **Security**: All credentials stored in Key Vault; ADF authenticates via system-assigned managed identity; RBAC enforced across all resources

## Project Structure

```
├── notebooks/
│   ├── bronze_to_silver.py     # Level 1 transformation
│   └── silver_to_gold.py       # Level 2 transformation
├── sql/
│   └── get_schema.sql          # Dynamic table discovery query
├── pipeline/                   # ADF pipeline definitions (JSON)
└── README.md
```
