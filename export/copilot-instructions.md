# GitHub Copilot Export

**Purpose:** Package the dbt FDP Build generator as GitHub Copilot custom instructions.

---

## Setup Instructions

1. In your dbt project root, create `.github/copilot-instructions.md`
2. Copy the content below into that file
3. Copilot will reference these instructions when generating dbt models

---

## File: `.github/copilot-instructions.md`

```markdown
# dbt Foundation Data Product Conventions

This project uses declarative pipeline configuration to generate dbt Foundation Data Products. Follow these conventions when generating or modifying dbt models.

## Model Naming

- Staging: `stg_{source_name}.sql`
- Intermediate: `int_{entity}_{verb}.sql` (e.g., `int_fund_master_mapped.sql`)
- Foundation: `fnd_{entity}.sql`
- Quarantine: `int_{entity}_quarantine.sql`

## SQL Structure

Every model follows the CTE pattern:

1. `with source as (select * from {{ source() }} or {{ ref() }})` - always first
2. Named transformation CTEs: `renamed`, `calculated`, `enriched`, `filtered`, `curated`
3. `final as (select {explicit columns} from {last_cte})`
4. `select * from final` - always last

Never use subqueries. Never use `SELECT *` in the final model's column list.

## Schema Transform Pattern

Map source fields to canonical model:
- Explicit `CAST()` for every field
- Comment dropped fields
- Deduplication via `ROW_NUMBER()` when needed

## Calculated Fields Pattern

- Add as CTE after `renamed`
- SQL expressions only (no joins, no lookups)
- Must be idempotent (same input = same output)
- Complex logic: reference a dbt macro via `escape_to_code`

## Data Enrichment Pattern

- LEFT JOIN to reference models via `{{ ref() }}`
- Always handle lookup misses: COALESCE with defaults or warn flag
- For temporal joins: include date range in ON clause

## Data Curation Pattern

- FULL OUTER JOIN sources on entity key
- Survivorship via COALESCE in source priority order
- Per-field priority overrides when different sources own different attributes

## Testing

- Every primary key: `unique` + `not_null`
- Every foreign key: `relationships`
- Every enum: `accepted_values`
- Every model: at least one test

## Contracts

Foundation models use `contract: { enforced: true }`:
- Every column has explicit `data_type`
- Column order matches the contract
- `meta` tags for PII, sensitivity, ownership

## Materialisation Defaults

- Intermediate: `view` (lightweight, recomputed)
- Foundation: `table` (consumed by many downstream)
- Aggregation: `table` (expensive to recompute)
- Quarantine: `table` (persists for investigation)

## Documentation

- Every model: `description` in schema YAML
- Every column: `description` in schema YAML
- `meta` tags: domain, owner, tier, pii flags
```

---

## Limitations

GitHub Copilot cannot:
- Run the full generate-review-iterate loop (use Claude Code for that)
- Validate YAML configs against the schema
- Run multiple review passes

The Copilot instructions provide conventions only. Copilot will reference them when you write or edit dbt models, but it will not orchestrate the full workflow.

For the full generation + review pipeline, use the Claude Code skill or Claude.ai project exports.
