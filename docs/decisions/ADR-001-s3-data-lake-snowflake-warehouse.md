# ADR-001: Amazon S3 for Data Lake and Snowflake for Data Warehouse

**Date:** 20 November 2026

## Context

The Cryptocurrency Market Data Engineering Platform collects market data from the CoinGecko API. The pipeline needs to retain raw API responses, process them into structured datasets, maintain historical records, and make the data available for analytical queries.

A storage architecture is required that separates raw data retention from analytical processing. It should also support historical backfills, incremental updates, and future expansion without introducing unnecessary infrastructure.

## Decision

Amazon S3 will serve as the primary data lake, while Snowflake will be used as the data warehouse.

The two services have separate responsibilities:

- **Amazon S3:** Stores raw API responses and processed datasets.
- **Snowflake:** Stores structured data used for analytical queries and reporting.
- **Python:** Handles API ingestion and application-level processing.
- **PyArrow:** Converts and processes data into Parquet format.
- **Apache Airflow:** Orchestrates pipeline execution.
- **dbt:** Manages transformations and analytical models in Snowflake.
- **DuckDB:** Supports local data exploration and development.

## Architecture

The storage design follows a Bronze and Silver structure in S3, with analytical models maintained in Snowflake.

### Bronze layer — Raw data

Original API responses are stored in JSON format.

The raw layer retains the source data without applying business transformations. This allows the data to be reprocessed if transformation logic changes or an issue is discovered.

### Silver layer — Processed data

Raw responses are converted into structured datasets.

Processing includes:
- Flattening nested JSON structures where required.
- Standardizing data types and timestamps.
- Handling missing and null values.
- Removing duplicate records.
- Validating required fields and business rules.
- Storing processed datasets in Parquet format.

### Snowflake — Analytical layer

Processed datasets are loaded from S3 into Snowflake.

The warehouse is organized into three logical layers:

- **Staging:** Data loaded from S3 with minimal source-aligned transformations.
- **Intermediate:** Reusable transformations and standardized business entities.
- **Marts:** Analytical fact tables, dimension tables, and reporting datasets.

dbt manages the transformations between these layers.

Business transformations are primarily handled in Snowflake rather than being duplicated in both PyArrow and dbt.

## Data flow

```text
CoinGecko API
      |
      v
Python Ingestion
      |
      v
Amazon S3 - Bronze
      |
      v
Python + PyArrow
      |
      v
Amazon S3 - Silver
      |
      v
Snowflake External Stage
      |
      v
Snowflake Staging
      |
      v
dbt Transformations
      |
      v
Intermediate Models
      |
      v
Analytical Marts
      |
      v
Reporting and Analytics
```

Apache Airflow coordinates the pipeline stages. GitHub Actions handles automated code checks and tests.

## S3 and Snowflake Integration

Snowflake accesses the processed data in S3 through an external stage.

The integration uses:
- An AWS IAM role with restricted S3 permissions.
- A Snowflake storage integration.
- An external stage pointing to the required S3 location.
- `COPY INTO` commands to load data into Snowflake.

Long-lived AWS credentials will not be embedded in application code or committed to the repository.

## Alternatives Considered

### S3 with Athena

Athena can query Parquet files directly from S3 without loading them into a separate warehouse.

This is a simpler option for a lake-centric analytical architecture. However, Snowflake provides a better opportunity to implement and demonstrate warehouse modeling, structured transformations, and dbt workflows.

Athena remains a possible addition if direct querying of S3 becomes necessary.

### Snowflake-only architecture

Storing all data directly in Snowflake would reduce the number of infrastructure components.

However, retaining S3 as an independent storage layer provides a clear separation between source data retention and analytical processing. It also allows raw data to remain available independently of the warehouse.

## Trade-offs

The selected architecture introduces additional configuration and operational responsibilities.

- S3 and Snowflake have separate pricing models.
- Secure access between AWS and Snowflake requires IAM and storage integration configuration.
- Data loading introduces an additional pipeline stage.
- Data schemas must remain consistent between processed S3 datasets and Snowflake staging tables.

The benefits are independent raw data retention, clear transformation boundaries, and a dedicated analytical warehouse.

## Cost Considerations

Cloud resources will be introduced only after local development and testing.

- DuckDB will be used for local analytical queries.
- Airflow will run locally during development.
- Managed Airflow infrastructure will not be provisioned.
- Glue, Athena, Lambda, and other AWS services will only be added when their functionality is required.
- S3 storage, requests, and data transfer costs will be monitored.
- Snowflake compute and storage usage will be reviewed before running warehouse workloads.

## Technology Scope

The initial project is batch-oriented. The following technologies are outside its current scope:

- **Apache Spark:** Not necessary for the expected initial data volume.
- **Kafka:** Intended for a separate streaming project.
- **Apache Iceberg:** Not required for the current S3 and Snowflake design.
- **AWS Lambda:** No specific requirement currently justifies its use.
- **AWS Glue:** Optional; not required for the initial processing workflow.
- **Amazon MWAA:** Local Airflow is sufficient for development.

These technologies can be reconsidered if the requirements or data volume change.

## Future Considerations

The architecture can be extended if additional requirements emerge.

Potential changes include:
- Adding an S3 Gold layer for consumers that need curated datasets outside Snowflake.
- Introducing Athena for direct querying of lake data.
- Using distributed processing if data volume grows significantly.
- Adopting a lakehouse table format if concurrent updates, schema evolution, or time-travel requirements justify it.

