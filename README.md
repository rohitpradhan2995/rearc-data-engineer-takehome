# Rearc Data Engineer Take Home Assignment

## Overview
This repository contains my solution to the Rearc Data Engineer take-home challenge.

## Architecture

```mermaid
flowchart LR
  subgraph Sources
    BLS[BLS time-series files<br/>download.bls.gov/pub/time.series/pr/]
    API[Population API<br/>honolulu-api.datausa.io/tesseract]
  end

  subgraph DBX[Azure Databricks - Serverless]
    WF[Databricks Workflow<br/>Scheduled daily]
    A[Task A: Ingest BLS Part 1]
    B[Task B: Ingest Population Part 2]
    C[Task C: Analytics Part 3]
  end

  subgraph UC[Unity Catalog + ADLS Gen2]
    V1[UC Volume: raw_bls<br/>/Volumes/rearc_quest/lakehouse/raw_bls/pr.data.0.Current]
    V2[UC Volume: raw_datausa<br/>/Volumes/rearc_quest/lakehouse/raw_datausa/population.json]
    T1[Delta: rearc_quest.lakehouse.population_stats_2013_2018]
    T2[Delta: rearc_quest.lakehouse.bls_best_year_by_series]
    T3[Delta: rearc_quest.lakehouse.report_prs30006032_q01]
  end

  WF --> A
  WF --> B
  A --> V1
  B --> V2

  A --> C
  B --> C
  V1 --> C
  V2 --> C

  C --> T1
  C --> T2
  C --> T3

  BLS --> A
  API --> B
```


### How to read this architecture
1. Two independent ingestion tasks pull data from external public sources
2. Raw data is stored in Unity Catalog Volumes backed by ADLS Gen2
3. Analytics runs only after both ingestions succeed
4. Curated outputs are written as Delta tables in Unity Catalog
5. Everything runs on Databricks Serverless compute

This architecture mirrors the intent of the original AWS-based design, but uses Databricks-native primitives instead of Lambda, SQS, and S3 notifications.


