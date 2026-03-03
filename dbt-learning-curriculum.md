# dbt Core + Snowflake — Full Learning Curriculum

**Prepared by:** King  
**Date:** February 2026  
**Stack:** dbt Core 1.11.6 | Snowflake | Azure

---

##  Beginner

| # | Topic | What You'll Learn |
|---|---|---|
| 1 | Your First Model | ref(), seeds, dbt run basics |
| 2 | Materializations | view, table, incremental, ephemeral |
| 3 | Sources | source(), sources.yml, freshness testing |
| 4 | Tests — Built-in | not_null, unique, accepted_values, relationships |
| 5 | Documentation | descriptions, schema.yml, dbt docs serve |
| 6 | Project Structure | staging, intermediate, mart layer conventions |
| 7 | dbt_project.yml Deep Dive | configs, tags, meta, model paths |
| 8 | Seeds Deep Dive | column types, seed configs, when to use vs not use |
| 9 | Jinja Basics | variables, if/else, loops in SQL |
| 10 | ref() vs source() | when to use each, lineage best practices |

---

##  Intermediate

| # | Topic | What You'll Learn |
|---|---|---|
| 11 | Snapshots | SCD Type 2, tracking historical changes |
| 12 | Macros | reusable SQL functions, arguments, calling macros |
| 13 | dbt Packages | dbt-utils, dbt-expectations, installing and using |
| 14 | Hooks | pre-hook, post-hook, on-run-start, on-run-end |
| 15 | Custom Generic Tests | writing your own reusable tests |
| 16 | Custom Singular Tests | one-off SQL tests for specific business rules |
| 17 | Exposures | documenting dashboards and downstream consumers |
| 18 | Tags & Selectors | running subsets of models, tagging strategies |
| 19 | Variables | dbt vars, passing variables at runtime |
| 20 | Analyses | ad-hoc SQL in dbt, when to use analyses folder |

---

##  Advanced

| # | Topic | What You'll Learn |
|---|---|---|
| 21 | Incremental Strategies Deep Dive | append, merge, delete+insert, insert_overwrite |
| 22 | Advanced Jinja | custom macros with loops, dynamic SQL generation |
| 23 | dbt Metadata & Artifacts | manifest.json, run_results.json, catalog.json |
| 24 | Performance Tuning | clustering keys, warehouse sizing, query optimization |
| 25 | Testing Limitations | what dbt tests can't catch, gaps to watch for |
| 26 | Advanced Snapshots | hard deletes, custom strategies, snapshot testing |
| 27 | Contracts & Constraints | model contracts, column-level constraints, versioning |
| 28 | Model Versioning | versioned models, deprecation, breaking changes |
| 29 | Multi-environment Setup | dev/staging/prod profiles, CI/CD patterns |
| 30 | dbt at Scale | large project patterns, modularity, monorepo vs multi-repo |
| 31 | Edge Cases & Gotchas | common pitfalls senior engineers hit in production |
| 32 | Testing dbt's Limits | stress testing, breaking things intentionally |

---

