# dbt Summary & Reference

**Stack:** dbt Core 1.11.6 | Snowflake | Windows 11  
**Author:** King | **Date:** February 2026

---

##  dbt_project.yml Deep Dive 

### Key Configs

```yaml
models:
  dbt_learning:
    +materialized: view           # default for all models
    staging:
      +materialized: view
      +schema: staging            # builds into raw_staging in Snowflake
      +tags: ['staging', 'daily'] # run subsets with tag selector
      +meta:
        owner: 'king@taskus.com'
        team: 'analytics'
        priority: 'low'
    intermediate:
      +materialized: view
      +schema: intermediate
      +tags: ['intermediate', 'daily']
      +meta:
        owner: 'king'
        priority: 'medium'
    marts:
      +materialized: table
      +schema: marts
      +tags: ['marts', 'daily', 'reporting']
      +meta:
        owner: 'king'
        priority: 'high'

seeds:
  dbt_learning:
    Raw:
      +tags: ['seed', 'raw']
      raw_customers:
        +column_types:
          id: integer
          created_at: date
          status: varchar

vars:
  start_date: '2024-01-01'
  active_statuses: ['active', 'inactive']
  env: 'dev'
```

### Tag Selectors
```bash
dbt run --select tag:staging
dbt run --select tag:reporting
dbt run --select tag:daily
```

### Key Concepts
- `+` prefix = applies to all models in that folder and subfolders
- `+schema` creates a separate Snowflake schema per layer
- Snowflake prefixes default schema: `raw` + `staging` = `raw_staging`
- `meta` stores ownership info for alerting and routing
- `vars:` must be at top level — same indent as `models:` and `seeds:`
- Model-level `{{ config() }}` always overrides `dbt_project.yml`

---

##  Seeds Deep Dive 

### Final dbt_project.yml Seeds Config
```yaml
seeds:
  dbt_learning:
    Raw:
      +tags: ['seed', 'raw']
      raw_customers:
        +column_types:
          id: integer
          created_at: date
          status: varchar
      raw_orders:
        +column_types:
          order_id: integer
          customer_id: integer
          amount: float
          created_at: date
          status: varchar
      raw_products:
        +column_types:
          product_id: integer
          unit_price: float
          created_at: date
      status_mapping:
        +column_types:
          status_code: varchar
          status_label: varchar
          status_category: varchar
          is_active: boolean
```

### status_mapping.csv
```csv
status_code,status_label,status_category,is_active
active,Active Customer,customer,true
inactive,Inactive Customer,customer,false
pending,Pending Review,customer,false
completed,Order Completed,order,true
cancelled,Order Cancelled,order,false
```

### stg_customers_enriched.sql
```sql
{{ config(materialized='view') }}

select
    c.customer_id,
    c.full_name,
    c.email,
    c.status,
    s.status_label,
    s.is_active,
    c.created_at
from {{ ref('stg_customers') }} c
left join {{ ref('status_mapping') }} s
    on c.status = s.status_code
    and s.status_category = 'customer'
```

### Seeds: When to Use

| Use Seeds | Use Sources |
|---|---|
| Static lookup/reference data | Raw operational data |
| Config/mapping tables | Data loaded by pipelines |
| Small datasets < 1MB | Large frequently changing data |
| Analytics team owns it | Source system owns it |

### Seed Limitations
| Limitation | Impact | Fix |
|---|---|---|
| No incremental loading | Full reload every time | Use sources for large data |
| Schema changes need full-refresh | Can't alter existing table | `dbt seed --full-refresh` |
| Slow for large files | Not designed for big data | Keep seeds small |

### seed config controls
| Config | Purpose |
|---|---|
| `+column_types` | Enforce data types, prevent inference bugs |
| `+schema` | Control which Snowflake schema seed lands in |
| `+tags` | Run seeds by tag |
| `+enabled` | Toggle seed on/off without deleting file |

### Errors & Fixes
| Error | Cause | Fix |
|---|---|---|
| `raw_raw` schema | `+schema: raw` on seeds with default schema `raw` | Remove `+schema` from seeds config |
| `invalid identifier` on new column | Table exists without new column | `dbt seed --full-refresh` |

### dbt seed vs dbt seed --full-refresh
| Command | What happens | When to use |
|---|---|---|
| `dbt seed` | INSERT new rows into existing table | Data changed, structure same |
| `dbt seed --full-refresh` | DROP → CREATE → INSERT all rows | Structure changed (columns added/removed) |

---

## Jinja Basics 

### Four Jinja Patterns

#### 1. vars() — pull from dbt_project.yml
```sql
-- Jinja
where created_at >= '{{ var("start_date") }}'
and status in ({{ "'" + "','".join(var("active_statuses")) + "'" }})

-- Compiled SQL
where created_at >= '2024-01-01'
and status in ('active','inactive')
```

Override at runtime:
```bash
dbt run --select model_name --vars '{"start_date": "2024-01-05"}'
```

#### 2. if/else — conditional SQL
```sql
{% if var('env') == 'dev' %}
    'DEV_' || cast(order_id as varchar) as order_reference
{% else %}
    cast(order_id as varchar) as order_reference
{% endif %}
```

#### 3. for loop — dynamic column generation
```sql
{% set order_statuses = ['completed', 'pending', 'cancelled'] %}

{% for status in order_statuses %}
    sum(case when status = '{{ status }}' then amount else 0 end) as {{ status }}_amount
    {% if not loop.last %},{% endif %}
{% endfor %}
```

Compiled output:
```sql
sum(case when status = 'completed' then amount else 0 end) as completed_amount,
sum(case when status = 'pending' then amount else 0 end) as pending_amount,
sum(case when status = 'cancelled' then amount else 0 end) as cancelled_amount
```

#### 4. set — local variables
```sql
{% set high_value_threshold = 500 %}

case
    when total_spent >= {{ high_value_threshold }} then 'VIP'
    else 'Standard'
end as customer_tier
```

### Jinja Symbols Reference
| Symbol | Purpose |
|---|---|
| `{{ }}` | Output a value |
| `{% %}` | Logic block (if, for, set) |
| `{# #}` | Comment — not compiled |
| `var()` | Pull from dbt_project.yml vars |
| `loop.last` | True on last loop iteration |

### When to Use Jinja
| Pattern | Use Jinja? | Reason |
|---|---|---|
| Single hardcoded value |  No | Longhand cleaner |
| Value shared across many models |  Yes | Change once, updates everywhere |
| Different values per environment |  Yes | `--vars` at runtime |
| 3+ repetitive columns |  Yes | Loop saves time |
| 2 or fewer columns |  No | Longhand more readable |

### Errors & Fixes
| Error | Cause | Fix |
|---|---|---|
| `Required var '' not found` | `vars:` nested inside `seeds:` in dbt_project.yml | Move `vars:` to top level |
| Date var without quotes | `{{ var("start_date") }}` missing quotes | Use `'{{ var("start_date") }}'` |
| Trailing comma on last column | Missing `{% if not loop.last %}` check | Add comma guard in loop |

---

## ref() vs source() Deep Dive 

### The Simple Rule
```
Raw data coming IN from outside  →  source()
Models talking to each other     →  ref()

source() lives in  →  staging models only
ref() lives in     →  everywhere else
```

### How They Work

| | `source()` | `ref()` |
|---|---|---|
| Used for | Raw external tables | dbt models |
| Declared in | sources.yml | Auto-detected |
| Enables | Freshness testing | DAG ordering |
| Lives in | Staging only | Intermediate + Marts |

### The DAG
Every `ref()` and `source()` call draws an arrow in dbt's dependency map:
```
raw_customers (source)
      ↓ source()
stg_customers
      ↓ ref()
int_customer_orders
      ↓ ref()
mart_customer_summary
```
dbt reads this map and runs models in correct order automatically.

### + Selector — Cost Impact

| Selector | Models Run | Cost |
|---|---|---|
| `--select stg_orders` | 1 model | Cheapest |
| `--select stg_orders+` | stg_orders + all downstream | Medium |
| `--select +mart_customer_summary` | mart + all upstream | Medium |
| `--select +mart_customer_summary+` | entire chain | Expensive |
| `dbt run` | everything | Most expensive |

```bash
# run model + downstream
dbt run --select stg_orders+

# run model + upstream
dbt run --select +mart_customer_summary

# run full chain
dbt run --select +int_customer_orders+
```

### Production Cost Best Practices
| Situation | Command |
|---|---|
| Fixed one staging model | `--select model+` |
| Added new mart | `--select +new_mart` |
| Full nightly refresh | `dbt run` |
| Testing one model | `--select model_name` |
| Hotfix in production | `--select model+ --full-refresh` |

### ref() Environment Resolution
Same code — different environments — zero changes:
| Environment | ref('stg_customers') resolves to |
|---|---|
| dev | `dbt_db.raw_staging.stg_customers` |
| prod | `prod_db.staging.stg_customers` |

### Key Nuances
- `ref()` fails at **compile time** not runtime — catches broken references early
- Never hardcode schema names in SQL — always use `ref()` or `source()`
- Smart selectors can save thousands in Snowflake credits monthly
- `dbt compile --select model_name` shows what ref() resolves to without running

---

## Global Commands Reference 

```bash
# compile without running — see resolved SQL
dbt compile --select model_name --no-partial-parse

# run by tag
dbt run --select tag:staging
dbt run --select tag:reporting

# run with runtime vars
dbt run --select model_name --vars '{"start_date": "2024-01-05"}'

# run chains
dbt run --select model+          # model + downstream
dbt run --select +model          # model + upstream
dbt run --select +model+         # full chain

# seeds
dbt seed                         # load new rows
dbt seed --full-refresh          # drop and recreate
dbt seed --select seed_name      # one seed only

# verify compiled SQL
type target\compiled\dbt_learning\models\staging\model_name.sql
```

---

*dbt Summary | King | February 2026*
