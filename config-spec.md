# dbt FDP Build - YAML Config Specification

**Created:** 2026-03-25
**Status:** Active
**Purpose:** SSOT for the YAML config schema accepted by the dbt FDP Build generator prompt

---

## Relationship to the Data Product Framework

This config schema uses the **exact top-level structure** from the Data Product Framework DESIGN.md §4.1:

```
pipeline:    → product identity and metadata
steps:       → ordered sequence of pattern configurations
checkpointing: → retry and incremental strategy
```

The framework schema uses file references (`rules_file:`, `mapping_file:`, `contract_file:`) because it targets multiple platforms. This dbt-specific pack **expands those references inline** - you define mappings, rules, and contracts directly in the config rather than pointing to separate files. This makes the config self-contained for prompt-based generation.

**Expression language:** This pack uses SQL expressions (standard SQL, not dialect-specific). The framework's neutral expression language (DESIGN.md §8, decision D9) remains an open design problem - this dbt pack does not resolve or change that. SQL is used here because the target is dbt, which is SQL-native.

---

## Full Config Schema

### Top Level

```yaml
pipeline:
  name: string                           # required - product identifier, e.g. "foundation.fund_master"
  version: string                        # required - semver, e.g. "1.0.0"
  product_type: "foundation-base"        # required - fixed for this pack
  owner: string                          # required - team or person, e.g. "investment-data-team"
  domain: string                         # required - business domain, e.g. "investments"
  description: string                    # required - human-readable purpose

  # dbt-specific output configuration
  dbt:
    project_name: string                 # optional - existing dbt project name (for integration)
    target_schema: string                # default: "intermediate" - schema for intermediate models
    foundation_schema: string            # default: "foundation" - schema for final foundation model
    model_prefix: string                 # default: "int" - prefix for intermediate models
    final_model_prefix: string           # default: "fnd" - prefix for the final foundation model
    materialization: string              # default: "view" - default for intermediate models
                                         #   view: lightweight, no storage cost, recomputed on query
                                         #   table: persisted, faster reads, higher storage
                                         #   incremental: append/merge, for large or growing datasets
                                         #   ephemeral: CTE only, not queryable directly
    final_materialization: string        # default: "table" - for the foundation model
                                         #   table recommended: foundation products are consumed widely
                                         #   incremental: when dataset is large and has a reliable update key
    tags: [string]                       # optional - dbt tags applied to all generated models
    grants: {}                           # optional - dbt grants config, e.g. { select: ["analyst_role"] }

steps: [StepConfig]                      # required - ordered pattern sequence (see below)

checkpointing: CheckpointConfig          # optional - incremental and retry strategy (see below)
```

---

### Pattern: batch_transfer

**Purpose:** Declares the source tables that feed this pipeline. In dbt, this translates to `sources.yml` entries - no model is generated.

```yaml
- pattern: "batch_transfer"
  config:
    sources:
      - name: string                     # required - source name for source() macro
        database: string                 # optional - database name if cross-database
        schema: string                   # required - raw/staging schema name
        table: string                    # required - raw table name
        description: string              # required - flows into sources.yml documentation
        loaded_at_field: string          # optional - column for freshness checks
        freshness:                       # optional - dbt source freshness config
          warn_after:
            count: integer
            period: string               # "hour" | "day" | "week"
          error_after:
            count: integer
            period: string
        columns:                         # optional but recommended - source column documentation
          - name: string
            description: string
            data_type: string            # optional - for documentation only at source level
            tests: [string]              # optional - source-level tests, e.g. ["not_null", "unique"]
```

**Notes:**
- Each source entry maps to one `source()` reference used by downstream models.
- If your dbt project already has `sources.yml` for these tables, the generator will reference the existing sources rather than duplicating definitions.
- `loaded_at_field` enables `dbt source freshness` checks - recommended for production monitoring.

---

### Pattern: schema_transform

**Purpose:** Maps source fields to the canonical foundation schema. Structural mapping only - renaming, type casting, null handling, dropping unwanted fields. No business logic.

```yaml
- pattern: "schema_transform"
  config:
    source_ref: string                   # required - which batch_transfer source to map from
                                         #   Use the source name, e.g. "gp_quarterly_report"
                                         #   For multiple sources, use one schema_transform per source
                                         #   and combine via data_curation
    model_name: string                   # optional - override output model name
                                         #   default: "{model_prefix}_{entity}_mapped"
    materialization: string              # optional - override pipeline-level default

    mappings:                            # required - field-level mapping spec
      - source_field: string             # required - field name in source table
        target_field: string             # required - field name in output (canonical name)
        data_type: string                # required - target SQL data type
                                         #   Standard types: varchar, integer, bigint, decimal(p,s),
                                         #   boolean, date, timestamp, float, double
        cast_expression: string          # optional - custom SQL CAST/conversion expression
                                         #   When omitted, generator uses: CAST(source_field AS data_type)
                                         #   Use for non-trivial conversions:
                                         #     "TO_DATE(report_period, 'YYYY-MM-DD')"
                                         #     "NULLIF(TRIM(raw_status), '')"
                                         #     "COALESCE(amount_usd, amount_local * fx_rate)"
        nullable: boolean                # default: true - whether NULL is allowed
        description: string              # required - flows into schema.yml column description
        pii: boolean                     # default: false - tagged in meta for governance

    drop_fields: [string]                # optional - source fields explicitly not carried forward
                                         #   Documenting drops is better than silently omitting

    null_handling:                        # optional - default null behavior for the transform
      strategy: string                   # "propagate" (default) | "default"
                                         #   propagate: NULLs flow through as-is
                                         #   default: apply defaults per field below
      defaults:                          # required if strategy is "default"
        - field: string                  # target field name
          value: string                  # SQL expression for default value
                                         #   e.g. "'UNKNOWN'", "0", "CURRENT_DATE"

    deduplication:                        # optional - remove duplicate source records
      enabled: boolean                   # default: false
      keys: [string]                     # fields to deduplicate on (source field names)
      strategy: string                   # "latest" | "first" | "custom"
                                         #   latest: keep most recent by order_field
                                         #   first: keep earliest by order_field
                                         #   custom: use custom_expression
      order_field: string                # required for latest/first - field to order by
      custom_expression: string          # required for custom - SQL ROW_NUMBER() expression
```

**Notes:**
- Schema transform is **structural only**. If you need to compute derived values, use `calculated_fields`. If you need to join reference data, use `data_enrichment`.
- One `schema_transform` per source. If your FDP combines multiple sources, use separate transforms and combine them in `data_curation`.
- The `deduplication` section handles source duplicates. This is common in CDC-based extracts where multiple versions of a record arrive.

---

### Pattern: calculated_fields

**Purpose:** Derive new fields from business rules applied to existing columns. Rules are SQL expressions - no joins, no lookups, no external dependencies. For anything requiring reference data, use `data_enrichment`.

```yaml
- pattern: "calculated_fields"
  config:
    apply_mode: string                   # "inline" (default) | "separate"
                                         #   inline: adds calculated fields as CTEs in the
                                         #     preceding model (schema_transform or enrichment)
                                         #   separate: generates a new model file
    model_name: string                   # optional - only used when apply_mode is "separate"
                                         #   default: "{model_prefix}_{entity}_calc"
    materialization: string              # optional - only used when apply_mode is "separate"

    fields:
      - name: string                     # required - output field name
        expression: string               # required (unless escape_to_code) - SQL expression
                                         #   Can reference any field from upstream models
                                         #   Examples:
                                         #     "first_name || ' ' || last_name"
                                         #     "CASE WHEN status IN ('A','P') AND end_date IS NULL THEN true ELSE false END"
                                         #     "DATE_PART('year', AGE(CURRENT_DATE, inception_date))"
                                         #     "committed_capital - called_capital"
                                         #     "ROUND(nav / NULLIF(called_capital, 0), 4)"
        data_type: string                # required - output data type
        description: string              # required - business meaning of this field
        nullable: boolean                # default: true

        # Escape to code - for complex logic that doesn't fit a SQL expression
        escape_to_code: boolean          # default: false
        macro_name: string               # required if escape_to_code - dbt macro to call
                                         #   The macro must exist in your dbt project's macros/ directory
                                         #   Generator will produce: {{ macro_name(field_args) }}
        macro_args: {}                   # optional - arguments to pass to the macro
                                         #   e.g. { "amount_field": "nav", "currency_field": "currency" }

    # Rule versioning metadata - optional but recommended
    rule_version: string                 # e.g. "2026Q1" - when these rules were last updated
    rule_owner: string                   # who is accountable for these business rules
```

**Notes:**
- Calculated fields must be **idempotent**: same input always produces the same output.
- Expressions must not reference other calculated fields defined in the same step. If field B depends on field A, either: (a) put A's expression inline in B's expression, or (b) use two separate `calculated_fields` steps with A before B.
- `escape_to_code` is the mechanism for genuinely complex logic (multi-step algorithms, statistical calculations, ML feature computations). The macro is tested independently - the generator only produces the call.

---

### Pattern: data_enrichment

**Purpose:** Join incoming records against reference datasets to resolve codes, add attributes, or expand lookup values. Reference datasets should be governed data products (or at minimum, version-controlled reference tables).

```yaml
- pattern: "data_enrichment"
  config:
    apply_mode: string                   # "inline" (default) | "separate"
    model_name: string                   # optional - for apply_mode "separate"
    materialization: string              # optional - for apply_mode "separate"

    enrichments:
      - name: string                     # required - descriptive name, e.g. "strategy_lookup"
        reference_model: string          # required - dbt ref() or source() target
                                         #   e.g. "ref('dim_fund_strategies')" or "source('ref', 'regions')"
                                         #   Use ref() for models in the same dbt project
                                         #   Use source() for external reference tables
        join_type: string                # default: "left"
                                         #   "left": keep all source records (nulls for no match)
                                         #   "inner": drop records with no match (use with caution)
        join_keys:                       # required - supports composite keys
          - source_field: string         # field in the main model
            reference_field: string      # field in the reference table
        fields:                          # required - fields to bring from reference
          - source_field: string         # field name in the reference table
            target_field: string         # optional - rename in output (defaults to source_field)
            description: string          # required - what this enriched field means
        fallback:                        # optional - what to do when lookup misses
          strategy: string               # "null" (default) | "default" | "warn"
                                         #   null: leave enriched fields as NULL
                                         #   default: substitute default values
                                         #   warn: leave as NULL + add a boolean flag column
          default_values:                # required if strategy is "default"
            - field: string
              value: string              # SQL expression for default
          warn_flag_column: string       # required if strategy is "warn"
                                         #   Name of the boolean flag column
                                         #   e.g. "strategy_lookup_missing"

        # Temporal join - for point-in-time correct enrichment
        temporal:                         # optional - use when reference data changes over time
          enabled: boolean               # default: false
          valid_from_field: string       # field in reference table for validity start
          valid_to_field: string         # field in reference table for validity end
                                         #   Use NULL or '9999-12-31' for current records
          event_date_field: string       # field in main model to match against validity range
```

**Notes:**
- `join_type: "inner"` drops records. If used, the generator will add an audit comment showing record count before and after the join. Prefer `"left"` with a fallback strategy.
- Temporal joins are critical for historical accuracy. If you enrich with data that changes over time (pricing, classifications, exchange rates), always use the temporal config. Without it, you get the current reference value for historical events - producing incorrect historical analysis.
- Each enrichment generates a CTE in the model (when inline) or a model in the chain (when separate).

---

### Pattern: data_filtering

**Purpose:** Exclude or flag records based on explicit predicates. Every exclusion is documented and auditable - no silent WHERE clauses buried in SQL.

```yaml
- pattern: "data_filtering"
  config:
    apply_mode: string                   # "inline" (default) | "separate"
    model_name: string                   # optional - for apply_mode "separate"

    filters:
      - name: string                     # required - descriptive name, e.g. "exclude_test_funds"
        predicate: string                # required - SQL WHERE clause fragment
                                         #   Written as the INCLUSION condition (records to keep)
                                         #   e.g. "fund_id NOT LIKE 'TEST%'"
                                         #   e.g. "investment_date >= '2015-01-01'"
                                         #   e.g. "status != 'DELETED'"
        description: string              # required - WHY this filter exists (audit trail)
        severity: string                 # "exclude" (default) | "flag"
                                         #   exclude: records failing predicate are dropped
                                         #   flag: records failing predicate are kept but flagged

    audit:                                # optional but recommended
      log_counts: boolean                # default: true
                                         #   When true, generator adds a comment block showing:
                                         #   "-- Filter: {name} - {description}"
                                         #   "-- Records before: {{ count_before }}"
                                         #   "-- Records excluded: {{ count_excluded }}"
      flag_column: string                # required if any filter has severity "flag"
                                         #   Name of the boolean column added to output
                                         #   e.g. "is_filtered" or "filter_flag"
      flag_detail_column: string         # optional - column with the name of the filter that flagged
                                         #   Useful when multiple filters use "flag" severity
```

**Notes:**
- Predicates are written as **inclusion** conditions (what to keep). The generator applies them as WHERE clauses.
- `severity: "flag"` keeps the record but adds a boolean column. This is useful for soft exclusions where downstream consumers decide whether to include flagged records.
- `log_counts: true` adds audit comments to the SQL. In production, you may want to materialize counts to a monitoring table - that's a custom extension beyond this pack.

---

### Pattern: data_validation (pre and post)

**Purpose:** Detect and handle data that violates quality constraints. Pre-validation checks source data before transformation. Post-validation checks the foundation model output.

```yaml
- pattern: "data_validation"
  stage: string                          # required - "pre" or "post"
                                         #   pre: validates source/staging inputs
                                         #   post: validates the final foundation model

  config:
    rules:
      - field: string                    # required for field-level rules, omit for cross-field
        type: string                     # required - rule type (see table below)
        severity: string                 # required - "error" | "warn"
                                         #   error: failing tests produce dbt test failures
                                         #   warn: failing tests produce dbt warnings
        description: string              # required - what this rule checks and why
        threshold: number                # optional - for completeness/distribution rules
                                         #   e.g. 0.95 = "95% of records must pass"
        params: {}                       # type-specific parameters (see table below)

    quarantine:                           # optional - route failing records to quarantine model
      enabled: boolean                   # default: false
      quarantine_model: string           # default: "quarantine_{entity}" - model name
      error_threshold: number            # abort pipeline if error rate exceeds this
                                         #   e.g. 0.001 = abort if >0.1% errors
      warn_threshold: number             # flag if warn rate exceeds this
```

**Validation rule types:**

| Type | Field required? | Params | dbt test mapping |
|------|----------------|--------|-----------------|
| `completeness` | yes | `threshold` (proportion, default 1.0) | `not_null` or custom threshold test |
| `format` | yes | `pattern` (regex) | `dbt_expectations.expect_column_values_to_match_regex` or custom |
| `range` | yes | `min`, `max` (number or date) | `dbt_utils.accepted_range` or custom |
| `referential` | yes | `to` (model ref), `field` (FK field) | `relationships` test |
| `uniqueness` | yes | `combination_of_columns` (optional list) | `unique` or `dbt_utils.unique_combination_of_columns` |
| `accepted_values` | yes | `values` (list of allowed values) | `accepted_values` |
| `distribution` | no | `metric` ("record_count"), `tolerance_pct`, `baseline` | Custom singular test |
| `consistency` | no | `expression` (SQL boolean) | Custom singular test |
| `custom_sql` | no | `sql` (full SQL test query) | Singular test file |

**Quarantine model:**
When `quarantine.enabled: true`, the generator creates an additional model that captures records failing error-severity rules. The quarantine model includes:
- All original record fields
- `_quarantine_rule`: name of the failing rule
- `_quarantine_severity`: error or warn
- `_quarantine_timestamp`: when the record was quarantined
- `_quarantine_field`: which field failed (if applicable)

The main model filters these records out, keeping only passing records.

**Notes:**
- Pre-validation targets source models (via `source()` or `ref()` to staging).
- Post-validation targets the final foundation model.
- `dbt_expectations` and `dbt_utils` packages are used for advanced test types. The generator will note required packages in the output.

---

### Pattern: data_aggregation

**Purpose:** Produce pre-computed summaries from validated data. Aggregation logic is explicit - every grouping dimension and aggregate measure is declared.

```yaml
- pattern: "data_aggregation"
  config:
    model_name: string                   # optional - default: "{model_prefix}_{entity}_aggregated"
    materialization: string              # default: "table" - aggregates are typically materialized
    description: string                  # required - what this aggregation represents

    group_by: [string]                   # required - fields to group by
                                         #   e.g. ["fund_id", "reporting_quarter"]

    aggregates:                          # required - aggregate expressions
      - name: string                     # required - output field name
        expression: string               # required - SQL aggregate expression
                                         #   e.g. "SUM(nav_amount)", "COUNT(*)", "AVG(return_pct)"
                                         #   e.g. "SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END)"
        data_type: string                # required - output data type
        description: string              # required - what this measure represents

    window_functions:                    # optional - window expressions (non-aggregating)
      - name: string                     # required - output field name
        expression: string               # required - SQL window expression
                                         #   e.g. "ROW_NUMBER() OVER (PARTITION BY fund_id ORDER BY quarter DESC)"
                                         #   e.g. "LAG(nav_amount) OVER (PARTITION BY fund_id ORDER BY quarter)"
                                         #   e.g. "SUM(nav_amount) OVER (PARTITION BY strategy ORDER BY quarter ROWS BETWEEN 3 PRECEDING AND CURRENT ROW)"
        data_type: string
        description: string

    having: string                       # optional - SQL HAVING clause
                                         #   e.g. "COUNT(*) > 1" to exclude groups with single records

    late_arrival:                         # optional - for incremental aggregations
      strategy: string                   # "recompute" | "ignore"
                                         #   recompute: re-aggregate affected periods on each run
                                         #   ignore: only aggregate new periods
      lookback_periods: integer          # how many periods to recompute
                                         #   e.g. 3 = recompute the last 3 quarters
      period_field: string               # the date/period field to use for lookback
```

**Notes:**
- Aggregation models are almost always materialized as tables (or incremental). Views would recompute on every query.
- `window_functions` are applied after `group_by` aggregation in a separate CTE. This allows you to compute things like "running total" or "previous period's NAV" on the aggregated results.
- `late_arrival` is only relevant for incremental materializations. It defines how many periods to recompute to account for late-arriving data.

---

### Pattern: data_curation

**Purpose:** Resolve the same real-world entity appearing in multiple sources into a single golden record. For Foundation Base, this is the final step that produces the canonical entity.

```yaml
- pattern: "data_curation"
  config:
    model_name: string                   # optional - default: "{model_prefix}_{entity}_curated"
    materialization: string              # optional

    entity_key: string                   # required - the canonical business key after resolution
                                         #   e.g. "fund_id", "customer_id"
                                         #   This key is unique in the output

    match_keys:                          # required - how to match records across sources
      - source: string                   # required - ref() to a source-specific model
                                         #   e.g. "int_fund_master_mapped_gp"
        keys: [string]                   # required - fields to match on (deterministic)
                                         #   e.g. ["fund_id"] for exact match
                                         #   e.g. ["fund_name", "vintage_year"] for composite
        priority: integer                # required - source precedence (lower = higher priority)
                                         #   Priority 1 is the most trusted source

    resolution_strategy: string          # "deterministic" - only supported strategy
                                         #   Matches on declared keys only
                                         #   No probabilistic/fuzzy matching

    survivorship:                        # required - how to resolve attribute conflicts
      default_priority: [string]         # required - ordered list of source refs
                                         #   First source with a non-null value wins
                                         #   e.g. ["int_fund_gp_mapped", "int_fund_admin_mapped"]
      field_overrides:                   # optional - per-field priority overrides
        - field: string                  # the target field name
          priority: [string]             # ordered source refs for this specific field
          rule: string                   # "first_non_null" (default) | "most_recent" | "source_priority"
                                         #   first_non_null: first source in priority list with a value
                                         #   most_recent: source with the latest update timestamp
                                         #   source_priority: always use the first source, even if null

      # When most_recent is used, specify the timestamp field per source
      recency_fields: {}                 # optional - source_ref → timestamp field name
                                         #   e.g. { "int_fund_gp_mapped": "gp_report_date",
                                         #          "int_fund_admin_mapped": "admin_update_date" }

    # Escape to code - for fuzzy/probabilistic matching
    fuzzy_matching:                       # optional - documented but NOT implemented by generator
      enabled: boolean                   # default: false
      note: string                       # guidance: "Implement as a custom dbt macro or Python model.
                                         #   The macro should accept match keys and return a mapping
                                         #   table of source_id → canonical_id with confidence scores.
                                         #   Reference the macro in the data_curation step."
      macro_name: string                 # the dbt macro to call (must exist in macros/)
```

**Notes:**
- `data_curation` is required when your FDP has multiple sources for the same entity. If you have a single source, skip this pattern - `schema_transform` is sufficient.
- `match_keys` define how records are joined across sources. For deterministic matching, keys must be exact. If the same entity has different IDs in different sources, you need an ID mapping table (modelled as a reference via `data_enrichment`).
- `survivorship` resolves conflicts when two sources provide different values for the same field. The most common rule is `first_non_null` with source priority.
- Fuzzy matching (Levenshtein, Soundex, probabilistic) is explicitly out of scope for the generator. When you need it, write a custom dbt macro or Python model and reference it via `macro_name`.

---

### Pattern: data_contracts

**Purpose:** Define the published contract for the foundation data product. This is what downstream consumers depend on. In dbt, this maps to model contracts (enforced schema) and column-level constraints.

```yaml
- pattern: "data_contracts"
  config:
    contract_enforced: boolean           # default: true
                                         #   When true, dbt enforces the schema at build time
                                         #   Downstream consumers can depend on exact columns/types
    description: string                  # required - contract-level description

    columns:                             # required - full contract column specification
      - name: string                     # required - column name (must match model output)
        data_type: string                # required - dbt contract data type
                                         #   Must use the target warehouse's type syntax
                                         #   e.g. "varchar(255)", "integer", "decimal(18,4)",
                                         #   "boolean", "date", "timestamp"
        description: string              # required - business meaning
        constraints:                     # optional - dbt column constraints
          - type: string                 # "not_null" | "unique" | "primary_key" |
                                         #   "foreign_key" | "check" | "accepted_values"
            expression: string           # required for "check" - SQL boolean expression
                                         #   e.g. "vintage_year >= 1990 AND vintage_year <= 2030"
            to: string                   # required for "foreign_key" - ref model
            to_columns: [string]         # required for "foreign_key" - ref columns
            values: [string]             # required for "accepted_values"
            warn_unenforced: boolean     # optional - dbt will warn if warehouse can't enforce
        meta:                            # optional - dbt meta tags per column
          pii: boolean
          sensitivity: string            # "public" | "internal" | "confidential"
          source_system: string          # which source this field originated from
          business_owner: string
          # arbitrary additional key-value pairs allowed
```

**Notes:**
- dbt model contracts are enforced at build time. If the model output doesn't match the contract, `dbt build` fails. This is the strongest guarantee you can provide to consumers.
- `primary_key` constraint implies both `not_null` and `unique`. If you specify `primary_key`, you don't need separate `not_null` and `unique` constraints on the same column.
- `foreign_key` constraints reference other models in the dbt project. In most warehouses these are not enforced at the database level but are documented and can be tested.
- `meta` tags are visible in dbt docs and the manifest. Use them for governance metadata (PII flags, sensitivity classification, ownership).

---

### Pattern: lineage_capture

**Purpose:** Record the dependency graph. In dbt, lineage is captured natively through `ref()` and `source()` calls. This config adds an explicit mapping document.

```yaml
- pattern: "lineage_capture"
  config:
    generate_mapping_doc: boolean        # default: true
                                         #   Generates a config-model-mapping document that traces
                                         #   each config step to the model/CTE that implements it
    include_field_lineage: boolean       # default: true
                                         #   When true, the mapping doc includes field-level traces:
                                         #   source_field - transform step - output_field
```

**Notes:**
- dbt's native lineage (DAG) is always generated regardless of this config. This pattern adds an explicit audit document linking config steps to generated artefacts.
- The mapping doc is a markdown file, not a dbt artefact. It's for human review and audit.

---

### Pattern: metadata_capture

**Purpose:** Ensure every generated model and column is documented and tagged for discovery. In dbt, this means complete schema.yml entries with descriptions, meta tags, and documentation.

```yaml
- pattern: "metadata_capture"
  config:
    meta_tags:                           # optional - applied to all generated models
      domain: string                     # business domain, e.g. "investments"
      owner: string                      # team or person
      tier: string                       # "tier1" | "tier2" | "tier3" - data product tier
      pii_contains: boolean              # whether any column contains PII
      sla: string                        # e.g. "daily by 06:00 UTC"
      custom: {}                         # arbitrary additional key-value pairs
    doc_blocks:                          # optional - reusable documentation blocks
      - name: string                     # doc block name, e.g. "fund_master_overview"
        content: string                  # markdown content for the doc block
```

**Notes:**
- Every column description comes from the pattern configs (schema_transform, enrichment, contracts). This pattern adds model-level metadata.
- `doc_blocks` generate dbt `{% docs %}` blocks that can be referenced in schema.yml descriptions.

---

### Pattern: schema_publish

**Purpose:** Control visibility and versioning of the foundation model. In dbt, this maps to access modifiers, model versions, and documentation.

```yaml
- pattern: "schema_publish"
  config:
    access: string                       # "public" (default) | "protected" | "private"
                                         #   public: any dbt project can ref() this model
                                         #   protected: only the same dbt project can ref()
                                         #   private: only models in the same group
    group: string                        # optional - dbt model group for access control
    version: integer                     # optional - dbt model version number
                                         #   When set, generates a versioned model definition
                                         #   e.g. {{ ref('fnd_fund_master', v=1) }}
    deprecation_date: string             # optional - ISO date when this version is deprecated
                                         #   Only relevant when version is set
    generate_version_manifest: boolean   # default: false
                                         #   When true, generates a separate version manifest file
                                         #   documenting schema history, breaking changes, and
                                         #   consumer migration guidance
                                         #   Recommended for dbt mesh environments
```

---

### Pattern: data_publish

**Purpose:** Guidance for making the foundation product available to consumers. In dbt, publication happens via `dbt build` and access controls.

```yaml
- pattern: "data_publish"
  config:
    build_command_guidance: boolean      # default: true
                                         #   Includes a comment block in the final model with
                                         #   the recommended dbt build command and options
    access_controls:                     # optional - downstream access documentation
      groups: [string]                   # dbt groups that can access this model
      roles: [string]                    # warehouse roles with SELECT grants
      notes: string                      # human-readable access guidance
    post_hook: string                    # optional - dbt post-hook SQL
                                         #   e.g. "GRANT SELECT ON {{ this }} TO ROLE analyst"
```

---

### Checkpointing

**Purpose:** Incremental processing strategy and retry guidance. In dbt, this maps to incremental model configuration and `dbt retry`.

```yaml
checkpointing:
  enabled: boolean                       # default: true
  strategy: string                       # "incremental" | "full_refresh"
                                         #   incremental: models use dbt incremental materialisation
                                         #   full_refresh: models rebuild completely each run

  incremental:                           # required when strategy is "incremental"
    unique_key: string | [string]        # required - column(s) for merge/upsert
                                         #   Single key: "fund_id"
                                         #   Composite: ["fund_id", "reporting_quarter"]
    strategy: string                     # "merge" (default) | "delete+insert" | "append"
                                         #   merge: MERGE statement, updates existing + inserts new
                                         #   delete+insert: DELETE matching keys, then INSERT all
                                         #   append: INSERT only, no updates (log/event tables)
    on_schema_change: string             # "append_new_columns" (default) | "sync_all_columns" | "fail"
                                         #   What happens when source schema changes between runs
    incremental_predicates: [string]     # optional - additional WHERE clauses for incremental filter
                                         #   e.g. ["reporting_date >= dateadd(month, -3, current_date)"]
    full_refresh_on: string              # optional - condition that triggers a full refresh
                                         #   e.g. "is_month_end" - custom logic

  retry:                                  # optional - guidance for production retry configuration
    max_retries: integer                 # default: 3 - documented in generated model comments
    backoff: string                      # "exponential" - for documentation
    notes: string                        # human-readable retry guidance
```

**Notes:**
- dbt handles retries natively via `dbt retry` (reruns failed nodes from the last run's `run_results.json`). The retry config here is guidance, not generated code.
- `incremental_predicates` are powerful for large tables - they limit the scan range on incremental runs. Without them, dbt scans the full target table for the merge.
- When `strategy: "full_refresh"`, all models use `materialized: "table"` (or `view`) and rebuild completely. No incremental logic is generated.

---

## Config Validation Rules

The generator validates the config before generating any artefacts. These rules are checked:

1. **Required fields:** All required fields must be present.
2. **Product type:** Must be `"foundation-base"` for this pack.
3. **Pattern ordering:** Steps must follow the framework composition order (batch_transfer - validation - schema_transform - calculated_fields - enrichment - filtering - curation - aggregation - validation - contracts - lineage - metadata - schema_publish - data_publish).
4. **No forbidden patterns:** Origin-only patterns (batch_ingestion) must not appear.
5. **Source references:** Every `source_ref` in schema_transform must match a source declared in batch_transfer.
6. **Key consistency:** `entity_key` in data_curation must appear in the schema_transform or calculated_fields output.
7. **Contract alignment:** Every column in data_contracts must be producible from the declared transforms, enrichments, and calculations.
8. **No duplicate field names:** Across all transforms and calculations, no two fields produce the same output name.
9. **Quarantine consistency:** If quarantine is enabled, the quarantine model name must not conflict with other model names.
10. **Incremental consistency:** If checkpointing strategy is incremental, the unique_key must be present in the contract columns.

---

## Minimal Config Example

The simplest valid config - a single-source foundation product with schema mapping only:

```yaml
pipeline:
  name: "foundation.product_dim"
  version: "1.0.0"
  product_type: "foundation-base"
  owner: "product-team"
  domain: "catalog"
  description: "Canonical product dimension from the catalog system"

steps:
  - pattern: "batch_transfer"
    config:
      sources:
        - name: "catalog_products"
          schema: "raw"
          table: "products"
          description: "Product catalog from the inventory system"

  - pattern: "schema_transform"
    config:
      source_ref: "catalog_products"
      mappings:
        - source_field: "prod_id"
          target_field: "product_id"
          data_type: "varchar"
          description: "Unique product identifier"
        - source_field: "prod_nm"
          target_field: "product_name"
          data_type: "varchar"
          description: "Human-readable product name"
        - source_field: "cat_cd"
          target_field: "category_code"
          data_type: "varchar"
          description: "Product category classification code"
        - source_field: "price_amt"
          target_field: "unit_price"
          data_type: "decimal(10,2)"
          description: "Current unit price"
        - source_field: "active_flg"
          target_field: "is_active"
          data_type: "boolean"
          cast_expression: "CASE WHEN active_flg = 'Y' THEN true ELSE false END"
          description: "Whether the product is currently active"

  - pattern: "data_validation"
    stage: "post"
    config:
      rules:
        - field: "product_id"
          type: "uniqueness"
          severity: "error"
          description: "Product ID must be unique"
        - field: "product_id"
          type: "completeness"
          severity: "error"
          description: "Product ID must not be null"
        - field: "product_name"
          type: "completeness"
          severity: "warn"
          threshold: 0.99
          description: "Product name should be at least 99% complete"

  - pattern: "data_contracts"
    config:
      contract_enforced: true
      description: "Canonical product dimension - stable interface for downstream consumers"
      columns:
        - name: "product_id"
          data_type: "varchar"
          description: "Unique product identifier"
          constraints:
            - type: "primary_key"
        - name: "product_name"
          data_type: "varchar"
          description: "Human-readable product name"
        - name: "category_code"
          data_type: "varchar"
          description: "Product category classification code"
        - name: "unit_price"
          data_type: "decimal(10,2)"
          description: "Current unit price"
        - name: "is_active"
          data_type: "boolean"
          description: "Whether the product is currently active"

  - pattern: "lineage_capture"
    config:
      generate_mapping_doc: true
  - pattern: "metadata_capture"
    config:
      meta_tags:
        domain: "catalog"
        owner: "product-team"
        tier: "tier2"
  - pattern: "schema_publish"
    config:
      access: "public"
  - pattern: "data_publish"
    config:
      build_command_guidance: true

checkpointing:
  enabled: true
  strategy: "full_refresh"
```

This produces:
- `sources.yml` with `catalog_products` source definition
- `int_product_dim_mapped.sql` with field mappings and type casts
- `fnd_product_dim.sql` as the final foundation model with contract enforcement
- Schema.yml files with tests (uniqueness, completeness), descriptions, and meta tags
- Config-model-mapping document
