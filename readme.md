# End-to-End Data Engineering Pipeline

### Incremental Data Ingestion | Data Transformation | Data Quality | Orchestration

## Project Overview

This project explores the design of a modern, end-to-end data engineering pipeline, integrating structured data from an **Agentic Database (PostgreSQL)** and file-based data from **Amazon S3** into a unified analytical platform.

The architecture focuses on incremental ingestion, data modelling, transformation, automated quality checks, and workflow orchestration.

## Architecture

**Data Sources → Incremental Ingestion → Databricks → Data Modelling → Data Quality & Transformation**

## 1. Data Sources

The pipeline integrates data from two primary sources:

### Agentic DB (PostgreSQL)

- PostgreSQL serves as the relational database supporting an agentic SQL chatbot.
- Structured data is extracted using Change Data Capture (CDC).
- CDC enables incremental ingestion of new and modified records rather than repeatedly processing the entire source dataset.
- Database changes are propagated into the downstream data processing platform.

### Amazon S3

- Amazon S3 acts as the file-based data source.
- Data is ingested from objects stored in S3 buckets.
- Supports incremental file ingestion to process newly arriving datasets.
- Provides an additional source of data alongside PostgreSQL.

## 2. Incremental Data Ingestion

Both sources feed into an incremental ingestion layer, allowing efficient processing of database changes and newly arriving files.

## 3. Data Processing & Modelling

- Databricks is used as the processing platform.
- Source datasets are consolidated into a **One-Big-Table (OBT)** analytical model.
- Additional curated data models support downstream analytical requirements.

## 4. Data Quality & Transformation

- Data quality checks validate the reliability of ingested datasets.
- dbt supports SQL-based transformation and testing workflows.
- Validated datasets are prepared for analytical consumption.

## 5. Workflow Orchestration

Apache Airflow coordinates ingestion, transformations, and validation through scheduled workflows and task dependencies.

## Technology Stack

| Technology | Purpose |
|---|---|
| Agentic DB (PostgreSQL) | Structured source data for SQL chatbot |
| Amazon S3 | File-based source datasets |
| CDC | Incremental database change ingestion |
| Databricks | Data processing and modelling |
| dbt | Transformation and data quality testing |
| Apache Airflow | Pipeline orchestration |
| Python & SQL | Pipeline development and transformation |

## Acknowledgement

Architecture and learning concepts referenced from educational content by [Ansh Lamba](https://www.youtube.com/@AnshLambaJSR).

This repository documents my learning and practical exploration of modern data engineering architecture.