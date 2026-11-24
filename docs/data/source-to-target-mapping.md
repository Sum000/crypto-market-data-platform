# Source-to-Target Mapping

## 1. Purpose

This document defines how CoinGecko API responses are captured, stored, and transformed into the platform's canonical Silver datasets.

It specifies source fields, target columns, transformation rules, timestamp handling, and validation requirements.

The mappings are based on the API responses tested during source analysis. Runtime validation will also be performed for every ingestion run.

## 2. Source Endpoints

| Endpoint | Purpose | Target dataset |
|---|---|---|
| `/coins/markets` | Daily market data for the top 20 cryptocurrencies | `market_snapshot` |
| `/coins/{id}/market_chart` | Historical price, market capitalization, and volume | `historical_market_observation` |
| `/global` | Global cryptocurrency market statistics | `global_market_snapshot` |
| `/coins/list` | Cryptocurrency identifiers and reference metadata | `dim_cryptocurrency` |

All requests initially use USD as the quote currency.

## 3. Common Ingestion Metadata

The following metadata is associated with every raw API response:

| Metadata field | Description |
|---|---|
| `endpoint` | API endpoint used |
| `request_parameters` | Request parameters, excluding credentials |
| `ingested_at` | UTC time when the response was retrieved |
| `http_status_code` | HTTP response status |
| `response_checksum` | Checksum of the raw response |
| `run_id` | Unique pipeline execution identifier |
| `raw_file_path` | Location of the stored Bronze artifact |

API keys and credentials must never be written to raw request metadata.

## 4. Cryptocurrency Reference Mapping

**Source:** `/coins/list`

**Target:** `dim_cryptocurrency`

| Source JSON field | Target column | Transformation |
|---|---|---|
| `id` | `coin_id` | Direct mapping; required identifier |
| `symbol` | `symbol` | Direct mapping |
| `name` | `name` | Direct mapping |

Additional metadata:

- `last_seen_at`: UTC ingestion time when the cryptocurrency was observed.
- `updated_at`: UTC time when the Silver reference record was updated.

**Grain:** One record per CoinGecko ID.

**Business key:** `coin_id`

**Processing rules:**

- Use `id` as the identifier, not `symbol`.
- Validate uniqueness and non-null values for `id`.
- Maintain the latest reference attributes using an upsert.
- Retain the full API response in Bronze.
- The reference endpoint contains the wider cryptocurrency universe; market ingestion remains restricted to the selected top 20.

## 5. Daily Market Snapshot Mapping

**Source:** `/coins/markets`

**Target:** `market_snapshot`

| Source JSON field | Target column | Transformation |
|---|---|---|
| `id` | `coin_id` | Direct mapping |
| `current_price` | `current_price` | Preserve numeric value |
| `market_cap` | `market_cap` | Preserve numeric value |
| `total_volume` | `total_volume` | Preserve numeric value |
| `market_cap_rank` | `market_cap_rank` | Integer |
| `circulating_supply` | `circulating_supply` | Preserve numeric value |
| `total_supply` | `total_supply` | Preserve numeric value |
| `max_supply` | `max_supply` | Preserve numeric value or NULL |
| `last_updated` | `source_last_updated_at` | Convert source timestamp to UTC |

Additional derived fields:

| Target column | Derivation |
|---|---|
| `snapshot_date` | UTC calendar date of the scheduled collection |
| `snapshot_at` | Actual UTC collection timestamp |
| `quote_currency` | Request parameter; initially `usd` |
| `ingested_at` | UTC retrieval timestamp |
| `run_id` | Pipeline execution identifier |

**Grain:** One cryptocurrency per scheduled daily snapshot per quote currency.

**Business key:** `coin_id + snapshot_date + quote_currency`

**Processing rules:**

- Request the top 20 using market-cap descending order.
- Do not hardcode cryptocurrency IDs.
- Preserve optional null values.
- Do not replace missing values with zero.
- Validate that 20 distinct cryptocurrency IDs are returned before marking the snapshot complete.
- Preserve the original response in Bronze.

## 6. Historical Market Observation Mapping

**Source:** `/coins/{id}/market_chart`

**Target:** `historical_market_observation`

The source response contains three arrays: `prices`, `market_caps`, and `total_volumes`.

Each array contains pairs in the following format:

`[timestamp_in_milliseconds, value]`

| Source field | Target column | Transformation |
|---|---|---|
| `prices[][0]` | `observation_timestamp` | Convert Unix milliseconds to UTC timestamp |
| `prices[][1]` | `price` | Preserve numeric value |
| `market_caps[][1]` | `market_cap` | Match using the observation timestamp |
| `total_volumes[][1]` | `total_volume` | Match using the observation timestamp |

Additional fields:

| Target column | Derivation |
|---|---|
| `coin_id` | CoinGecko ID used in the request |
| `observation_date` | UTC calendar date derived from `observation_timestamp` |
| `quote_currency` | Request parameter; initially `usd` |
| `ingested_at` | UTC retrieval timestamp |
| `processed_at` | UTC processing timestamp |
| `run_id` | Pipeline execution identifier |

**Grain:** One cryptocurrency per source observation timestamp per quote currency.

**Business key:** `coin_id + observation_timestamp + quote_currency`

**Processing rules:**

- Align all three arrays using timestamps rather than assuming array position is a permanent identifier.
- Validate matching timestamp sequences before combining values.
- Validate uniqueness and chronological ordering of timestamps.
- Preserve source nulls.
- Retain the exact source timestamp.
- Do not treat the latest partial observation as a finalized daily closing value.
- Refresh the recent observation window to account for source revisions.
- Retain the original response in Bronze.

**Backfill:** Up to 365 days for the currently selected top 20, subject to API plan restrictions.

## 7. Global Market Snapshot Mapping

**Source:** `/global`

**Target:** `global_market_snapshot`

| Source JSON field | Target column | Transformation |
|---|---|---|
| `data.total_market_cap.usd` | `total_market_cap_usd` | Preserve numeric value |
| `data.total_volume.usd` | `total_volume_usd` | Preserve numeric value |
| `data.market_cap_percentage.btc` | `btc_dominance` | Preserve percentage value |
| `data.active_cryptocurrencies` | `active_cryptocurrencies` | Integer |
| `data.markets` | `markets` | Integer |
| `data.market_cap_change_percentage_24h_usd` | `market_cap_change_percentage_24h` | Preserve percentage value |
| `data.volume_change_percentage_24h_usd` | `volume_change_percentage_24h` | Preserve percentage value |
| `data.updated_at` | `source_updated_at` | Convert Unix seconds to UTC timestamp |

Additional fields:

| Target column | Derivation |
|---|---|
| `snapshot_date` | UTC calendar date of the scheduled collection |
| `snapshot_at` | Actual UTC collection timestamp |
| `ingested_at` | UTC retrieval timestamp |
| `run_id` | Pipeline execution identifier |

**Grain:** One global market snapshot per scheduled date.

**Business key:** `snapshot_date`

**Processing rules:**

- Extract USD values from the nested market-cap and volume objects.
- Extract BTC dominance from `market_cap_percentage`.
- Convert `updated_at` from Unix seconds, not milliseconds.
- Preserve the full source response in Bronze.
- Keep the nested `fear_and_greed_index` object in Bronze until its analytical grain and update semantics are separately established.

## 8. Timestamp Handling

All timestamps are standardized to UTC.

| Source | Timestamp representation | Processing |
|---|---|---|
| `/coins/markets` | Source `last_updated` timestamp | Parse and normalize to UTC |
| `/market_chart` | Unix milliseconds | Convert milliseconds to UTC |
| `/global` | Unix seconds | Convert seconds to UTC |
| `/coins/list` | No source observation timestamp used | Attach ingestion timestamp |

The following timestamps must remain conceptually separate:

- Source observation/update time.
- Scheduled snapshot date.
- Actual collection time.
- Ingestion time.
- Processing time.

## 9. Null Handling

- Source nulls remain null in Silver.
- Missing optional fields do not invalidate an otherwise valid record.
- Required identifiers and business-key fields must be present.
- Zero and null must never be treated as interchangeable.
- Raw Bronze responses remain unchanged.

## 10. Validation Requirements

| Dataset | Validation |
|---|---|
| `dim_cryptocurrency` | Non-null and unique `coin_id` |
| `market_snapshot` | Expected record count, unique coin IDs, valid business key |
| `historical_market_observation` | Matching source timestamps, uniqueness, chronological order |
| `global_market_snapshot` | One canonical record per scheduled date |
| All datasets | Valid timestamps, numeric types, ingestion metadata |

Additional checks:

- Price, market capitalization, volume, and supply must not be negative when present.
- Market-cap rank must be positive when present.
- Unexpected response schemas must be detected and logged.
- Failed validation must prevent incomplete datasets from being published as complete.

## 11. Known Limitations

- API validation has been performed using representative responses, not every cryptocurrency.
- Historical backfill covers the currently selected top 20, not reconstructed historical top-20 membership.
- Historical global statistics are outside the initial scope.
- Fear & Greed data is retained in Bronze but not yet modeled in Silver.
- API response structures and plan limits must be revalidated during implementation and when the source changes.
