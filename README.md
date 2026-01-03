# ELT Pipeline with dbt + Snowflake + Airflow (Astronomer Cosmos)

This project is a hands-on ELT pipeline built with **dbt Core** for transformations, **Snowflake** as the data warehouse, and **Apache Airflow** (via **Astronomer Cosmos**) for orchestration.

It uses the **Snowflake sample TPCH dataset** as the source and builds a small dimensional-style model with staging, marts, macros, and tests.

---

## Introduction

Modern analytics teams often work with data that is already available in a cloud warehouse, but it is not immediately usable for reporting and decision-making. Raw tables usually come with inconsistent naming, unclear business meaning, and no guarantees about data quality.

This project is a compact end-to-end example of how companies build a reliable ELT pipeline using an “analytics engineering” workflow.

---

## The story (What’s happening in real life?)

Imagine a company that already has order and line-item data in its warehouse. Analysts and BI users want simple, trustworthy answers to questions like:
- How much revenue did we generate per order?
- How much discount was applied per order?
- Are the order statuses valid?
- Can we trust that keys and relationships are consistent?

The raw tables can answer these questions, but not safely and not consistently—especially when multiple people write their own SQL in different ways.

---

## The problem

Raw warehouse data is rarely analytics-ready:
- **Inconsistent column naming** and unclear definitions
- **Business logic duplicated** across many queries (hard to maintain)
- **No automated data quality checks** (nulls, duplicates, broken relationships)
- **No orchestration** (transformations are not scheduled or observable)
- **No clear lineage** (hard to know what depends on what)

This leads to “metric confusion”, brittle dashboards, and wasted time.

---

## The solution (What we built)

We implemented a structured ELT pipeline where:

1. **Snowflake** acts as the central warehouse (storage + compute).  
   We use the free **TPCH sample dataset** as the source.

2. **dbt** transforms raw source tables into trusted analytics models:
   - **Staging layer (views):** 1:1 models aligned with source tables (renaming/standardization)
   - **Marts layer (tables):** business transformations + aggregations
   - A final **fact table (`fct_orders`)** containing business-ready measures (sales, discounts)

3. **dbt tests** validate data quality automatically:
   - Generic tests: `unique`, `not_null`, `relationships`, `accepted_values`
   - Singular tests: SQL queries that return failing rows

4. **Airflow (Astronomer Cosmos)** orchestrates dbt:
   - Runs models and tests on a schedule
   - Builds a dependency graph automatically
   - Improves visibility and operational reliability

---

## Why these tools?

- **Snowflake:** scalable warehouse with separated compute/storage and easy access to sample datasets.
- **dbt:** best practice for analytics transformations (SQL + version control + tests + modular models).
- **Airflow + Cosmos:** production-style orchestration with scheduling, retries, logging, and a DAG view; Cosmos makes dbt integration straightforward.

---

## Outcome

At the end of this pipeline, we produce a clean and tested **fact table (`fct_orders`)** that can be consumed by dashboards or downstream analytics with confidence:
- consistent definitions
- reusable business logic (macros)
- validated data quality (tests)
- repeatable runs (orchestration)

---

## Architecture (High-level)

**ELT flow**
1. **Extract & Load**: Data already exists in Snowflake (`SNOWFLAKE_SAMPLE_DATA.TPCH_SF1`)
2. **Transform**: dbt models create:
   - **Staging views** (1:1 with source tables)
   - **Marts tables** (business transformations, aggregations)
   - A **fact table** (`fct_orders`) with numeric measures

**Orchestration**
- Airflow runs dbt using **Astronomer Cosmos**, which automatically builds a task graph based on dbt dependencies (`ref`, `source`, tests).

---

## Repository Structure

- `data_pipeline/`  
  The **dbt project** (models, macros, tests, packages)
  - `models/staging/`: staging models (views) — 1:1 with source tables  
  - `models/marts/`: business transformations (tables), includes intermediate + fact model  
  - `macros/`: reusable SQL logic (DRY)  
  - `tests/`: singular tests (SQL queries returning failing rows)  
  - `packages.yml`: third-party dbt packages (e.g., `dbt_utils`)

- `dbt-dag/`  
  The **Airflow/Astro project** that runs dbt via Astronomer Cosmos
  - `dags/`: Airflow DAG definition(s)
  - `Dockerfile`: includes a dbt virtual environment inside the Airflow image
  - `requirements.txt`: includes `astronomer-cosmos` and Snowflake provider

---

## Data Modeling Notes

### Staging Layer
Staging models are **1:1 mappings** to the source tables:
- Rename columns into analytics-friendly names
- Apply light type casting if needed
- Keep logic minimal

### Marts Layer
Marts models contain business logic:
- `int_order_items` joins orders + line items and computes discounts
- `int_order_items_summary` aggregates sales & discount measures per order
- `fct_orders` combines order attributes with numeric measures

### Why a Fact Table?
A **fact table** stores numeric measures produced by a business process (e.g., sales amounts, discounts), and connects to dimensions via keys.  
In this project, `fct_orders` stores measures like:
- `gross_item_sales_amount`
- `item_discount_amount`

---

## dbt Macros (DRY)
A macro is used to reuse business logic across models:
- `macros/pricing.sql`: `discounted_amount(extended_price, discount_percentage)`

---

## Testing Strategy (dbt)
Two types of dbt tests are included:

### Generic Tests (YAML-based)
- `unique`, `not_null`
- `relationships` (foreign key validation)
- `accepted_values` for `status_code` (`P`, `O`, `F`)

### Singular Tests (SQL-based)
Located in `tests/`:
- `fct_orders_discount.sql`: ensures discounts are not positive (or checks valid logic)
- `fct_orders_date_valid.sql`: ensures order dates are within an acceptable range

---

## How to Run Locally (dbt)

> You need a Snowflake account and credentials.

1. Install dependencies
```bash
pip install dbt-core dbt-snowflake
```

2. Initialize / configure your dbt profile (`~/.dbt/profiles.yml`)  
Then run:
```bash
dbt deps
dbt run
dbt test
```

---

## How to Run with Airflow (Astro)

1. Install Astro CLI
2. Start Airflow locally
```bash
astro dev start
```

3. In Airflow UI, create a Snowflake connection (`snowflake_conn`) and set:
- account
- warehouse
- database
- role
- schema
- username/password

4. Trigger the DAG:
- `dbt_dag`

---

## Cost Control (Snowflake)
To avoid unexpected usage, use:
- **X-Small** warehouse size
- Auto-suspend (e.g., 60s)
- (Optional) Resource monitors with quotas

---

## Credits
Built by following a code-along tutorial and adapting it into a complete GitHub project structure:
- dbt + Snowflake + Airflow (Astronomer Cosmos)
