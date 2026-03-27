# dbt Data Product Builder

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A complete prompt-based system for generating production-ready dbt models from declarative YAML pipeline configurations. Targets the **Foundation Base (silver/FDP build) layer**: the transformation from staging to curated, entity-centric data products.

Built on the principle that every field mapping, quality rule, enrichment join, and contract column is something you must know to build the pipeline correctly, regardless of technology. The declarative config captures that knowledge explicitly rather than burying it in SQL.

---

## Philosophy

### Business Logic Outlives Technology

The sources, business rules, quality thresholds, and data contracts that define a data product change far less frequently than the technology used to execute them. Business rules evolve quarterly. Platforms change on multi-year cycles - and when they change, the change is total.

This asymmetry is the reason declarative pipeline configuration exists. If business logic is captured in config (engine-agnostic, vendor-neutral), then a technology migration is a framework concern. The business logic, quality rules, and contracts do not change. Migration becomes a one-time effort on the translator, not a per-pipeline recode.

### Metaprogramming, Not Hand-Crafted SQL

Traditional dbt projects require teams to hand-write SQL models, tests, and documentation for every data product. This works, but it has scaling problems:

- **Inconsistency.** Different engineers write different patterns for the same transformation type. One team uses CTEs, another uses subqueries. One team tests primary keys, another doesn't.
- **Boilerplate.** 80% of a Foundation model is structural: field mapping, type casting, joins to reference tables, standard tests. Only 20% is unique business logic.
- **Drift.** Documentation diverges from code. Tests don't cover what the contract promises. Schema YAML describes fields that no longer exist.

The prompt pack solves this by treating dbt models as **generated artefacts from a specification**:

```
YAML Config (specification)  →  Generator Prompt  →  dbt Models (artefacts)
       ↑                                                    ↓
       │                                              Reviewer Prompts
       │                                                    ↓
       └──────────────── Fix & Iterate ◄────────── Findings
```

The YAML config is the source of truth. The generated dbt code is disposable - if deleted, it can be regenerated from the config. If the config changes, code is regenerated, not patched.

### Composable Building-Block Patterns

Every Foundation Data Product is assembled from the same set of transformation patterns, composed in a defined order:

```
Source Tables
     │
     ▼
Schema Transform ──► Rename, cast, map to canonical model
     │
     ▼
Calculated Fields ──► Derive new fields from business rules
     │
     ▼
Data Enrichment ──► Join reference tables to resolve/add attributes
     │
     ▼
Data Filtering ──► Exclude or flag records with audit trail
     │
     ▼
Data Curation ──► Entity resolution, golden record, survivorship
     │
     ▼
Data Validation ──► Quality checks, quarantine bad records
     │
     ▼
Data Contracts ──► Enforce published schema for consumers
     │
     ▼
Foundation Data Product (published, consumed by downstream)
```

The value is in the **composition model**: knowing which patterns go together, in what order, and what configuration each needs.

---

## The Patterns

| # | Pattern | What It Does | dbt Translation |
|---|---------|-------------|----------------|
| 2 | Batch Transfer | Declares source tables | `sources.yml` entries |
| 3 | Schema Transform | Maps source fields to canonical model | `int_{entity}_mapped.sql` with CAST/rename CTEs |
| 4 | Calculated Fields | Derives new fields from SQL expressions | Inline CTEs or separate model |
| 5 | Data Enrichment | Joins reference tables for lookups | LEFT JOIN CTEs with fallback handling |
| 6 | Data Filtering | Excludes/flags records with audit trail | WHERE clauses with comments |
| 7 | Data Validation | Quality checks (pre and post) | dbt tests in schema.yml + singular tests + optional quarantine model |
| 8 | Data Aggregation | Pre-computed summaries | GROUP BY model with window functions |
| 9 | Data Curation | Entity resolution across sources | Deterministic matching + survivorship rules |
| 10 | Data Contracts | Published interface for consumers | dbt model contracts + column constraints |
| 11 | Lineage Capture | Dependency tracking | dbt native + config-model mapping doc |
| 12 | Metadata Capture | Discovery and documentation | schema.yml descriptions + meta tags |
| 13 | Schema Publish | Visibility and versioning | Access modifiers + version manifest |
| 14 | Data Publish | Making the product available | Build guidance + grants |
| 15 | Checkpoint & Retries | Incremental processing | dbt incremental models + retry guidance |

---

## Prerequisites

1. **A dbt project** with a working connection to your data warehouse (Snowflake, BigQuery, Databricks, PostgreSQL, DuckDB, etc.)
2. **Staging models** already built - the prompt pack generates Foundation (intermediate) and above, not staging
3. **An AI assistant** - Claude (Code or .ai), GitHub Copilot, or any LLM that accepts markdown prompts
4. **Understanding of your data** - you need to know your source fields, business rules, and quality requirements to write the YAML config

---

## Quick Start

### Step 1: Write Your Config

Create a YAML file following the schema in [config-spec.md](config-spec.md). Start with the [minimal example](config-spec.md#minimal-config-example) and add patterns as needed.

### Step 2: Run the Generator

Pass the generator prompt ([generator.md](generator.md)) and your config to your AI assistant:

> "Using the dbt FDP Build generator prompt, generate dbt models from this config: [paste config]"

### Step 3: Review the Output

Run the orchestrator ([orchestrator.md](orchestrator.md)) to review the generated artefacts through four review passes:

> "Using the dbt FDP Build orchestrator, review these generated artefacts against the config: [paste config and output]"

### Step 4: Copy to Your Project

Copy the generated files into your dbt project directory.

### Step 5: Build and Test

```bash
dbt deps                              # Install required packages
dbt compile                            # Verify SQL compiles
dbt build --select fnd_{entity}+       # Build and test
dbt docs generate && dbt docs serve    # Verify documentation
```

---

## Config Reference

Full specification: [config-spec.md](config-spec.md)

The config uses the exact schema from the Data Product Framework, expanded with inline detail for dbt:

```yaml
pipeline:        # Product identity, dbt-specific settings
  name: "foundation.{entity}"
  product_type: "foundation-base"
  dbt:           # target_schema, model_prefix, materialisation, tags

steps:           # Ordered pattern sequence
  - pattern: "batch_transfer"      # Source declarations
  - pattern: "data_validation"     # Pre-validation rules
    stage: "pre"
  - pattern: "schema_transform"    # Field mappings, type casts
  - pattern: "calculated_fields"   # SQL expressions, escape-to-code
  - pattern: "data_enrichment"     # Reference table joins
  - pattern: "data_filtering"      # Inclusion predicates
  - pattern: "data_curation"       # Entity resolution
  - pattern: "data_validation"     # Post-validation rules
    stage: "post"
  - pattern: "data_contracts"      # Published schema
  - pattern: "lineage_capture"     # Mapping doc generation
  - pattern: "metadata_capture"    # Meta tags
  - pattern: "schema_publish"      # Access, versioning
  - pattern: "data_publish"        # Build guidance, grants

checkpointing:   # Incremental or full_refresh strategy
```

Not every pattern is required. The minimal config needs: `batch_transfer` + `schema_transform` + `data_contracts`.

---

## Generator

The generator ([generator.md](generator.md)) takes your YAML config and produces:

| Artefact | Description |
|----------|-------------|
| `_sources.yml` | Source definitions from batch_transfer |
| `int_{entity}_mapped.sql` | Schema transform model(s) |
| `int_{entity}_curated.sql` | Entity resolution model (if multi-source) |
| `fnd_{entity}.sql` | Final foundation model with contract |
| `_int_{entity}__models.yml` | Intermediate schema with tests |
| `_fnd_{entity}__models.yml` | Foundation schema with contract, meta, access |
| Singular test files | Custom SQL tests from validation rules |
| Quarantine model | Failed records (if quarantine enabled) |
| Config-model mapping | Lineage document tracing config to models |

The generator validates your config before generating. If the config is invalid (missing fields, wrong ordering, unresolvable references), it stops and reports errors.

---

## Review Pipeline

Four reviewers run in sequence, managed by the orchestrator ([orchestrator.md](orchestrator.md)):

| Order | Reviewer | Focus |
|-------|----------|-------|
| 1 | [dbt Best Practices](reviewers/dbt-best-practices.md) | Naming, CTE structure, ref/source, materialisation, test coverage |
| 2 | [Pattern Compliance](reviewers/pattern-compliance.md) | Every config step implemented, no silent omissions |
| 3 | [Contract Conformance](reviewers/contract-conformance.md) | Output matches declared contract exactly |
| 4 | [SSOT / Duplication](reviewers/ssot-duplication.md) | No business logic repeated across models |

### The Iteration Loop

```
Generate → Review 1 → Review 2 → Review 3 → Review 4 → Done
              ↑           ↑           ↑           ↑
              └─ fix ─────┘─ fix ─────┘─ fix ─────┘
```

- **ERROR findings:** Must be fixed before proceeding. The orchestrator fixes and re-reviews (max 3 iterations per reviewer).
- **WARNING findings:** Collected and surfaced to you at the end. Not blocking.
- **INFO findings:** Suggestions. Take or leave.
- **Safety cap:** 3 iterations per reviewer, 10 total. After that, remaining errors are surfaced for manual intervention.

---

## Worked Examples

### Investment Domain - Fund Master

A multi-source FDP with GP reports and administrator data, entity resolution, enrichment, quarantine, and incremental strategy.

- [Config](examples/investment/config.yml) - full YAML config with all patterns
- [Expected Output](examples/investment/expected-output.md) - all generated dbt artefacts
- [Config-Model Mapping](examples/investment/config-model-mapping.md) - lineage trace

### Customer/Orders Domain - Customer

A simpler FDP with CRM and billing sources, full refresh, no quarantine. Demonstrates PII tagging and protected access.

- [Config](examples/customer-orders/config.yml) - YAML config
- [Expected Output](examples/customer-orders/expected-output.md) - generated artefacts
- [Config-Model Mapping](examples/customer-orders/config-model-mapping.md) - lineage trace

---

## Working with Generated Artefacts

### Copying into Your Project

Generated files follow dbt conventions. Copy them into the matching directories:

```
your_dbt_project/
├── models/
│   ├── sources/     ← _sources.yml
│   ├── intermediate/ ← int_*.sql, _int_*__models.yml
│   └── foundation/  ← fnd_*.sql, _fnd_*__models.yml
├── tests/singular/  ← assert_*.sql
├── macros/          ← quarantine_*.sql (if applicable)
└── docs/            ← config_model_mapping_*.md
```

If your project uses different directory names (e.g., `marts/` instead of `foundation/`), adjust the paths accordingly. The generator adapts to your conventions when you provide your `dbt_project.yml`.

### Running the Models

```bash
# Install required packages first
dbt deps

# Full build (models + tests)
dbt build --select fnd_{entity}+

# Just the models (skip tests)
dbt run --select fnd_{entity}+

# Just the tests
dbt test --select fnd_{entity}

# Source freshness check (if configured)
dbt source freshness

# Generate and serve documentation
dbt docs generate
dbt docs serve
```

### Evolving the Output

The generated code is **disposable by design**. Update the config and regenerate when requirements change:

1. Update the YAML config (source of truth)
2. Re-run the generator to produce new artefacts
3. Re-run the reviewers to validate
4. Replace the old files with the new ones

Do not hand-edit generated models to add business logic. Instead:
- Add a `calculated_fields` step to the config
- Add an `escape_to_code` reference to a custom macro
- Add a `data_enrichment` step

If you must hand-edit (e.g., a warehouse-specific optimisation), document it in the model header. Be aware that regeneration will overwrite your edit.

### Extending Beyond Foundation Base

This prompt pack covers Foundation Base. For other layers:

- **Staging:** Not generated - you write staging models by hand or with dbt codegen
- **Foundation Feature (2b):** Uses the same patterns plus `data_aggregation`. Extend the config with aggregation steps.
- **Consumption (marts):** Purpose-built for specific consumers. Often hand-written since they're domain-specific.

---

## Export to Other Tools

See the [Export Guide](export/export-guide.md) for detailed instructions on packaging for:

- **[Claude Code](export/claude-code-skill.md)** - as a slash command skill with review sub-skills
- **[Claude.ai](export/claude-ai-project.md)** - as project knowledge for web-based sessions
- **[GitHub Copilot](export/copilot-instructions.md)** - as custom instructions in `.github/copilot-instructions.md`

The SSOT is always the markdown files in this directory. Exports are views - never edit the exports directly.

---

## Relationship to the Data Product Framework

This prompt pack implements the concepts from the [Data Product Framework](reference/data-product-framework.md) for dbt specifically. The framework is platform-agnostic (Python meta-layer with target translators); this pack is the dbt translation.

| Framework Concept | Prompt Pack Implementation |
|-------------------|---------------------------|
| 15 building-block patterns | Config steps with dbt-specific translation rules |
| YAML pipeline config | Exact same schema, expanded with inline detail |
| Pattern composition model | Enforced ordering and mandatory/optional rules |
| Target translator | The generator prompt IS the dbt translator |
| Verification strategy | The 4 reviewer prompts + orchestrator |
| Conventions | Embedded in the generator and reviewer prompts |

When the framework's Phase 3 (dbt translator as Python code) is eventually built, this prompt pack will have validated the approach and serves as the specification.

---

## File Index

| File | Purpose |
|------|---------|
| [README.md](README.md) | This guide |
| [config-spec.md](config-spec.md) | YAML config schema specification |
| [generator.md](generator.md) | Main generator prompt |
| [orchestrator.md](orchestrator.md) | Review pipeline orchestrator |
| [reviewers/dbt-best-practices.md](reviewers/dbt-best-practices.md) | Reviewer 1 |
| [reviewers/pattern-compliance.md](reviewers/pattern-compliance.md) | Reviewer 2 |
| [reviewers/contract-conformance.md](reviewers/contract-conformance.md) | Reviewer 3 |
| [reviewers/ssot-duplication.md](reviewers/ssot-duplication.md) | Reviewer 4 |
| [examples/investment/](examples/investment/) | Investment domain worked example |
| [examples/customer-orders/](examples/customer-orders/) | Customer domain worked example |
| [templates/](templates/) | Project layout and mapping templates |
| [export/](export/) | Export guides for Claude Code, Claude.ai, Copilot |
| [reference/data-product-framework.md](reference/data-product-framework.md) | Underlying data product architecture |
| [reference/pipeline-building-blocks.md](reference/pipeline-building-blocks.md) | The 15 building-block patterns |

---

## License

MIT. See [LICENSE](LICENSE).
