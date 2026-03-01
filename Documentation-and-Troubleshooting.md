# dbt Troubleshooting Documentation

**Stack:** dbt Core 1.11.6 | Snowflake | Windows 11 | Python 3.11.9  
**Date:** February 2026 | **Author:** King

---

## First Model 

### What We Did
- Created `seeds/Raw/raw_customers.csv` with 5 rows
- Created `models/stg_customers.sql` using `{{ ref('raw_customers') }}`
- Ran `dbt seed` then `dbt run`

### Key Commands
```bash
dbt seed
dbt run
```

### Key Concepts
- `{{ ref() }}` references other models/seeds and builds dependency order automatically
- dbt builds a DAG (Directed Acyclic Graph) to determine execution order
- Models are created as views by default in Snowflake

### Verification
```sql
select * from dbt_db.raw.stg_customers;
```

---

## Materializations 

### Four Materialization Types

| Type | Physical in Snowflake | Use Case |
|---|---|---|
| `view` | No (saved query) | Lightweight, always fresh |
| `table` | Yes (full rebuild) | Frequently queried, expensive logic |
| `incremental` | Yes (new rows only) | Large datasets, append/upsert |
| `ephemeral` | No (CTE only) | Reusable logic, no storage needed |

### Models Created
- `stg_customers.sql` — table
- `stg_customers_vw.sql` — view on top of stg_customers
- `stg_customers_incremental.sql` — incremental
- `ephemeral_customers.sql` — ephemeral
- `final_customers.sql` — table referencing ephemeral

### Incremental Model Pattern
```sql
{{ config(materialized='incremental', unique_key='customer_id') }}

select * from {{ ref('raw_customers') }}

{% if is_incremental() %}
    where created_at > (select max(created_at) from {{ this }})
{% endif %}
```

### How to Verify Incremental
- **Snowflake Query History** — look for `MERGE INTO` (incremental) vs `CREATE OR REPLACE TABLE` (full)
- **Row count before/after** adding new seed rows
- `dbt run --debug` to see compiled SQL

### Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Nothing to do` | Partial parse cache thinks nothing changed | `dbt run --no-partial-parse` or `dbt clean` |
| `Object does not exist` for ephemeral | Space in `config(materialized = 'ephemeral')` | Remove spaces: `config(materialized='ephemeral')` |
| Models in wrong folder | Files in `models/example/` instead of `models/` | Drag files to `models/` directly |
| `dbt_project.yml` not picking up models | Extra indentation under `dbt_learning:` | Fix to `+materialized: view` at correct indent level |

### Critical Gotcha
```sql
--  Breaks silently
{{ config(materialized = 'ephemeral') }}

--  Correct
{{ config(materialized='ephemeral') }}
```

### Key Concepts
- `{{ this }}` refers to the model's own table in Snowflake
- `{% if is_incremental() %}` block skipped on first run, applied on subsequent runs
- Ephemeral models are inlined as CTEs — check `target/compiled/` to see this
- `dbt run --full-refresh` forces full rebuild of incremental models

---

## Exercise 3: Sources 

### What We Did
- Created `models/sources.yml` declaring `raw_customers` as a source
- Updated `stg_customers.sql` to use `{{ source() }}` instead of `{{ ref() }}`
- Ran `dbt source freshness`

### sources.yml (Final Version)
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
            description: "Primary Key"
          - name: first_name
            description: "Customer first name"
          - name: last_name
            description: "Customer last name"
          - name: email
            description: "Customer email address"
          - name: created_at
            description: "Record creation timestamp"
```

### ref() vs source()

| | `ref()` | `source()` |
|---|---|---|
| Use for | Model to model | Raw/external tables |
| Syntax | `{{ ref('model_name') }}` | `{{ source('source_name', 'table_name') }}` |
| Defined in | Automatically detected | `sources.yml` |

### Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Nothing to do` on freshness | Missing `loaded_at_field` | Add `loaded_at_field` to sources.yml |
| `Expected timestamp, received date` | `created_at` is date type not timestamp | Cast: `"cast(created_at as timestamp)"` |
| `ERROR STALE` | Data is older than `error_after` threshold | Expected — data is from 2024, threshold exceeded |
| Deprecation warning on `freshness` | dbt 1.11 moved freshness inside `config:` | Nest `loaded_at_field` and `freshness` under `config:` |
| VS Code showing 2 problems in sources.yml | dbt Power User extension schema validation lag | Ignore — tooling hasn't caught up with dbt 1.11 |

### Key Concepts
- `ERROR STALE` is correct behavior — means pipeline stopped delivering fresh data
- `freshness` and `loaded_at_field` must be inside `config:` block in dbt 1.11+
- Freshness thresholds should match your pipeline SLA in production

---

## Built-in Tests 

### Four Built-in Tests

| Test | What it checks |
|---|---|
| `unique` | No duplicate values in column |
| `not_null` | No null values in column |
| `accepted_values` | Column only contains specified values |
| `relationships` | Foreign key integrity between models |

### schema.yml (Final Version)
```yaml
version: 2

models:
  - name: stg_customers
    description: "Staged customer data"
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null
      - name: full_name
        tests:
          - not_null
      - name: email
        tests:
          - not_null
          - unique
      - name: status
        tests:
          - accepted_values:
              values: ['active', 'inactive']
```

### Added status Column to CSV
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

### Key Commands
```bash
dbt seed --full-refresh
dbt run --no-partial-parse --select stg_customers --full-refresh
dbt test --no-partial-parse
dbt test --store-failures  # saves failing rows to Snowflake audit table
```

### Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `invalid identifier STATUS` on seed | Column added after table already existed | `dbt seed --full-refresh` |
| Tests pass despite bad data in table | dbt tests are observability tools, not filters | Use `dbt build` or CI/CD gate to enforce |
| Deprecation warning on test syntax | dbt 1.11 changed test format | Cosmetic only, doesn't break tests |

### Critical Concept: Tests Don't Block Data
- `dbt run` loads ALL rows including invalid ones
- `dbt test` only reports pass/fail — does NOT delete or block rows
- Use `dbt build` in production (runs seed + run + test, stops pipeline on failure)
- Use `--store-failures` to save failing rows to a queryable audit table:
```sql
select * from dbt_db.dbt_test__audit.accepted_values_stg_customers_status;
```

---

## Documentation 

### What We Did
- Added rich descriptions to `schema.yml` for models and columns
- Generated and served dbt docs

### Key Commands
```bash
dbt docs generate --no-partial-parse
dbt docs serve
# Opens http://localhost:8080
# Press Ctrl+C to stop
```

### What the Docs UI Shows
- Full model catalog with descriptions
- Column-level descriptions and test definitions
- Lineage graph (DAG): `raw_customers → stg_customers → stg_customers_vw → final_customers`
- Source freshness status

### Best Practices
- Every model should have a `description`
- Every column should have a `description`
- Models without descriptions show up blank in the catalog
- Good documentation is a senior engineer habit — treat it like code

---

## Global Notes & Patterns

### Commands to Run Every Session
```bash
.\dbt_env\Scripts\activate
cd dbt_learning
dbt debug
```

### Most Used Flags
| Flag | Purpose |
|---|---|
| `--no-partial-parse` | Forces full reparse, fixes most "Nothing to do" errors |
| `--full-refresh` | Rebuilds tables/seeds from scratch |
| `--select model_name` | Runs only one specific model |
| `--store-failures` | Saves failing test rows to Snowflake |
| `--debug` | Shows compiled SQL and full execution details |

### Common "Nothing to do" Fix Sequence
```bash
dbt clean
dbt run --no-partial-parse --full-refresh
```

### Project Structure Best Practices
- Keep models directly in `models/` not `models/example/`
- Delete auto-generated example files after `dbt init`
- Clear `models/example/schema.yml` references to avoid warnings

---

*dbt Exercises 1–5 Documentation | King | February 2026*
