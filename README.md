# Retailplus-azure-pipeline

### RetailPulse is an end-to-end data engineering pipeline on Azure that turns raw AdventureWorks retail sales data (2015–2017, ~56K transactions, 18K customers, 293 products) into a BI-ready analytics layer — built with a medallion (Bronze/Silver/Gold) architecture, a star schema, and a customer RFM segmentation mart.

### Infrastructure is defined as code (Bicep), pipeline logic is validated automatically on every push (GitHub Actions), and secrets are never hardcoded (Azure Key Vault).

## Contents
. Architecture

. Results

. Tech stack

. Repo structure

. Quickstart — validate locally in 2 minutes

. Full deployment to Azure

. Design decisions

. What I'd improve next

## Architecture

### Pointwise breakdown:

. Source — Raw AdventureWorks CSVs (customers, products, categories, sales 2015–2017, returns, territories, calendar).

. Ingestion — Azure Data Factory copies raw files from a landing container into ADLS Gen2 raw/ zone; orchestrates the whole pipeline end to end.

. Bronze layer — Databricks/PySpark notebook reads raw files, enforces schema, adds ingestion metadata (_ingested_at, _source_file), writes as Delta tables. No business 

. logic here — just a faithful raw copy.

. Silver layer — Cleans and conforms Bronze data: fixes the ISO-8859-1 encoding issue, deduplicates, casts types, standardizes text, unions the 3 years of sales files, filters invalid rows (nulls, negative quantities).

. Gold layer — Models Silver into a star schema (fact_sales, dim_customer, dim_product, dim_territory, dim_calendar, fact_returns) plus two business marts: 

. monthly_sales_summary and customer_rfm_segments (Recency/Frequency/Monetary scoring → Champions/Loyal/At Risk/Lost).

. Serving layer — Azure Synapse Serverless SQL creates external tables directly over the Gold Delta files — no data duplication, pay-per-query only.

. Secrets management — Azure Key Vault stores the storage account key and Databricks token; nothing hardcoded in ADF or notebooks.

. Infrastructure as Code — Bicep template provisions all of the above (storage, ADF, Databricks, Synapse, Key Vault) with managed identities pre-wired, deployable via one script.

. CI/CD — GitHub Actions validates the pipeline logic and Bicep syntax on every push; Azure DevOps documented as the path to deploy ADF/Databricks changes via pull request.
Results

. Pipeline logic has been run end-to-end against the real dataset in this repo (notebooks/local_validation_pandas.py — see Quickstart):

### Metric	Value

. Sales transactions processed	56,046
. Customers	18,148 (17,416 with completed orders)
. Total revenue	$24,914,583.66
. Total profit	$10,457,736.53
. Customer segments (RFM)	Champions: 2,589 · Loyal: 4,859 · At Risk: 5,966 · Lost/Low-value: 4,002
. Tech stack

### Layer	Tool

Orchestration	Azure Data Factory
Storage	Azure Data Lake Storage Gen2
Transformation	Azure Databricks (PySpark), Delta Lake
Serving	Azure Synapse Analytics (Serverless SQL)
Secrets	Azure Key Vault
Infrastructure as Code	Bicep
CI	GitHub Actions

### Repo structure

Pointwise breakdown:


. PUSH_TO_GITHUB.md — Instructions to push this pre-built repo to your own GitHub.
. notebooks/ — 01_bronze_ingestion.py, 02_silver_transformation.py, 03_gold_aggregation.py (Databricks/PySpark), plus local_validation_pandas.py to prove the logic without a cluster.
. sql/ — 01_synapse_external_tables.sql (DDL) and 02_analytical_queries.sql (revenue, return rate, RFM value, MoM growth queries).
. adf/ — pipelines/ (importable master pipeline JSON), linkedServices/ and datasets/ (connection templates).
. infra/ — bicep/main.bicep (IaC) and deploy.sh (one-command provisioning script).
. architecture/ — architecture.svg, the diagram embedded in the README.
. github/workflows/validate.yml — CI pipeline that runs on every push.
. data/bronze/ and data/gold/ — sample source CSVs and proof-of-run output.
. Quickstart — validate locally in 2 minutes

No Azure account needed to prove the transformation logic works:

### bash
git clone https://github.com/YOUR_USERNAME/retailpulse-azure-pipeline.git
cd retailpulse-azure-pipeline
pip install -r requirements.txt
python3 notebooks/local_validation_pandas.py

### This mirrors the Bronze → Silver → Gold logic in pandas against the real CSVs in data/bronze/ and writes Gold-layer CSVs to data/gold/ — the same numbers reported in Results above.

### Full deployment to Azure
Prerequisites: an Azure subscription, Azure CLI installed, az login run.
Provision infrastructure:
bash
   cd infra
   ./deploy.sh

This creates a resource group and deploys bicep/main.bicep — a storage account (ADLS Gen2 with raw/bronze/silver/gold/landing containers), Azure Data Factory, Azure Databricks workspace, Azure Synapse workspace, and Key Vault, with managed identities wired up so ADF and Synapse can read/write the lake without hardcoded credentials. 3. Upload source data: upload data/bronze/*.csv to the landing container. 4. Azure Data Factory: open ADF Studio → import linked services from adf/linkedServices/LS_templates.json (fill in your real resource names), datasets from adf/datasets/DS_templates.json, and the pipeline from adf/pipelines/PL_AdventureWorks_Master.json. 5. Azure Databricks: attach a small cluster, import the three notebooks from notebooks/, update the storage_account widget default to your deployed account name. 6. Run it: trigger PL_AdventureWorks_Master in ADF — it copies files, then runs Bronze → Silver → Gold notebooks in sequence with retry policy and a failure webhook. 7. Azure Synapse: open a Serverless SQL script, run sql/01_synapse_external_tables.sql, then explore with sql/02_analytical_queries.sql. 8. Power BI: see powerbi/POWERBI_SETUP.md for the data model and ready-to-paste DAX measures.


### Design decisions — why this way

Medallion architecture (Bronze/Silver/Gold) — separates raw data from business logic, so Silver/Gold can be reprocessed without re-pulling from source, and lets you isolate whether an issue is a source problem or a transform bug by checking which layer diverges.
Delta Lake over plain Parquet — gives ACID transactions, schema enforcement, and time travel, which matters once more than one pipeline run writes to the same table.
Serverless Synapse SQL instead of Dedicated — this dataset doesn't need standing compute; serverless only charges per query. sql/01_synapse_external_tables.sql documents how to swap to a Dedicated pool with COPY INTO if latency/volume required it.
RFM customer segmentation — turns raw transactions into a business-usable output (who's a "Champion" vs "At Risk") instead of stopping at clean tables.
Infrastructure as code (Bicep) + CI (GitHub Actions) — the environment and pipeline logic are both reproducible and reviewed through pull requests, not hand-clicked in the Azure portal.
Real encoding bug caught and fixed: AdventureWorks_Customers.csv is ISO-8859-1 encoded (contains an accented name, "ANDRÉS"), not UTF-8 — reading it as UTF-8 throws a UnicodeDecodeError. Handled explicitly in both the Databricks ingestion notebook and the local validation script.
What I'd improve next
Auto Loader / streaming ingestion instead of static file lists, for true incremental loads
SCD Type 2 on dim_customer and dim_product to track historical changes
Data quality checks (Great Expectations or Databricks-native) between layers
Refactor notebook cells into testable Python functions/package, with unit tests in CI
Move the Synapse SQL admin password in infra/bicep/main.bicep to a Key Vault reference instead of an inline parameter (flagged in the template comments)
For recruiters / resume material


