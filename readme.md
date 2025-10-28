# Azure Data Factory Hybrid Medallion Lakehouse (On-Prem + API + SQL → ADLS)
## End-to-end, production-style ADF pipelines with Self-Hosted IR, watermark-less incremental loads, and Mapping Data Flows (Delta) with CI/CD

A complete, Azure lakehouse that demonstrates real-world ingestion, streaming/batch processing, dimensional modeling, and deployment automation.

## Description

This project is a production-style Azure Data Factory (ADF) build that ingests data from on-prem file shares, REST APIs, and Azure SQL Database into ADLS Gen2. It models the data using the Medallion architecture:

* Bronze: Fast, schema-light landing (CSV/JSON/Parquet/Delta) for raw capture.
* Silver: Standardized Delta with data cleaning, type casting, derivations, and upsert logic keyed by business IDs.
* Gold: Curated business views (joins, aggregations, dense ranking, Top-N) refreshed with overwrite.

Orchestration is handled by a Parent Pipeline that chains the three ingestion paths and kicks off transformations. Hybrid connectivity to the on-prem share is enabled via Self-Hosted Integration Runtime (SHIR). A publish folder is included for ARM-based CI/CD.

<img src="screenshots/azure-resource-group.png" alt="Azure Resource Group with Storage, ADF, Databricks, SQL, Access Connector" />

## Resource setup (Azure)

This solution sits in a single Azure Resource Group, which typically contains:

* Azure Data Factory (the factory that hosts pipelines, datasets, dataflows, linked services).
* Azure Storage (ADLS Gen2) with containers: bronze, silver, gold.
* Azure SQL Database (source for incremental ingestion).
* Self-Hosted Integration Runtime (SHIR) host (VM/Server) that can reach your on-prem file path.

 <img src="Reasource group" alt="Azure Resource Group " />

### Self-Hosted Integration Runtime (SHIR)

<img src="SHIR" />


## Ingestion — ADF (Bronze)

Goal: land raw data reliably and quickly, with minimal coupling to schema, so upstream changes don’t break ingestion.

# Sources covered

* On-prem CSVs → bronze/onprem/... via SHIR + File System linked service
* REST API JSON (GitHub raw) → bronze/github/DimAirport.json via HTTP linked service
* Azure SQL (incremental) → bronze/sql/... (Parquet/Delta) via SQL linked service

# Pipelines & patterns

* onprem_ingestion
    * ForEach over a parameterised file list (e.g., DimPassenger.csv, DimAirline.csv, DimFlight.csv).
    * Copy from on-prem path (via SHIR) → ADLS bronze/onprem/.
    * No column mapping in Bronze (just land the bytes).

<img src="on prem" />

* api_ingustion
    * Web/HTTP + Copy to fetch GitHub raw endpoint and land JSON in bronze/github/.
    * Note: Always use the raw URL to avoid HTML payloads.

<img src="api" />

* sqltodatalake (Incremental without watermark tables)
    * Lookup A: SELECT MAX(booking_date) AS LatestLoad FROM dbo.FactBookings
    * Lookup B: read lastload.json from ADLS (state file).
    * Copy: WHERE booking_date > @lastload AND booking_date <= @latestload → bronze/sql/
    * Write-back: update the state JSON with the new lastload.

<img src="sqltoadls" />

# Why this design
* Idempotent: re-runs won’t duplicate data.
* Resilient: schema-light landing avoids brittle mappings.
* Simple ops: state lives in the lake; no separate watermark table to maintain.


# Bronze Parent (Ingestion Orchestrator)

Goal: single entry point that runs all Bronze ingestion pipelines in a controlled sequence or parallel where safe.

* What it runs
    * On-prem files → `onprem_ingestion`
    * REST API → `api_ingustion`
    * Azure SQL incremental → `sqltodatalake`

<img src="Bronze parent" />

## Standardised Delta (Silver Layer)

Goal: convert raw landings into typed, consistent, analytics-ready Delta while preserving history via upserts.

# Data Flow: DataTransformation
* Column standardisation: rename/select, upper(country), tidy column set for dimensional modelling.
* Business derivations:
    * Gender flags via conditional expressions / regex cleanup.
    * first_name via split/extract from full name

* Type enforcement: cast ticket_cost and other metrics to INT/DECIMAL.

* Upsert with Alter Row:
    * Define business keys (e.g., AirlineID, FlightID, PassengerID).
    * Alter Row conditions drive insert/update semantics into Silver Delta.

* Schema strategy
    * Schema drift: enable only where sources evolve often; otherwise validate strictly to catch regressions
    * Partitioning (optional): by date/tenant to speed downstream joins/aggregates.

<img src="silver" />

## Curated Business Views (Gold Layer)

Goal: deliver business-friendly views with clear value (e.g., Top airlines by revenue) optimised for BI and ad-hoc SQL.

* Data Flow: BusinessView
    * Joins: FactBookings ↔ DimAirline / DimFlight (use left outer to retain fact grain).
    * Aggregations: sum(ticket_cost) by airline_name (and other groupings as needed).
    * Windowing: denseRank() over revenue to avoid rank gaps; keep Top-N (e.g., top-5).
    * Sink strategy: Overwrite on each run (gold is a curated snapshot, not a history store).

* Performance & governance
    * Pre-aggregate enough to keep BI fast; leave drill-downs to Silver.
    * Keep transformation logic deterministic and auditable.

<img src="gold" />

## Parent Pipeline (Orchestration)

Goal: single‑entry pipeline that runs all layers with guardrails.

* Execution order
    *Bronze ingestion: `onprem_ingestion`, `api_ingustion`, `sqltodatalake`
    *Silver transformation: `DataTransformation` (Mapping Data Flow)
    * Gold curation: `BusinessView` (Mapping Data Flow)

* Control & safety
    * Parameters for files list / batch size so Bronze can scale without edits.
    * Idempotent re‑runs (incremental query + watermark JSON).
    * Ready to attach a Schedule trigger (e.g., daily 00:00 UTC) and optional alerting (Logic App / email) on failure.

<img src="parent pipline" />

## Final Word

This project shows the end-to-end path from hybrid sources to curated analytics:
* Hybrid ingestion: on-prem (via SHIR), REST API, and Azure SQL into ADLS Bronze.
* Robust modelling: Silver with typed, cleaned, upserted Delta.
* Business value: Gold snapshot views with rankings/Top-N for instant insights.
* Operate & ship: Parent orchestration, Monitor visibility, ARM publish for CI/CD.

