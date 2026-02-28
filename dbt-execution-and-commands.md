# dbt Order of Execution & Bash Commands Reference

**Stack:** dbt Core 1.11.6 | Snowflake | Windows 11  
**Author:** King | **Date:** February 2026

---

## Order of Execution for New Table Creation

Every time you create a new model in dbt, follow this sequence:

```
1. Create the CSV (if new raw data needed)
        ↓
2. Add to sources.yml (if referencing raw table)
        ↓
3. Create the .sql model file
        ↓
4. Add to schema.yml (descriptions + tests)
        ↓
5. dbt seed --full-refresh (if CSV changed)
        ↓
6. dbt run --no-partial-parse --select model_name
        ↓
7. dbt test --no-partial-parse
        ↓
8. Verify in Snowflake
```

---

## In Practice

### Scenario 1: Brand new raw data
```bash
# 1. Create seeds/Raw/raw_orders.csv
# 2. Add raw_orders to sources.yml
# 3. Create models/stg_orders.sql
# 4. Add stg_orders to schema.yml

dbt seed --full-refresh
dbt run --no-partial-parse --select stg_orders
dbt test --no-partial-parse
```

### Scenario 2: New model from existing data
```bash
# 1. Create models/mart_customers.sql using ref()
# 2. Add mart_customers to schema.yml

dbt run --no-partial-parse --select mart_customers
dbt test --no-partial-parse
```

### Scenario 3: Modified existing model
```bash
# 1. Edit the .sql file
# 2. Update schema.yml if columns changed

dbt run --no-partial-parse --select model_name --full-refresh
dbt test --no-partial-parse
```

---

## Full Bash Command Reference in Context

### Session Start (run every time)
```bash
.\dbt_env\Scripts\activate        # activate Python environment
cd dbt_learning                   # navigate to project
dbt debug                         # confirm Snowflake connection
```

### Building Models
```bash
dbt run                                           # run all models
dbt run --select model_name                       # run one model
dbt run --select model_name --full-refresh        # force full rebuild
dbt run --no-partial-parse                        # force full reparse
dbt run --no-partial-parse --full-refresh         # fix most "nothing to do" errors
dbt run --select model_a model_b                  # run multiple specific models
dbt run --select +model_name                      # run model + all upstream dependencies
dbt run --select model_name+                      # run model + all downstream dependents
```

### Seeds
```bash
dbt seed                          # load CSVs into Snowflake
dbt seed --full-refresh           # drop and recreate seed tables (use when columns change)
dbt seed --select seed_name       # load one specific seed
```

### Testing
```bash
dbt test                                          # run all tests
dbt test --no-partial-parse                       # run all tests with full reparse
dbt test --select model_name                      # test one model only
dbt test --store-failures                         # save failing rows to Snowflake audit table
```

### Sources
```bash
dbt source freshness                              # check if raw data is fresh
dbt source freshness --no-partial-parse           # same with full reparse
```

### Documentation
```bash
dbt docs generate                                 # build the docs site
dbt docs serve                                    # open docs at localhost:8080
dbt docs generate --no-partial-parse              # generate with full reparse
```

### Listing & Inspection
```bash
dbt ls                                            # list all models dbt detects
dbt ls --select model_name                        # check if specific model is detected
dbt compile                                       # compile SQL without running
dbt compile --select model_name                   # compile one model to check SQL output
```

### Debugging
```bash
dbt debug                                         # test Snowflake connection
dbt run --debug                                   # show full SQL + execution details
dbt clean                                         # delete target/ and dbt_packages/ folders
dbt parse                                         # parse project without running
```

### The Nuclear Option (fixes almost everything)
```bash
dbt clean
dbt run --no-partial-parse --full-refresh
```

---

## Decision Tree: Which Command to Use?

```
New model created?
├── Yes → dbt run --no-partial-parse --select model_name

Column added to CSV?
├── Yes → dbt seed --full-refresh → dbt run --no-partial-parse --full-refresh

"Nothing to do" error?
├── Yes → dbt clean → dbt run --no-partial-parse --full-refresh

Tests failing?
├── Yes → dbt test --store-failures → query audit table in Snowflake

Connection issues?
└── Yes → dbt debug → check profiles.yml
```

---

*dbt Order of Execution & Bash Commands Reference | King | February 2026*
