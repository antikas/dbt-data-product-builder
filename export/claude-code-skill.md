# Claude Code Skill Export

**Purpose:** Package the dbt Data Product Builder generator + orchestrator as a Claude Code slash command.

---

## Setup Instructions

1. In your dbt project, create `.claude/commands/fdp-build.md`
2. Copy the skill content below into that file
3. Invoke with: `/fdp-build` in a Claude Code session
4. The skill will ask you to provide or point to your YAML config

---

## Skill File: `.claude/commands/fdp-build.md`

```markdown
# dbt Data Product Builder Generator

Generate production-ready dbt models from a declarative YAML pipeline configuration for Curated Base data products.

## What This Skill Does

1. Reads a YAML pipeline config (you provide it or point to a file)
2. Validates the config structure and composition rules
3. Generates complete dbt artefacts (models, schema YAML, tests, macros, mapping doc)
4. Runs 4 review passes in sequence:
   - dbt best practices (naming, structure, tests)
   - Pattern compliance (config fully implemented)
   - Contract conformance (output matches contract)
   - SSOT/duplication (no repeated logic)
5. Iterates fixes until all ERROR findings are resolved (max 3 per reviewer)
6. Outputs final artefacts with a consolidated review summary

## Instructions

Ask the user for their YAML pipeline config. It should follow this structure:

- `pipeline:` - name, version, product_type (must be "curated-base"), owner, domain, description
- `pipeline.dbt:` - target_schema, model_prefix, materialisation, tags
- `steps:` - ordered list of pattern configs (batch_transfer, schema_transform, calculated_fields, data_enrichment, data_filtering, data_validation, data_curation, data_aggregation, data_contracts, lineage_capture, metadata_capture, schema_publish, data_publish)
- `checkpointing:` - incremental or full_refresh strategy

Read the user's existing dbt project (dbt_project.yml, existing models) to match conventions.

### Config Validation

Before generating, validate:
1. All required fields present
2. product_type is "curated-base"
3. Steps follow valid pattern ordering
4. Source references resolve
5. Entity key is producible from transforms
6. Contract columns are all producible

### Generation

For each pattern step, generate the corresponding dbt artefact:
- batch_transfer → sources.yml
- schema_transform → int_{entity}_mapped.sql (CTE: source → renamed → final)
- calculated_fields → inline CTEs or separate model
- data_enrichment → LEFT JOIN CTEs with fallback handling
- data_filtering → WHERE clauses with audit comments
- data_curation → entity resolution with survivorship
- data_validation → dbt tests in schema.yml + singular tests + optional quarantine
- data_contracts → contract: {enforced: true} + column constraints
- Final model → cur_{entity}.sql with explicit column list

### Review Loop

After generation, run 4 review passes in order. For each:
- If ERROR findings: fix and re-review (max 3 iterations)
- If only WARNING/INFO: collect and continue
- If stuck after 3 iterations: surface to user

### Output

Present generated files with full paths. End with summary: files generated, required packages, warnings, next steps (dbt deps, dbt build, dbt test).
```

---

## Optional: Reviewer Sub-Skills

For more granular control, create separate reviewer skills:

- `.claude/commands/fdp-review-dbt.md` - dbt best practices reviewer
- `.claude/commands/fdp-review-compliance.md` - pattern compliance reviewer
- `.claude/commands/fdp-review-contract.md` - contract conformance reviewer
- `.claude/commands/fdp-review-ssot.md` - SSOT/duplication reviewer

Copy the content from the corresponding `reviewers/*.md` files in the prompt pack. The orchestrator skill can then call these as sub-agents.
