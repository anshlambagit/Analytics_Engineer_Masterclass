# Analytics Engineer Masterclass

> A comprehensive visual guide covering the full analytics engineering stack — from role definition to production-grade data pipelines.

---

## Table of Contents

1. [Who is an Analytics Engineer?](#1-who-is-an-analytics-engineer)
2. [Data Lifecycle](#2-data-lifecycle)
3. [Data Modeling](#3-data-modeling)
4. [Slowly Changing Dimensions (SCD)](#4-slowly-changing-dimensions-scd)
5. [Data Warehouse, Data Lake & Lakehouse](#5-data-warehouse-data-lake--lakehouse)
6. [Open Table Formats](#6-open-table-formats)
7. [Delta Lake](#7-delta-lake)
8. [Apache Spark & PySpark](#8-apache-spark--pyspark)
9. [dbt - Data Build Tool](#9-dbt--data-build-tool)
10. [Orchestration with Airflow](#10-orchestration-with-airflow)
11. [Key Technologies at a Glance](#11-key-technologies-at-a-glance)

---

## 1. Who is an Analytics Engineer?

The **Analytics Engineer** sits at the intersection of three traditional data roles:

| Role | Primary Responsibility |
|---|---|
| **Data Engineer** | Build and maintain data pipelines & infrastructure |
| **Data Analyst** | Query data, build reports, surface insights |
| **DBA** | Manage databases, performance, and data integrity |

The Analytics Engineer bridges all three — they transform raw data into clean, tested, documented datasets that analysts can trust, using software engineering best practices applied to data.

```
DATA ENGINEER ──┐
                ├──► ANALYTICS ENGINEER ──► Amazing Result
DATA ANALYST ───┘
      DBA ──────┘
```

---

## 2. Data Lifecycle

### ETL vs ELT vs ELTL

Modern analytics engineering has evolved through several pipeline paradigms:

```
ETL   →  Extract → Transform → Load
ELT   →  Extract → Load → Transform
ELTL  →  Extract → Load → Transform → Load (final layer)
```

| Paradigm | Description | When to Use |
|---|---|---|
| **ETL** | Transform data before loading into the warehouse | Legacy on-prem systems with limited storage |
| **ELT** | Load raw data first, transform inside the warehouse | Cloud data warehouses (Snowflake, BigQuery) |
| **ELTL** | Intermediate staging layer + final transformed layer | Complex pipelines needing a raw archive |

### Pipeline Stages

- **EXTRACT** — Pull data from sources (Web Applications, SQL Databases, CSV files, JSON logs, BLOB Storage)
- **LOAD** — Write raw data into a staging area or data lake
- **TRANSFORM** — Clean, model, and aggregate data for analytical use

### Source Systems

Common data sources handled in modern pipelines:

- Structured: `SQL Database`, relational tables
- Semi-structured: `JSON` log files (`log1.json`, `log2.json`, `log3.json`)
- Flat files: `CSV` (`file-01.csv`), `Parquet`
- Cloud storage: `AWS S3`, `ADLS Gen2`, `BLOB Storage`

---

## 3. Data Modeling

### RDB Modeling (Relational Database Modeling)

The classical approach — normalized tables with foreign key relationships, following **Normal Forms**:

| Normal Form | Goal |
|---|---|
| **1NF** | Eliminate repeating groups; atomic values |
| **2NF** | Remove partial dependencies |
| **3NF** | Remove transitive dependencies |

### Dimensional Data Modeling

Used in data warehouses to optimise for analytical queries:

- **Facts** — Measurable, numeric events (e.g. `orders_fact`)
- **Dimensions** — Descriptive context (e.g. `products_dim`, `orders_dim`, `lineitem_dim`)

Example schema:

```
orders_fact
├── order_sk        (surrogate key)
├── order_id
├── order_price
├── order_qty
└── line_item_sk ──► lineitem_dim
                         ├── line_item_sk
                         ├── line_item_id
                         └── line_item_barcode
```

### Data Modeling Shift

The industry is shifting from rigid dimensional models toward more flexible approaches:

| Model | Description |
|---|---|
| **Dimensional (Facts & Dims)** | Star/snowflake schema; best for well-defined reporting |
| **OBT (One Big Table)** | Denormalised wide table; fast queries, simple tooling |

---

## 4. Slowly Changing Dimensions (SCD)

Slowly Changing Dimensions handle the challenge of tracking historical changes to dimension attributes over time.

### Example Dimension Table: `products_dim`

```
products_dim
├── prod_sk        ← surrogate key
├── prod_id
├── prod_name
├── prod_cat
└── prod_subcat
```

Source table: `products` → `prod_id`, `prod_name`

### SCD Types

| Type | Strategy | Mechanism |
|---|---|---|
| **Type-1** | Overwrite — no history retained | `UPSERT` (Update + Insert) |
| **Type-2** | Full history retained as new rows | Add `in_use_flag`, `from_date`, `to_date` |
| **Type-3** | Limited history — current + one previous value | Extra column per tracked attribute |
| **Type-6** | Hybrid of Type-1 + Type-2 + Type-3 | Surrogate key + flag + date range |

### Type-2 SCD Table Structure

```
products_dim (Type-2)
├── prod_sk
├── prod_id
├── prod_name
├── prod_cat
├── prod_subcat
├── in_use_flag
├── from_date        e.g. 2000-01-01 TO 2026-01-01
└── to_date          e.g. 2026-01-01 TO 2026-01-02
```

Historical rows are closed out (`to_date` set, `in_use_flag = 0`) and a new row is inserted for the current record (`in_use_flag = 1`).

---

## 5. Data Warehouse, Data Lake & Lakehouse

### Storage Architecture Evolution

```
Data Warehouse          →  Structured, SQL-optimised, expensive at scale
Data Lake               →  Raw files, schema-on-read, cheap storage
Data Lake + Warehouse   →  Two separate systems with sync overhead
Data Lakehouse          →  Unified: lake storage + warehouse capabilities
```

### Scaling Strategies

| Strategy | Description |
|---|---|
| **Vertical Scaling** | Add more CPU/RAM to a single node |
| **Horizontal Scaling** | Add more nodes to a distributed cluster |

Cloud data warehouses like **Snowflake** and **Azure Databricks** support elastic horizontal scaling on demand.

### Snowflake

Snowflake is a cloud-native data warehouse with:
- **Separation of storage and compute** — scale independently
- **Virtual warehouses** — independent compute clusters per workload
- Native support for semi-structured data (JSON, Parquet)
- Ideal for the ELT pattern

### Azure Ecosystem

| Service | Role |
|---|---|
| **ADLS Gen2** | Azure Data Lake Storage — scalable object storage |
| **Azure Databricks** | Managed Apache Spark platform |
| **Azure Data Factory** | Orchestration & ETL/ELT pipelines |
| **BLOB Storage** | General-purpose binary/object storage |

### AWS Ecosystem

| Service | Role |
|---|---|
| **AWS S3** | Object storage for data lakes |

---

## 6. Open Table Formats

Open Table Formats add **ACID transactions**, **schema evolution**, and **time travel** on top of raw file storage (Parquet files in a data lake).

| Format | Vendor / Origin | Key Feature |
|---|---|---|
| **Delta Lake** | Databricks (open source) | Transaction log, ACID, Z-ordering |
| **Apache Iceberg** | Netflix (open source) | Hidden partitioning, snapshot isolation |

Both formats allow data lakes to behave like transactional databases, enabling:

- `INSERT`, `UPDATE`, `DELETE` semantics on Parquet files
- Full **ACID** guarantees
- **Time travel** — query historical snapshots
- Schema evolution without rewriting data

---

## 7. Delta Lake

Delta Lake is the most widely adopted open table format, especially in the Azure / Databricks ecosystem.

### Core Architecture

```
Delta Table
├── Parquet data files
└── _delta_log/
    ├── 000.json   ← Transaction Log entry
    ├── 001.json
    └── ...
```

The **Transaction Log** is the source of truth — every operation (insert, update, delete, schema change) is recorded as a JSON commit entry.

### ACID Guarantees

| Property | Meaning |
|---|---|
| **Atomicity** | Operations fully succeed or fully fail |
| **Consistency** | Data always moves from one valid state to another |
| **Isolation** | Concurrent operations don't interfere |
| **Durability** | Committed data is permanently stored |

### Table Types in Databricks / Delta Lake

| Type | Description |
|---|---|
| **Managed (Persistent)** | Databricks manages data & metadata lifecycle |
| **External** | You control the data location; metadata in metastore |
| **Transient** | Temporary; dropped automatically when cluster ends |

### Common Operations

```sql
-- Incremental load using MERGE (UPSERT)
MERGE INTO orders USING updates
ON orders.order_id = updates.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- Delete a record
DELETE FROM orders WHERE order_id = 1;

-- Query a snapshot (time travel)
SELECT * FROM orders VERSION AS OF 5;
```

---

## 8. Apache Spark & PySpark

Apache Spark is the distributed computing engine underlying most modern data lakehouse platforms.

### Architecture

```
Driver Program
     │
     ▼
Cluster Manager
     │
  ┌──┴──┐
Worker  Worker  ← each has CPU + Disk (storage)
Node    Node
```

- **Distributed** — data is split across many nodes and processed in parallel
- **In-memory** — avoids slow disk I/O for iterative workloads
- **Fault-tolerant** — lost partitions are recomputed from lineage

### APIs

| API | Language | Use Case |
|---|---|---|
| **PySpark** | Python | Data engineering, ML pipelines |
| **SparkSQL** | SQL | Familiar SQL interface over distributed data |
| **Scala Spark** | Scala | High-performance native API |

### SparkSQL Example

```sql
-- SparkSQL runs over Delta/Parquet tables at scale
SELECT order_id, SUM(order_price) AS total
FROM orders_fact
GROUP BY order_id
ORDER BY total DESC;
```

---

## 9. dbt — Data Build Tool

dbt is the core tool of the modern Analytics Engineer. It brings **software engineering discipline** (version control, testing, documentation, modularity) to SQL transformations inside the data warehouse.

> *"dbt is a SQL-first transformation workflow that lets teams quickly and collaboratively deploy analytics code."*

### Key Features

| Feature | Description |
|---|---|
| **SQL-first** | Write transformations in plain SQL `SELECT` statements |
| **Jinja templating** | Dynamic SQL with variables, macros, and logic |
| **YAML configuration** | Define sources, models, tests, and documentation in YAML |
| **Python models** | Support for Python-based transformations (dbt-core v1.3+) |
| **Abstraction Layer** | Compile SQL with ref(), source() — no hardcoded paths |

### Project Structure

```
dbt_project/
├── models/
│   ├── staging/          ← Staging Layer: raw → clean
│   │   └── stg_orders.sql
│   ├── marts/
│   │   ├── facts/        ← Facts: measurable events
│   │   │   └── orders_fact.sql
│   │   ├── dims/         ← Dims: descriptive context
│   │   │   └── products_dim.sql
│   │   └── obt/          ← OBT: One Big Table (denormalised)
│   │       └── orders_obt.sql
├── tests/
├── macros/
└── dbt_project.yml       ← YAML project config
```

### Layers

| Layer | Purpose |
|---|---|
| **Staging Layer** | 1-to-1 with source, light cleaning, type casting, renaming |
| **Facts & Dims** | Dimensional model built on top of staging |
| **OBT (One Big Table)** | Flat, wide table for simplified analytics |

### Example dbt Model

```sql
-- models/marts/facts/orders_fact.sql
{{ config(materialized='incremental', unique_key='order_sk') }}

SELECT
    {{ dbt_utils.generate_surrogate_key(['order_id']) }} AS order_sk,
    order_id,
    order_price,
    order_qty
FROM {{ ref('stg_orders') }}

{% if is_incremental() %}
    WHERE updated_at >= '{{ var("last_load") }}'
{% endif %}
```

---

## 10. Orchestration with Airflow

**Apache Airflow** is the industry-standard tool for orchestrating data pipelines. It allows you to schedule, monitor, and manage complex workflows as **DAGs** (Directed Acyclic Graphs).

### Load Strategies

| Strategy | Description | SQL Pattern |
|---|---|---|
| **Initial Load** | Load all historical data on day one | `SELECT * FROM source` |
| **Incremental Load** | Load only new/changed records since last run | `WHERE updated_at >= last_load` |
| **Backfilling** | Re-run historical pipeline runs for a date range | `backfill_flag = 1` |

### Incremental Load Pattern

```sql
-- Incremental load: only process records since last successful run
SELECT *
FROM orders
WHERE updated_at >= last_load  -- e.g. last_load = 2026-01-03
```

### Backfilling

When a pipeline fails or data changes historically, **backfilling** lets you re-process past intervals without manual intervention:

```python
# Airflow backfill example
airflow dags backfill \
    --start-date 2026-01-01 \
    --end-date 2026-01-03 \
    orders_pipeline
```

### Day-by-Day Example

```
Day-1 (2026-01-01):  Initial load → load all rows
Day-2 (2026-01-02):  Incremental → load rows where updated_at >= 2026-01-01
Day-3 (2026-01-03):  Incremental → load rows where updated_at >= 2026-01-02
```

---

## 11. Key Technologies at a Glance

| Technology | Category | Role in Stack |
|---|---|---|
| **Snowflake** | Data Warehouse | Cloud-native SQL warehouse; ELT target |
| **Delta Lake** | Open Table Format | ACID transactions on data lake storage |
| **Apache Iceberg** | Open Table Format | Portable table format; snapshot isolation |
| **Apache Spark / PySpark** | Compute Engine | Distributed data processing |
| **dbt** | Transformation | SQL-first modelling & testing framework |
| **Apache Airflow** | Orchestration | DAG-based pipeline scheduling |
| **Azure Databricks** | Unified Platform | Managed Spark + Delta Lake on Azure |
| **Azure Data Factory** | ETL/Orchestration | Cloud-native data integration service |
| **ADLS Gen2** | Storage | Azure Data Lake Storage |
| **AWS S3** | Storage | Object storage for data lakes |
| **Python** | Language | Pipeline logic, PySpark, dbt Python models |
| **SQL** | Language | Core transformation language |
| **Jinja** | Templating | Dynamic SQL in dbt models |
| **YAML** | Config | dbt project, source, and test definitions |
| **Parquet** | File Format | Columnar storage format for analytics |
| **JSON** | File Format | Semi-structured data, Delta Lake log |
| **Power BI** | Reporting | Business intelligence & dashboards |

---

## Architecture Overview

```
SOURCE SYSTEMS
  Web App │ SQL Database │ CSV │ JSON Logs │ APIs
          │
          ▼
    ┌─────────────┐
    │   EXTRACT   │  Azure Data Factory / Airflow / Custom Scripts
    └──────┬──────┘
           │
           ▼
    ┌─────────────────────────┐
    │   RAW / STAGING LAYER   │  ADLS Gen2 / AWS S3 / BLOB Storage
    │   (Parquet / CSV / JSON) │
    └──────────┬──────────────┘
               │
               ▼
    ┌─────────────────────────┐
    │   OPEN TABLE FORMAT     │  Delta Lake / Iceberg
    │   (ACID + Time Travel)  │
    └──────────┬──────────────┘
               │  ← Spark / SparkSQL transforms
               ▼
    ┌─────────────────────────┐
    │   DATA WAREHOUSE        │  Snowflake / Azure Databricks
    │   Staging → Facts/Dims  │  ← dbt transformations
    │         → OBT           │
    └──────────┬──────────────┘
               │
               ▼
    ┌─────────────────────────┐
    │   REPORTING LAYER       │  Power BI / Dashboards / SQL
    └─────────────────────────┘
```

---

*Built with ❤️ for aspiring Analytics Engineers. Master the stack, own the pipeline.*
