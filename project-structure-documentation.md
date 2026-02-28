# dbt Exercise: Project Structure Documentation

**Stack:** dbt Core 1.11.6 | Snowflake | Windows 11  
**Author:** King | **Date:** February 2026 |
**Duration:** More than 2 hours debugging

---

## Overview

Exercise 6 introduces the **layered architecture** pattern in dbt — the industry standard for organizing models in production projects.

```
raw (seeds/external)
      ↓
staging (stg_)       — 1:1 with source, clean and rename only
      ↓
intermediate (int_)  — joins, business logic, calculations
      ↓
marts (mart_)        — final aggregated output for BI tools
```

---

## Final Project Structure

```
models/
├── staging/
│   ├── stg_customers.sql
│   ├── stg_orders.sql
│   ├── stg_products.sql
│   ├── stg_customers_vw.sql
│   ├── staging_schema.yml
│   └── sources.yml
├── intermediate/
│   ├── int_customer_orders.sql
│   └── intermediate_schema.yml
└── marts/
    ├── mart_customer_summary.sql
    └── marts_schema.yml

seeds/
└── Raw/
    ├── raw_customers.csv
    ├── raw_orders.csv
    └── raw_products.csv
```

---

## Seed Data

### raw_customers.csv
```csv
id,first_name,last_name,email,created_at,status
1,John,Santos,john.santos@email.com,2024-01-01,active
2,Maria,Cruz,maria.cruz@email.com,2024-01-02,active
3,Jose,Reyes,jose.reyes@email.com,2024-01-03,inactive
4,Ana,Garcia,ana.garcia@email.com,2024-01-04,active
5,Pedro,Dela Cruz,pedro.dc@email.com,2024-01-05,inactive
6,Carlo,Mendoza,carlo.mendoza@email.com,2024-01-06,active
7,Lisa,Torres,lisa.torres@email.com,2024-01-07,active
8,Mark,Reyes,mark.reyes@email.com,2024-01-08,pending
```

### raw_orders.csv
```csv
order_id,customer_id,product_name,amount,status,created_at
1,1,Laptop,1200.00,completed,2024-01-05
2,2,Mouse,25.00,completed,2024-01-06
3,3,Keyboard,75.00,pending,2024-01-07
4,1,Monitor,300.00,completed,2024-01-08
5,4,Headset,50.00,cancelled,2024-01-09
6,2,Webcam,80.00,completed,2024-01-10
7,5,Laptop,1200.00,pending,2024-01-11
```

### raw_products.csv
```csv
product_id,product_name,category,unit_price,created_at
1,Laptop,Electronics,1200.00,2024-01-01
2,Mouse,Accessories,25.00,2024-01-01
3,Keyboard,Accessories,75.00,2024-01-01
4,Monitor,Electronics,300.00,2024-01-01
5,Headset,Accessories,50.00,2024-01-01
6,Webcam,Accessories,80.00,2024-01-01
```

---

## SQL Models

### staging/stg_customers.sql
```sql
{{ config(materialized='view') }}

select
    id as customer_id,
    first_name,
    last_name,
    first_name || ' ' || last_name as full_name,
    email,
    status,
    created_at
from {{ source('raw', 'raw_customers') }}
```

### staging/stg_orders.sql
```sql
{{ config(materialized='view') }}

select
    order_id,
    customer_id,
    product_name,
    amount,
    status,
    created_at
from {{ source('raw', 'raw_orders') }}
```

### staging/stg_products.sql
```sql
{{ config(materialized='view') }}

select
    product_id,
    product_name,
    category,
    unit_price,
    created_at
from {{ source('raw', 'raw_products') }}
```

### intermediate/int_customer_orders.sql
```sql
{{ config(materialized='view') }}

select
    o.order_id,
    o.customer_id,
    c.full_name,
    c.email,
    c.status as customer_status,
    o.product_name,
    o.amount,
    o.status as order_status,
    o.created_at as order_date
from {{ ref('stg_orders') }} o
left join {{ ref('stg_customers') }} c
    on o.customer_id = c.customer_id
```

### marts/mart_customer_summary.sql
```sql
{{ config(materialized='table') }}

select
    customer_id,
    full_name,
    email,
    customer_status,
    count(order_id)             as total_orders,
    sum(amount)                 as total_spent,
    avg(amount)                 as avg_order_value,
    min(order_date)             as first_order_date,
    max(order_date)             as last_order_date
from {{ ref('int_customer_orders') }}
group by 1, 2, 3, 4
```

---

## YAML Files

### staging/sources.yml
```yaml
version: 2

sources:
  - name: raw
    database: dbt_db
    schema: raw
    tables:
      - name: raw_customers
        description: "Raw customer data loaded from seeds."
        config:
          loaded_at_field: "cast(created_at as timestamp)"
          freshness:
            warn_after: {count: 500, period: day}
            error_after: {count: 1000, period: day}
        columns:
          - name: id
            description: "Primary key"
          - name: first_name
            description: "Customer first name"
          - name: last_name
            description: "Customer last name"
          - name: email
            description: "Customer email address"
          - name: created_at
            description: "Record creation timestamp"

      - name: raw_orders
        description: "Raw orders data loaded from seeds."
        columns:
          - name: order_id
            description: "Primary key"
          - name: customer_id
            description: "Foreign key to raw_customers"
          - name: product_name
            description: "Name of product ordered"
          - name: amount
            description: "Order amount in USD"
          - name: status
            description: "Order status"
          - name: created_at
            description: "Order creation timestamp"

      - name: raw_products
        description: "Raw products data loaded from seeds."
        columns:
          - name: product_id
            description: "Primary key"
          - name: product_name
            description: "Name of product"
          - name: category
            description: "Product category"
          - name: unit_price
            description: "Product unit price in USD"
          - name: created_at
            description: "Product creation timestamp"
```

### staging/staging_schema.yml
```yaml
version: 2

models:
  - name: stg_customers
    description: "Staged customer data cleaned and standardized from raw source."
    columns:
      - name: customer_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: full_name
        description: "Concatenation of first_name and last_name"
        tests:
          - not_null
      - name: email
        description: "Customer email address"
        tests:
          - not_null
          - unique
      - name: status
        description: "Customer account status"
        tests:
          - accepted_values:
              values:
                - active
                - inactive
      - name: created_at
        description: "Timestamp when customer record was first created"

  - name: stg_orders
    description: "Staged orders data cleaned from raw source."
    columns:
      - name: order_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: customer_id
        description: "Foreign key to stg_customers"
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id
      - name: product_name
        description: "Name of product ordered"
        tests:
          - not_null
      - name: amount
        description: "Order amount in USD"
        tests:
          - not_null
      - name: status
        description: "Order status"
        tests:
          - accepted_values:
              values:
                - completed
                - pending
                - cancelled
      - name: created_at
        description: "Order creation timestamp"

  - name: stg_products
    description: "Staged products data cleaned from raw source."
    columns:
      - name: product_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: product_name
        description: "Name of product"
        tests:
          - not_null
      - name: category
        description: "Product category"
      - name: unit_price
        description: "Product unit price in USD"
        tests:
          - not_null
      - name: created_at
        description: "Product creation timestamp"
```

### intermediate/intermediate_schema.yml
```yaml
version: 2

models:
  - name: int_customer_orders
    description: "Intermediate model joining customers and orders."
    columns:
      - name: order_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: customer_id
        description: "Foreign key to stg_customers"
        tests:
          - not_null
      - name: full_name
        description: "Customer full name"
      - name: email
        description: "Customer email"
      - name: customer_status
        description: "Customer account status"
      - name: product_name
        description: "Name of product ordered"
      - name: amount
        description: "Order amount in USD"
      - name: order_status
        description: "Status of the order"
        tests:
          - accepted_values:
              values:
                - completed
                - pending
                - cancelled
      - name: order_date
        description: "Date the order was created"
```

### marts/marts_schema.yml
```yaml
version: 2

models:
  - name: mart_customer_summary
    description: "Final customer summary mart for BI consumption."
    columns:
      - name: customer_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: full_name
        description: "Customer full name"
      - name: email
        description: "Customer email"
      - name: customer_status
        description: "Customer account status"
      - name: total_orders
        description: "Total number of orders per customer"
      - name: total_spent
        description: "Total amount spent by customer in USD"
      - name: avg_order_value
        description: "Average order value in USD"
      - name: first_order_date
        description: "Date of first order"
      - name: last_order_date
        description: "Date of most recent order"
```

### dbt_project.yml (models section)
```yaml
models:
  dbt_learning:
    +materialized: view
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
```

---

## Snowflake Schema Mapping

| dbt Layer | Snowflake Schema |
|---|---|
| seeds | `dbt_db.raw` |
| staging | `dbt_db.raw_staging` |
| intermediate | `dbt_db.raw_intermediate` |
| marts | `dbt_db.raw_marts` |

> **Note:** Snowflake prefixes schema names with your default schema (`raw`). So `+schema: staging` becomes `raw_staging` in Snowflake.

---

## Verification Queries

```sql
-- raw seeds
select * from dbt_db.raw.raw_customers;
select * from dbt_db.raw.raw_orders;
select * from dbt_db.raw.raw_products;

-- staging
select * from dbt_db.raw_staging.stg_customers;
select * from dbt_db.raw_staging.stg_orders;
select * from dbt_db.raw_staging.stg_products;
select * from dbt_db.raw_staging.stg_customers_vw;

-- intermediate
select * from dbt_db.raw_intermediate.int_customer_orders;

-- marts
select * from dbt_db.raw_marts.mart_customer_summary;
```

---

## Troubleshooting & Errors

| Error | Cause | Fix |
|---|---|---|
| `Object STG_ORDERS does not exist` | Model not built yet, wrong schema | Run staging models first before intermediate |
| `Nothing to do` on stg_orders | Partial parse cache false positive | `dbt clean` then `dbt run --no-partial-parse --full-refresh` |
| `stg_orders` missing from run output | Files outside `staging/` folder | Move all `.sql` files into correct subfolders |
| `raw_staging` schema instead of `staging` | dbt prefixes default schema to custom schema | Expected behavior — use `raw_staging` in Snowflake queries |
| `invalid identifier STATUS` | Column added after table existed | `dbt seed --full-refresh` |
| `Object does not exist` on intermediate | Staging models not built first | Run `stg_orders stg_customers stg_products` before `int_customer_orders` |
| `MissingArgumentsPropertyInGenericTestDeprecation` | Old `accepted_values` syntax | Use list format under `values:` instead of inline list |
| `##` comments in YAML | YAML only supports `#` for comments | Replace `##` with `#` |
| `Product_name` capital P in sources.yml | Case mismatch with CSV column name | Use lowercase `product_name` |
| VS Code showing errors on schema.yml | dbt Power User extension validation lag | Ignore — cosmetic only, doesn't affect pipeline |
| `stg_customers` built as table not view | Hardcoded `materialized='table'` in config | Remove config or change to `view` to inherit from `dbt_project.yml` |

---

## Key Lessons Learned

- **Layer separation is strict** — staging only touches `source()`, intermediate and marts only use `ref()`
- **`dbt_project.yml` schema config** — `+schema: staging` creates a separate Snowflake schema per layer
- **Snowflake schema naming** — dbt prefixes your default schema to custom schemas (e.g. `raw` + `staging` = `raw_staging`)
- **Model config overrides project config** — hardcoded `{{ config() }}` in a model always wins over `dbt_project.yml`
- **YAML comments** — use `#` only, never `##`
- **Column name casing** — always match CSV column names exactly in sources.yml
- **`accepted_values` new syntax** — use list format in dbt 1.11+:
```yaml
#  New syntax
values:
  - completed
  - pending

#  Old syntax (still works but deprecated)
values: ['completed', 'pending']
```
- **sources.yml belongs in staging/** — only staging models reference raw sources
- **One sources.yml** — declare all raw tables in one file, not per table
- **Named schema files** — use `staging_schema.yml`, `intermediate_schema.yml`, `marts_schema.yml` for clarity

---

## Best Practices

| Practice | Reason |
|---|---|
| staging models as `view` | No storage cost, always reflects latest raw data |
| intermediate models as `view` | Lightweight joins, no need to store |
| mart models as `table` | BI tools query these most — fast reads matter |
| One schema.yml per folder | Clean separation, easy to maintain |
| sources.yml in staging only | Only staging touches raw data |
| Named schema files | More professional, easier to navigate in large projects |
| `dbt clean` before troubleshooting | Clears stale cache causing false "Nothing to do" |

---

## Standard Run Sequence for Exercise 6

```bash
# Session start
.\dbt_env\Scripts\activate
cd dbt_learning

# Full rebuild
dbt clean
dbt seed --full-refresh
dbt run --no-partial-parse --full-refresh
dbt test --no-partial-parse

# Check docs
dbt docs generate --no-partial-parse
dbt docs serve
```

---

*dbt Exercise 6 Documentation | King | February 2026*
