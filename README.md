# Cryptocurrency Market Data Engineering Platform

## 1. Project Overview

An end-to-end data engineering platform designed to collect, process, store, and analyze cryptocurrency market data using the CoinGecko API.

The platform aims to provide reliable historical and incremental market datasets for cryptocurrency market analytics.

## 2. Project Objectives

- Build a reliable batch data ingestion pipeline.
- Collect historical and daily cryptocurrency market data.
- Implement data validation, deduplication, and transformation.
- Develop analytics-ready data models.
- Orchestrate pipeline execution.
- Implement testing, monitoring, and failure handling.
- Apply production-oriented engineering practices while keeping infrastructure costs minimal.

## 3. Initial Scope

| Requirement | Initial Decision |
|---|---|
| Data provider | CoinGecko |
| Cryptocurrency universe | Top 20 |
| Quote currency | USD |
| Collection frequency | Daily |
| Processing approach | Batch |
| Historical data | Historical backfill, subject to API limitations |
| Analytics | Price trends, market capitalization, trading volume, rankings, global market trends |

## 4. Engineering Principles

- Reliability and fault tolerance
- Idempotent data processing
- Data quality and validation
- Secure credential management
- Modular and maintainable code
- Reproducible development environments
- Cost-efficient infrastructure
- Comprehensive documentation

## 5. Project Status

Currently in the initial development and data-source research stage.

## 6. Data Attribution

Market data will be sourced from CoinGecko. Applicable attribution and data-use requirements will be followed.

Source: https://www.coingecko.com/