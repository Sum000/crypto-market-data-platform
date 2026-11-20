# Physical Data Model

## 1. Overview

This document defines the physical data model for the Cryptocurrency Market Data Engineering Platform.

It describes the datasets, their structure, data types, business keys, storage organization, and data quality requirements across the Amazon S3 data lake and Snowflake data warehouse.

The physical model is based on the following design decisions:

- Amazon S3 serves as the primary data lake.
- Snowflake serves as the analytical data warehouse.
- Bronze stores original API responses in JSON format.
- Silver stores cleaned and standardized datasets in Parquet format.
- Python and PyArrow handle ingestion and lake-level processing.
- dbt manages transformations within Snowflake.
- DuckDB supports local analytical development.

The schemas in this document are preliminary. Final column definitions will be validated against actual CoinGecko API responses before implementation.

## 2. Data Modeling Conventions

### 2.1 Naming conventions

| Element | Convention | Example |
|---|---|---|
| Dataset names | snake_case | `market_snapshot` |
| Column names | snake_case | `market_cap` |
| Primary identifiers | `_id` suffix | `coin_id` |
| Date columns | `_date` suffix | `snapshot_date` |
| Timestamp columns | `_at` suffix | `ingested_at` |
| Monetary values | Currency suffix where applicable | `market_cap_usd` |
| Boolean columns | Descriptive boolean names | `is_active` |

### 2.2 Timestamp conventions

- UTC is the standard timezone across all datasets.
- Source timestamps are retained wherever available.
- Unix timestamps returned in milliseconds are converted into UTC timestamps during processing.
- Ingestion timestamps are maintained separately from source observation timestamps.
- Snowflake `TIMESTAMP_NTZ` columns represent UTC values by convention.

The following timestamps have distinct meanings:

| Timestamp | Meaning |
|---|---|
| `observation_timestamp` | Time represented by a historical market observation |
| `snapshot_at` | Actual time a market snapshot was collected |
| `source_last_updated_at` | Last update timestamp supplied by CoinGecko |
| `ingested_at` | Time the API response was retrieved |
| `processed_at` | Time the data was processed |

### 2.3 Numeric data types

Market prices, market capitalization, trading volume, and supply values require sufficient decimal precision.

The proposed representations are:

- Parquet: DECIMAL with precision 38 and scale 18, where supported by the source values.
- Snowflake: `NUMBER(38,18)`.
- Ranking fields: INTEGER.
- Counts: INTEGER.

Null values returned by the source must remain null. Missing values must not automatically be replaced with zero.

## 3. Data Layers and Datasets

The platform contains the following logical datasets:

| Dataset | Description | Primary storage |
|---|---|---|
| Cryptocurrency reference | Latest metadata for tracked cryptocurrencies | S3 Silver |
| Daily market snapshot | Daily market state of tracked cryptocurrencies | S3 Silver |
| Historical market observations | Historical price, market cap, and volume observations | S3 Silver |
| Global market snapshot | Daily global cryptocurrency market statistics | S3 Silver |
| Pipeline execution metadata | Pipeline execution and operational information | Logs / operational storage |

Daily top-20 universe membership will initially be derived from the daily market snapshot using market-cap rank. A separate physical dataset is not required at this stage.

## 4. Silver Layer Schemas

The following schemas represent the canonical processed datasets.

### 4.1 `dim_cryptocurrency`

**Purpose:** Maintains the latest reference information for each cryptocurrency.

**Grain:** One record per CoinGecko cryptocurrency ID.

| Column | Data type | Nullable | Description |
|---|---|---|---|
| `coin_id` | STRING | No | CoinGecko cryptocurrency identifier |
| `symbol` | STRING | Yes | Cryptocurrency symbol |
| `name` | STRING | Yes | Cryptocurrency name |
| `last_seen_at` | TIMESTAMP | No | Last time the cryptocurrency was observed |
| `updated_at` | TIMESTAMP | No | Last time the reference record was updated |

**Business key:** `coin_id`

**Update strategy:** Upsert using `coin_id`. The latest name and symbol are maintained. Historical metadata changes are not tracked as separate dimension versions in the initial implementation.

### 4.2 `market_snapshot`

**Purpose:** Stores the daily market state of the tracked cryptocurrencies.

**Grain:** One record per cryptocurrency per scheduled daily snapshot per quote currency.

| Column | Data type | Nullable | Description |
|---|---|---|---|
| `coin_id` | STRING | No | CoinGecko cryptocurrency identifier |
| `snapshot_date` | DATE | No | Scheduled UTC collection date |
| `snapshot_at` | TIMESTAMP | No | Actual collection timestamp |
| `quote_currency` | STRING | No | Reporting currency, initially USD |
| `current_price` | DECIMAL(38,18) | Yes | Current market price |
| `market_cap` | DECIMAL(38,18) | Yes | Market capitalization |
| `total_volume` | DECIMAL(38,18) | Yes | Trading volume |
| `market_cap_rank` | INTEGER | Yes | Market-cap ranking |
| `circulating_supply` | DECIMAL(38,18) | Yes | Circulating supply |
| `total_supply` | DECIMAL(38,18) | Yes | Total supply |
| `max_supply` | DECIMAL(38,18) | Yes | Maximum supply |
| `source_last_updated_at` | TIMESTAMP | Yes | Source-provided update timestamp |
| `ingested_at` | TIMESTAMP | No | API retrieval timestamp |
| `run_id` | STRING | No | Pipeline execution identifier |

**Business key:**

`coin_id + snapshot_date + quote_currency`

**Update strategy:**

- Maintain one canonical snapshot per scheduled date.
- Repeated executions for the same date update the existing logical snapshot.
- Preserve the actual collection timestamp.
- Publish the snapshot as complete only after the expected records have passed validation.

The collection schedule is 00:45 UTC (06:15 IST).

### 4.3 `historical_market_observation`

**Purpose:** Stores historical market measurements for each tracked cryptocurrency.

**Grain:** One record per cryptocurrency per source observation timestamp per quote currency.

| Column | Data type | Nullable | Description |
|---|---|---|---|
| `coin_id` | STRING | No | CoinGecko cryptocurrency identifier |
| `observation_timestamp` | TIMESTAMP | No | Source observation timestamp in UTC |
| `observation_date` | DATE | No | UTC calendar date derived from observation timestamp |
| `quote_currency` | STRING | No | Reporting currency |
| `price` | DECIMAL(38,18) | Yes | Historical price |
| `market_cap` | DECIMAL(38,18) | Yes | Historical market capitalization |
| `total_volume` | DECIMAL(38,18) | Yes | Historical trading volume |
| `ingested_at` | TIMESTAMP | No | API retrieval timestamp |
| `processed_at` | TIMESTAMP | No | Processing timestamp |
| `run_id` | STRING | No | Pipeline execution identifier |

**Business key:**

`coin_id + observation_timestamp + quote_currency`

**Update strategy:**

- Historical observations are initially populated through a backfill of up to 365 days, subject to API plan restrictions.
- Recently published historical observations are periodically refreshed.
- If the source revises an existing observation, the canonical Silver record is updated.
- Original API responses remain available in Bronze.

The historical market-chart endpoint provides separate arrays for prices, market capitalization, and total volume. These arrays must be aligned using their timestamps during processing.

### 4.4 `global_market_snapshot`

**Purpose:** Stores current global cryptocurrency market statistics collected daily.

**Grain:** One record per global market snapshot date.

| Column | Data type | Nullable | Description |
|---|---|---|---|
| `snapshot_date` | DATE | No | Scheduled UTC collection date |
| `snapshot_at` | TIMESTAMP | No | Actual observation timestamp |
| `total_market_cap_usd` | DECIMAL(38,18) | Yes | Total cryptocurrency market capitalization |
| `total_volume_usd` | DECIMAL(38,18) | Yes | Total cryptocurrency trading volume |
| `btc_dominance` | DECIMAL(38,18) | Yes | Bitcoin market dominance percentage |
| `active_cryptocurrencies` | INTEGER | Yes | Active cryptocurrency count |
| `markets` | INTEGER | Yes | Number of markets, if returned |
| `ingested_at` | TIMESTAMP | No | API retrieval timestamp |
| `run_id` | STRING | No | Pipeline execution identifier |

**Business key:** `snapshot_date`

**Update strategy:** Maintain one canonical global market snapshot per scheduled date.

The actual columns will be finalized against the `/global` endpoint response. Only fields returned by the source will be included.

## 5. Snowflake Warehouse Model

Snowflake will organize the data into separate schemas based on transformation responsibility.

### 5.1 Schema organization

| Schema | Responsibility |
|---|---|
| `STAGING` | Source-aligned datasets loaded from S3 |
| `INTERMEDIATE` | Standardized and reusable business transformations |
| `MART` | Analytical fact and dimension models |
| `OPS` | Operational metadata, if maintained in Snowflake |

### 5.2 Analytical models

| Model | Type | Grain |
|---|---|---|
| `DIM_CRYPTOCURRENCY` | Dimension | One record per cryptocurrency |
| `FCT_MARKET_SNAPSHOT` | Fact | One cryptocurrency per daily snapshot and quote currency |
| `FCT_HISTORICAL_MARKET` | Fact | One cryptocurrency per observation timestamp and quote currency |
| `FCT_GLOBAL_MARKET_SNAPSHOT` | Fact | One global market observation per daily snapshot |

### 5.3 Universe membership model

Top-20 membership will initially be derived from `FCT_MARKET_SNAPSHOT`.

The model will identify cryptocurrencies whose market-cap rank is between 1 and 20 for a given snapshot date.

The resulting model can expose:

- `coin_id`
- `snapshot_date`
- `market_cap_rank`

This provides historical membership and ranking analysis from the beginning of daily collection.

A separate physical membership table can be introduced if independent materialization or additional membership attributes become necessary.

### 5.4 Key and constraint handling

Logical business keys must be validated through pipeline checks and dbt tests.

Snowflake standard tables generally do not enforce primary-key and unique constraints in the same manner as transactional databases.

Required validation includes:

- Non-null checks on key columns.
- Uniqueness checks on dimension identifiers.
- Composite uniqueness checks on fact-table business keys.
- Referential-integrity checks between fact tables and cryptocurrency dimensions.

## 6. S3 Storage Organization

The initial S3 structure separates raw and processed data.

```text
s3://<bucket-name>/
│
├── bronze/
│   └── coingecko/
│       ├── coins_markets/
│       │   └── ingestion_date=YYYY-MM-DD/
│       │
│       ├── market_chart/
│       │   └── ingestion_date=YYYY-MM-DD/
│       │
│       ├── global/
│       │   └── ingestion_date=YYYY-MM-DD/
│       │
│       └── coins_list/
│           └── ingestion_date=YYYY-MM-DD/
│
└── silver/
    ├── dim_cryptocurrency/
    │
    ├── market_snapshot/
    │   └── year=YYYY/
    │       └── month=MM/
    │
    ├── historical_market_observation/
    │   └── year=YYYY/
    │       └── month=MM/
    │
    └── global_market_snapshot/
        └── year=YYYY/
            └── month=MM/
```

The bucket name and exact file naming conventions will be established during implementation.

### 6.1 Bronze storage

Bronze retains original API responses in JSON format.

Each retrieval should be preserved as a separate raw artifact.

Associated ingestion metadata should include:

- API endpoint.
- Request parameters, excluding secrets.
- Retrieval timestamp.
- HTTP status code.
- Response checksum.
- Pipeline execution identifier.
- Raw file location.

API keys and other credentials must never be stored in raw request URLs or metadata.

### 6.2 Silver storage

Silver contains canonical, cleaned, and standardized datasets in Parquet format.

The historical observation and daily snapshot datasets will initially use year/month partitioning.

Partitioning by individual cryptocurrency and date is avoided initially because the expected data volume does not justify the additional directory and file overhead.

File sizes and query patterns will guide future partitioning and compaction decisions.

## 7. Data Update and Correction Strategy

| Dataset | Update strategy | Business key |
|---|---|---|
| Cryptocurrency dimension | Upsert | `coin_id` |
| Daily market snapshot | Upsert | `coin_id + snapshot_date + quote_currency` |
| Historical observations | Upsert | `coin_id + observation_timestamp + quote_currency` |
| Global market snapshot | Upsert | `snapshot_date` |

### 7.1 Historical corrections in Parquet

Parquet files do not support ordinary row-level updates.

When historical observations are revised:

1. Identify the affected Silver partition.
2. Read the existing partition.
3. Merge incoming observations using the business key.
4. Retain the latest valid version of each observation.
5. Rewrite the affected Parquet output.
6. Validate uniqueness and record counts before publishing.

The Bronze layer remains unchanged.

### 7.2 Snowflake updates

Snowflake will use `MERGE` operations or equivalent incremental dbt strategies to apply updates from Silver.

Existing records are updated when their business keys match incoming records. New records are inserted when no matching key exists.

## 8. Data Quality Requirements

The following checks are required before processed datasets are considered ready for analytical consumption.

| Dataset | Validation rule |
|---|---|
| Cryptocurrency | `coin_id` must not be null |
| Cryptocurrency | `coin_id` must be unique |
| Market snapshot | Composite business key must be unique |
| Market snapshot | `snapshot_date` must be present |
| Market snapshot | Market-cap rank must be positive when present |
| Historical observations | Composite business key must be unique |
| Historical observations | Observation timestamps must be valid |
| Historical observations | Timestamps must fall within the requested historical window |
| Global snapshot | `snapshot_date` must be unique |
| All datasets | Required ingestion metadata must be present |

Additional business validations:

- Prices must not be negative.
- Market capitalization must not be negative.
- Trading volume must not be negative.
- Supply values must not be negative.
- Missing source values must remain null rather than being replaced with zero.
- Daily market snapshots must contain the expected number of records before being marked complete.

Failed validations must be logged with enough information to investigate the affected dataset, records, and pipeline execution.

## 9. Operational Metadata

Pipeline execution metadata is maintained separately from market datasets.

The following fields are proposed:

| Column | Data type | Description |
|---|---|---|
| `run_id` | STRING | Unique pipeline execution identifier |
| `pipeline_name` | STRING | Name of the executed pipeline |
| `started_at` | TIMESTAMP | Execution start time |
| `completed_at` | TIMESTAMP | Execution completion time |
| `run_status` | STRING | Execution status |
| `records_read` | INTEGER | Number of records read |
| `records_written` | INTEGER | Number of records written |
| `error_message` | STRING | Error details, when applicable |

This information supports execution tracking, troubleshooting, retry management, and operational monitoring.

## 10. Assumptions and Limitations

- The initial cryptocurrency universe contains the top 20 cryptocurrencies by market capitalization.
- The historical backfill covers the currently selected 20 cryptocurrencies, not the historical top-20 membership for the previous 365 days.
- Daily top-20 membership tracking begins with the first scheduled daily market snapshot.
- USD is the initial quote currency.
- Historical market observations are subject to CoinGecko API availability and plan restrictions.
- Current global market statistics are collected daily, subject to endpoint access.
- Historical global market statistics are not part of the initial scope.
- Cryptocurrency metadata changes are not historically versioned in the initial model.
- All physical schemas are subject to final validation against actual API responses.

## 11. Future Considerations

The following improvements may be considered as requirements evolve:

- Historical cryptocurrency metadata tracking using SCD Type 2.
- A separately materialized universe membership dataset.
- S3 Gold datasets for consumers outside Snowflake.
- Additional quote currencies.
- Improved file compaction and partition optimization.
- A lakehouse table format if transactional updates or time-travel requirements emerge.
- Additional operational datasets for pipeline metrics and data-quality history.

---

**Document note:** This is the initial physical data model. Final source-to-target mappings, exact data types, and file layouts will be confirmed during schema validation and implementation design.