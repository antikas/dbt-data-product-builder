# dbt Project Layout - Default Structure

**Purpose:** Prescribed default directory structure for generated dbt artefacts. The generator follows the user's existing project layout when one exists. When no project exists, this is the default.

---

## Default Layout

```
dbt_project/
├── dbt_project.yml
├── packages.yml                          # dbt_utils, dbt_expectations if needed
├── profiles.yml                          # connection config (not generated)
│
├── models/
│   ├── sources/
│   │   └── _sources.yml                  # source definitions from batch_transfer config
│   │
│   ├── staging/                          # NOT generated - assumed to exist
│   │   ├── stg_{source_name}.sql         # user's existing staging models
│   │   └── _staging__models.yml
│   │
│   ├── intermediate/                     # generated intermediate models
│   │   ├── int_{entity}_mapped.sql       # from schema_transform
│   │   ├── int_{entity}_enriched.sql     # from data_enrichment (if separate)
│   │   ├── int_{entity}_calc.sql         # from calculated_fields (if separate)
│   │   ├── int_{entity}_filtered.sql     # from data_filtering (if separate)
│   │   ├── int_{entity}_curated.sql      # from data_curation
│   │   ├── int_{entity}_aggregated.sql   # from data_aggregation
│   │   ├── int_{entity}_quarantine.sql   # from data_validation (if quarantine enabled)
│   │   └── _int_{entity}__models.yml     # schema, tests, docs for intermediate models
│   │
│   └── curated/                       # generated curated models
│       ├── cur_{entity}.sql              # the published curated data product
│       └── _cur_{entity}__models.yml     # schema with contract, tests, meta, access
│
├── tests/
│   └── singular/
│       └── assert_{entity}_{rule}.sql    # custom singular tests from data_validation
│
├── macros/
│   └── quarantine_{entity}.sql           # quarantine macro (if quarantine enabled)
│
└── docs/
    └── config_model_mapping_{entity}.md  # lineage mapping doc from lineage_capture
```

---

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Source definition file | `_sources.yml` | `models/sources/_sources.yml` |
| Staging model | `stg_{source_name}.sql` | `stg_gp_quarterly_report.sql` |
| Intermediate model | `int_{entity}_{step_verb}.sql` | `int_fund_master_mapped.sql` |
| Curated model | `cur_{entity}.sql` | `cur_fund_master.sql` |
| Quarantine model | `int_{entity}_quarantine.sql` | `int_fund_master_quarantine.sql` |
| Schema YAML (intermediate) | `_int_{entity}__models.yml` | `_int_fund_master__models.yml` |
| Schema YAML (curated) | `_cur_{entity}__models.yml` | `_cur_fund_master__models.yml` |
| Singular test | `assert_{entity}_{rule}.sql` | `assert_fund_master_bridge_identity.sql` |
| Quarantine macro | `quarantine_{entity}.sql` | `quarantine_fund_master.sql` |
| Mapping doc | `config_model_mapping_{entity}.md` | `config_model_mapping_fund_master.md` |

---

## Adapting to Existing Projects

When generating into an existing dbt project, the generator:

1. **Reads `dbt_project.yml`** to discover model paths and naming conventions.
2. **Checks for existing `sources.yml`**. If sources are already defined, the generator references them rather than creating duplicates.
3. **Follows existing directory patterns**. If the project uses `models/marts/` instead of `models/curated/`, the generator uses `models/marts/`.
4. **Matches existing naming conventions**. If existing models use `dim_` prefix instead of `cur_`, the generator follows suit.
5. **Respects existing schema.yml structure**. Some projects use one schema.yml per directory, others per model. The generator matches the existing pattern.

The `dbt:` config block in the pipeline YAML allows explicit overrides for target_schema, model_prefix, and final_model_prefix.

---

## Schema-per-Layer Convention

| Layer | Default Schema | dbt_project.yml config |
|-------|---------------|----------------------|
| Sources | `raw` | Defined in source YAML |
| Staging | `staging` | `+schema: staging` under models/staging |
| Intermediate | `intermediate` | `+schema: intermediate` under models/intermediate |
| Curated | `curated` | `+schema: curated` under models/curated |

These are defaults. Override via the `dbt:` config block or by matching the existing project's conventions.

---

## Materialisation Defaults

| Layer | Default | Rationale |
|-------|---------|-----------|
| Staging | `view` | Lightweight, no storage, recomputed from raw |
| Intermediate | `view` | Intermediate steps not queried directly |
| Curated | `table` | Consumed by many downstream models, must be fast |
| Aggregation | `table` | Pre-computed summaries, expensive to recompute |
| Quarantine | `table` | Persists for investigation |

Override per-step via `materialization:` in the config, or globally via `dbt:` block.
