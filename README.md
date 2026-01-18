# Rearc Data Engineer Take-Home Assignment

## Overview
This repository contains my solution to the Rearc Data Engineer take-home challenge, implemented using Azure Databricks (Serverless) and Unity Catalog.
The solution demonstrates:
1. ingestion of external public datasets
2. robust, production-friendly data pipelines
3. analytical transformations using PySpark
4. orchestration using Databricks Workflows
5. clear documentation and design trade-offs
The overall design mirrors the intent of the original AWS-oriented quest while leveraging Databricks-native primitives instead of Lambda, SQS, and S3 notifications.

## Architecture

```mermaid
fflowchart LR
  subgraph Sources
    BLS[BLS time-series files<br/>download.bls.gov/pub/time.series/pr/]
    API[Population API<br/>honolulu-api.datausa.io/tesseract]
  end

  subgraph DBX[Azure Databricks – Serverless]
    WF[Databricks Workflow<br/>Scheduled daily]
    A[Task A: Ingest BLS Part 1]
    B[Task B: Ingest Population Part 2]
    C[Task C: Analytics Part 3]
  end

  subgraph UC[Unity Catalog + ADLS Gen2]
    V1[UC Volume: raw_bls<br/>/Volumes/rearc_quest/lakehouse/raw_bls/pr.data.0.Current]
    V2[UC Volume: raw_datausa<br/>/Volumes/rearc_quest/lakehouse/raw_datausa/population.json]
    T1[Delta: population_stats_2013_2018]
    T2[Delta: bls_best_year_by_series]
    T3[Delta: report_prs30006032_q01]
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
1. Two independent ingestion tasks pull data from external public sources.
2. Raw data is stored in Unity Catalog Volumes backed by ADLS Gen2.
3. Analytics runs only after both ingestions complete successfully.
4. Curated outputs are written as Delta tables in Unity Catalog.
5. All tasks run on Databricks Serverless compute.


## Why Databricks (and How It Maps to AWS)
The original Rearc Quest is presented using AWS services (S3, Lambda, SQS, CloudWatch). For this submission, I intentionally chose Azure Databricks as the execution platform and re-implemented the solution using Databricks-native components.
This was a deliberate architectural decision rather than a limitation, and it preserves the intent and behavior of the original design while simplifying operations and reducing moving parts.

### Databricks vs AWS – Conceptual Mapping

| AWS Component           | Databricks Equivalent                 | Rationale                                                       |
| ----------------------- | ------------------------------------- | --------------------------------------------------------------- |
| Amazon S3               | ADLS Gen2 via Unity Catalog Volumes   | Governed object storage with centralized access control         |
| AWS Lambda              | Databricks Notebooks (Serverless)     | Managed execution environment without infrastructure management |
| Amazon SQS              | Databricks Workflow task dependencies | Deterministic orchestration without external queues             |
| CloudWatch Logs         | Databricks Job Run Logs               | Centralized execution logs and observability                    |
| EventBridge / S3 Events | Workflow scheduling + dependencies    | Explicit, predictable execution control                         |

### Why Databricks Was Chosen
1. Unified Data + Compute Platform
Databricks provides ingestion, transformation, analytics, and orchestration within a single platform. This removes the need to wire together multiple cloud services while still supporting scalable, production-grade pipelines.
2. Native Spark for Analytics
The Quest requires non-trivial analytical processing. Databricks allows these transformations to be expressed directly in PySpark, without maintaining separate compute services.
3. Serverless Execution Model
Using Databricks Serverless eliminates infrastructure management concerns (clusters, sizing, scaling), allowing the focus to remain on data correctness and pipeline design.
4. Governance via Unity Catalog
Unity Catalog provides centralized access control, lineage, and discovery across both raw and curated datasets—capabilities that would otherwise require multiple AWS services to replicate.
5. Simplified Orchestration
Databricks Workflows with task dependencies offer deterministic execution and clear failure handling, replacing event-driven chains (e.g., SQS + Lambda) with simpler, easier-to-reason-about control flow.

## Design Decisions & Trade-offs

### 1. Why Unity Catalog Volumes for raw data
Raw data (BLS files and API JSON) is stored in UC Volumes instead of unmanaged cloud paths.
#### Why this matters:
1. Governed, discoverable access to object storage
2. Centralized permissions, lineage, and auditing
3. Native compatibility with serverless compute
4. Clear separation between raw and curated layers

This aligns with modern lakehouse best practices.

### 2. Why overwrite-in-place for the API dataset
The population API output is written to a fixed path and overwritten on each run:
/Volumes/rearc_quest/lakehouse/raw_datausa/population.json

#### Alternatives considered:
1. Versioned paths (ingest_date=YYYY-MM-DD/)
2. File-arrival-based triggers

#### Why overwrite was chosen:
1. The Quest requires the latest population snapshot
2. Overwrite keeps analytics deterministic
3. Downstream logic always reads a single, known location
4. Avoids unnecessary partition discovery logic

This is a deliberate simplicity trade-off appropriate for the scope of this assignment.

### 3. Why tasks + dependencies instead of file arrival triggers
In Databricks, analytics can be triggered either by:
1. File arrival events, or
2. Workflow task dependencies
Task dependencies were chosen.

#### Why file arrival was not used:
1. Overwriting an existing file does not always emit a reliable “new file” event
2. This can introduce nondeterministic behavior
3. Serverless environments favor explicit orchestration

#### Benefits of task dependencies:
1. Guaranteed execution order
2. Clear failure visibility
3. Easier debugging and retries
4. Direct conceptual replacement for AWS SQS + Lambda chaining

This prioritizes reliability and clarity over architectural complexity.

## Workflow Orchestration (Part 4)
The pipeline is orchestrated using a single Databricks Workflow.

### Workflow Structure:
Task A – Ingest BLS (Part 1)
Downloads and stores BLS time-series data

Task B – Ingest Population API (Part 2)
Calls the public API and stores the JSON response

Task C – Analytics (Part 3)
Runs only after Tasks A and B succeed

Tasks A and B run independently and in parallel.
Task C depends on the successful completion of both.

Trigger
Scheduled daily
Serverless compute for all tasks

This design replaces:
AWS Lambda → Databricks notebooks
SQS → task dependencies
CloudWatch logs → Databricks job run logs

## Implementation Breakdown

### Part 1 – BLS Ingestion
1. Reads BLS directory listings
2. Downloads the pr.data.0.Current file
3. Stores it in a UC Volume
4. Parsing is deferred to analytics to keep raw data immutable

### Part 2 – Population API Ingestion
1. Calls the DataUSA Tesseract API
2. Includes retry, backoff, and timeout handling
3. Implements fallback to last cached snapshot on transient failures
4. Writes JSON output to UC Volume (overwrite semantics)

### Part 3 – Analytics
All analytics are implemented using PySpark DataFrames (no pandas).

Key computations:
1. Mean and standard deviation of US population (2013–2018)
2. Best year per BLS series (max annual sum)
3. Joined report for PRS30006032, Q01, and population by year

Results are stored as Delta tables in Unity Catalog.

## Additional Analysis (Beyond Requirements)
To demonstrate analytical depth beyond the assignment requirements, the following analyses were added:
1. Population trend analysis
2. Year-over-year population growth to assess stability and volatility
3. Series volatility analysis
Coefficient of variation per BLS series to identify stable vs volatile indicators
Per-capita normalization
BLS values normalized by population to enable fair cross-year comparison

## How to Run
This project runs entirely in Azure Databricks using serverless compute and Databricks Workflows.

### Prerequisites
An Azure Databricks workspace with:
1. Unity Catalog enabled
2. Access to serverless compute
3. An ADLS Gen2 storage account registered with Unity Catalog
4.Network access to public endpoints:
  1. download.bls.gov
  2. honolulu-api.datausa.io

### One-time Setup

Create catalog and schema (if not already present):
CREATE CATALOG IF NOT EXISTS rearc_quest;
CREATE SCHEMA IF NOT EXISTS rearc_quest.lakehouse;

Create Unity Catalog volumes for raw data:
CREATE VOLUME IF NOT EXISTS rearc_quest.lakehouse.raw_bls;
CREATE VOLUME IF NOT EXISTS rearc_quest.lakehouse.raw_datausa;

Import notebooks into the Databricks workspace:
10_ingest_bls
11_ingest_population
20_analytics

### Running the Pipeline via Databricks Workflow
#### Navigate to Workflows in the Databricks UI

1. Create a new job named:
rearc_quest_pipeline

2. Add the following tasks:
Task A: 10_ingest_bls
Task B: 11_ingest_population
Task C: 20_analytics

3. Configure Task C to depend on Task A and Task B
4. Set compute for all tasks to Serverless
5. Configure a daily schedule (or run manually for testing)

The workflow ensures analytics runs only after both ingestion tasks complete successfully.

### Manual Execution (for testing)
Each notebook can also be run independently:
1. Run 10_ingest_bls to ingest BLS time-series data
2. Run 11_ingest_population to fetch and store population API data
3. Run 20_analytics to generate analytical outputs
Manual execution is useful for development and debugging.

### Verifying Outputs
After a successful run, verify the following:
Raw data (Unity Catalog Volumes)

BLS data:
/Volumes/rearc_quest/lakehouse/raw_bls/pr.data.0.Current

Population API data:
/Volumes/rearc_quest/lakehouse/raw_datausa/population.json

Curated Delta tables
SELECT * FROM rearc_quest.lakehouse.population_stats_2013_2018;
SELECT * FROM rearc_quest.lakehouse.bls_best_year_by_series;
SELECT * FROM rearc_quest.lakehouse.report_prs30006032_q01;

## Failure Handling & Retries
1. External API calls include retries and exponential backoff
2. Fallback to last cached snapshot on transient API failures
3. Workflow tasks are independently retryable
4. Analytics executes only when upstream ingestion succeeds

## Final Notes
1. All notebooks are serverless-compatible
2. No unsupported persistence commands are used
3. The pipeline is deterministic, restartable, and easy to reason about
4. Design choices favor clarity, reliability, and correctness
