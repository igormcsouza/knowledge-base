---
tags:

- bigquery
- etl
- elt
- data-engineering
- data-warehouse

---

# BigQuery and Modern ETL/ELT

Reference notes for understanding BigQuery as part of a modern data
platform, with emphasis on concepts useful for Software Engineer / AI
Engineer interviews.

## BigQuery in the Data Stack

BigQuery should be understood primarily as a cloud data warehouse /
analytics engine rather than as the transactional database of an
application.

A common architecture is:

```text
Application / APIs / PostgreSQL / SaaS / Events
                    │
          ingestion / CDC / batch
                    ▼
             BigQuery RAW
                    │
              transformations
                    ▼
          BigQuery STAGING
                    │
                    ▼
         BigQuery ANALYTICS
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
         BI         ML        AI
```

Avoid thinking of BigQuery as a replacement for PostgreSQL in an OLTP
workload. PostgreSQL can remain the application database while BigQuery
becomes the analytical warehouse.

## ETL vs. ELT

**ETL** — `Extract → Transform → Load`. Data is transformed before being
loaded into the warehouse.

**ELT** — `Extract → Load → Transform`. Raw data is first loaded into the
warehouse and transformations are executed there, commonly with SQL.

For BigQuery, ELT is often attractive because BigQuery provides the
distributed compute needed for large analytical transformations.

!!! tip "Interview point"
    SQL operations such as `JOIN`, `GROUP BY`, filtering, `CASE` expressions,
    and window functions are often better performed inside BigQuery instead
    of pulling large datasets into Python purely to transform them.

## BigQuery Hierarchy

Typical organization:

```text
GCP Project
├── ecommerce_raw
│   ├── orders
│   ├── customers
│   └── products
│
├── ecommerce_staging
│   └── orders_clean
│
└── ecommerce_analytics
    ├── daily_sales
    ├── customer_ltv
    └── product_performance
```

Think in terms of projects, datasets, tables/views, schemas, partitions and
physical organization.

## Raw → Staging → Curated/Analytics

A useful warehouse pattern:

```text
RAW
  ↓
STAGING
  ↓
CURATED / ANALYTICS
```

- **RAW** — keep data close to its source representation. Useful for
  replay, auditing and recovering from downstream transformation changes.
- **STAGING** — clean and normalize source data: types, naming, null
  handling, deduplication, basic validation.
- **CURATED / ANALYTICS** — business-oriented models optimized for
  consumption, reporting, analytics, ML and AI.

## Data Ingestion Patterns

### Batch

```text
Source → CSV/JSON/Parquet → Cloud Storage → BigQuery
```

### Streaming

```text
Application → Pub/Sub → BigQuery
```

Useful when events need to become analytically available with low latency.

### CDC (Change Data Capture)

```text
PostgreSQL / source DB
        ↓
       CDC
        ↓
    BigQuery RAW
```

A CDC tool such as Datastream can propagate database changes into the
analytical platform.

The important architectural distinction:

- batch = process periodically
- streaming = process events continuously/as they arrive
- CDC = propagate changes from an operational source into the analytical
  system

## Python's Role

Python remains useful for: calling external APIs, extraction logic, custom
validation, enrichment, data preparation that is difficult to express in
SQL, custom ML/NLP/image processing, and orchestration helpers/services.

A key architectural question:

> Should this transformation run in Python or inside BigQuery?

Prefer BigQuery SQL for warehouse-native relational transformations. Prefer
Python / Dataflow / Spark or another compute layer when the transformation
requires complex procedural logic, external APIs, ML or other processing
that does not fit warehouse SQL well.

## Dataform

Dataform is a useful layer for treating SQL transformations as software.

Typical structure:

```text
dataform/
├── definitions/
│   ├── staging_orders.sqlx
│   ├── staging_customers.sqlx
│   ├── daily_sales.sqlx
│   └── customer_ltv.sqlx
└── workflow_settings.yaml
```

Concepts to know: SQLX, dependencies between models, reusable references,
incremental tables, assertions / data-quality checks, version control,
scheduled workflows.

```sql
config {
  type: "table",
  schema: "analytics"
}

SELECT
    DATE(order_date) AS date,
    SUM(total_amount) AS revenue
FROM ${ref("staging_orders")}
GROUP BY date
```

`${ref(...)}` expresses model dependencies so the workflow can build tables
in the required order.

## Data Quality

A successful SQL job does not necessarily mean a successful data pipeline.

Important checks: `NOT NULL` constraints where required, uniqueness,
referential integrity, valid ranges, valid enumerations, business rules,
row-count anomalies, freshness, schema compatibility.

Example business checks:

```text
order_id IS NOT NULL
customer_id IS NOT NULL
amount >= 0
order_id is unique
```

Data quality should be treated as a first-class pipeline concern.

## Incremental Processing

Do not recompute hundreds of millions of rows every day when only a small
amount of data changed.

```text
Existing warehouse: 500M rows
Today's new/changed data: 100K rows

Goal: process the 100K changed rows instead of rebuilding everything
```

Common strategies: partition by date/time, watermark columns, ingestion
timestamps, CDC streams, incremental models, MERGE/upsert patterns where
appropriate.

!!! tip "Interview concept"
    Incremental processing improves both performance and cost, but
    introduces correctness concerns around late-arriving data, updates,
    deletions and idempotency.

## Idempotency and Retries

Production pipelines must be designed so a failed/retried execution does
not duplicate data or corrupt downstream state.

Ask: *if this task runs twice, what happens?*

Useful techniques: deterministic record identifiers, deduplication,
MERGE/upsert, checkpoints / watermarks, atomic writes where possible,
exactly-once or effectively-once semantics where supported.

Retries are expected in distributed systems, so idempotency is a core
ETL/ELT design principle — see also [Idempotency](../databases/idempotency.md)
for the general database-level treatment of this concept.

## Orchestration

An orchestrator coordinates the pipeline rather than necessarily doing the
transformation itself.

Conceptual DAG:

```text
extract
   ↓
load
   ↓
transform
   ↓
validate
   ↓
publish
   ↓
notify
```

Tools to know: Airflow, Google Cloud Workflows, scheduler/event-driven
mechanisms, Dataform workflow scheduling.

Airflow is especially relevant when the pipeline contains many
dependencies, retries, sensors, backfills and cross-system tasks.

## Warehouse Modeling

Basic analytical modeling concepts:

- **Fact tables** — events/measures such as orders, sales, transactions,
  page views.
- **Dimension tables** — descriptive entities such as customers, products,
  stores, dates.

A common model is the star schema:

```text
          dim_customer
               │
               │
 dim_product ─ fact_sales ─ dim_date
               │
               │
           dim_store
```

The objective is analytical usability and performance rather than OLTP
normalization at all costs.

## Partitioning and Clustering

For BigQuery performance and cost, understand:

**Partitioning** — split table data into partitions, commonly by a
date/time column:

```text
orders
├── 2026-09-06
├── 2026-09-07
└── 2026-09-08
```

Queries that filter on the partitioning field can avoid scanning unrelated
partitions.

**Clustering** — organize data within partitions according to selected
columns to improve filtering/aggregation patterns.

!!! tip "Interview principle"
    Reduce the amount of data scanned and design tables around actual query
    patterns.

## Cost Awareness

With analytical warehouses, performance and cost are often related to how
much data a query processes. Good practices: avoid `SELECT *` when
unnecessary, use partitions effectively, cluster on useful access patterns,
filter early, avoid repeatedly rebuilding unchanged data, use incremental
transformations, monitor expensive queries/jobs.

## Observability

A production data platform needs visibility into: pipeline duration,
failures, retries, data freshness, row counts, throughput,
warehouse/query cost, schema changes, data-quality failures.

Think of observability at three levels:

```text
Infrastructure → Pipeline → Data
```

A green infrastructure dashboard does not guarantee healthy data.

## Schema Evolution

Source schemas change: new columns, renamed columns, removed columns, type
changes, nested structures.

A robust pipeline needs a strategy for detecting and handling schema
changes instead of silently producing incorrect downstream datasets.

## Security and Governance

Important concerns: IAM / least privilege, service accounts,
dataset/table permissions, encryption, sensitive/PII handling, auditing,
data retention, lineage, governance.

For enterprise systems, security and governance are part of the data
architecture rather than post-processing.

## BigQuery + AI

BigQuery can be part of an AI architecture, not only a BI/reporting system.

```text
Operational data
      ↓
   BigQuery
      ↓
Analytics / features / retrieval data
      ↓
Python / FastAPI
      ↓
LLM / AI application
```

Potential applications: analytics-assisted chat, feature generation,
customer segmentation, model training datasets, RAG metadata/data
preparation, natural-language analytics.

For an AI Engineer interview, the key idea is that high-quality AI depends
on reliable data pipelines.

## End-to-End Example Project

A useful personal project is an **E-commerce Data Platform**:

```text
Python data generator / external APIs
                │
                ▼
       Cloud Storage / events
                │
                ▼
         BigQuery RAW
                │
                ▼
            Dataform
                │
       ┌────────┴────────┐
       ▼                 ▼
   STAGING          ANALYTICS
                       │
          ┌────────────┼─────────────┐
          ▼            ▼             ▼
      daily_sales  customer_ltv  product_perf
                       │
                       ▼
                  FastAPI / AI
                       │
                       ▼
             Natural-language analytics
```

Suggested implementation scope:

1. Generate realistic customers/products/orders with Python.
1. Load raw data into BigQuery.
1. Create staging models with Dataform.
1. Create analytical tables such as sales, customer LTV and product
   performance.
1. Add data-quality assertions.
1. Implement incremental processing.
1. Add an orchestrated workflow with retries.
1. Add partitioning/clustering and inspect query cost.
1. Expose an analytics API with FastAPI.
1. Optionally add an LLM interface that answers questions using curated
   BigQuery data.

## Interview-Ready Architecture Answer

A strong high-level answer for a GCP data pipeline is:

> "I would separate ingestion, storage, transformation, orchestration and
> consumption. Operational systems would remain optimized for
> transactions, while batch data, events or CDC would feed a raw BigQuery
> layer. I would use Dataform or SQL-based transformations to build
> staging and curated analytical models, with incremental processing and
> data-quality assertions. An orchestrator such as Airflow would handle
> dependencies, retries and backfills when the workflow complexity
> requires it. I would also design for partitioning, cost control,
> observability, IAM and idempotency from the beginning."

## Concepts to Memorize for Interviews

```text
ETL vs ELT
OLTP vs OLAP
Raw / staging / curated
Batch vs streaming
CDC
Fact vs dimension
Star schema
Partitioning
Clustering
Incremental processing
Watermarks
Idempotency
Retries
Backfills
Data quality
Schema evolution
Orchestration
Data lineage
Observability
IAM / governance
Cost optimization
```

## Personal Positioning

If asked about professional BigQuery experience, be precise and honest. Do
not claim production experience that does not exist.

A good positioning is:

> "BigQuery wasn't part of my previous professional stack, but the
> underlying engineering concepts are familiar to me: distributed systems,
> APIs, asynchronous pipelines, cloud infrastructure, SQL/data processing
> and production operations. I've been studying BigQuery specifically as a
> warehouse and built a personal end-to-end pipeline to understand
> ingestion, ELT, Dataform, incremental processing and data quality."

This frames BigQuery as a tooling gap rather than a fundamental engineering
gap.

## Summary

- BigQuery is an analytical warehouse, not an OLTP replacement — pair it
  with an operational database rather than in place of one.
- ELT (load raw, transform in-warehouse with SQL) is usually preferred over
  ETL for BigQuery-centric pipelines.
- Layer data as raw → staging → curated/analytics, and choose
  batch/streaming/CDC ingestion based on latency and source needs.
- Idempotency, incremental processing, data quality, and observability are
  first-class pipeline concerns, not afterthoughts.
- Partitioning and clustering directly control cost and performance by
  reducing bytes scanned.

## Related Articles

- [Idempotency](../databases/idempotency.md) — the general idempotent-write
  concept applied here to pipeline retries and MERGE/upsert patterns.
- [Event-Driven Architecture](../architecture/event-driven-architecture.md)
  — relevant background for streaming ingestion and CDC-based pipelines.
