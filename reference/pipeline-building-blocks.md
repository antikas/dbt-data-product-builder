# Pipeline Building Blocks

**Created:** 2026-03-07 (rebuilt from Dec 2024 discussions, "Documenting Data Pipeline Design Patterns")
**Status:** Active
**Nature:** Pure logical architecture, platform and technology agnostic
**Related:** [data-product-framework.md](data-product-framework.md)

---

## Overview

A data product pipeline is a composition of discrete, reusable building blocks, each solving one specific problem. This catalog documents 15 such patterns. Any given pipeline is assembled from a subset of these, combined in a defined order.

The value of the catalog is the **composition model**: knowing which patterns go together, in what order, and for what product type. Individual patterns are straightforward in isolation. The composition model is what makes a config-driven framework possible. The framework encodes the compositions; teams supply the configuration.

---

## The 15 Building-Block Patterns

### 1. Batch Ingestion
**Problem:** Bringing data from a source system into the data platform in a controlled, repeatable way for bounded (non-streaming) data.

**Solution:** Extract a defined scope of records from the source on a schedule, land them in the origin layer with metadata attached, and track what was extracted vs. what arrived.

**Key decisions:**
- Full extract vs. incremental (delta by timestamp, sequence number, or CDC)
- Pull (platform initiates) vs. push (source delivers)
- Partitioning strategy at landing (by date, by source, by batch ID)
- Duplicate detection: idempotent extract or dedup on arrival?

**When to use:** Any structured source, scheduled refresh cadence, tolerable latency measured in minutes to hours.

**Anti-pattern:** Extracting from operational source systems without a read replica or export mechanism. You will degrade source performance.

---

### 2. Batch Transfer
**Problem:** Moving data between layers of the platform, or between platform zones, without re-ingesting from the original source.

**Solution:** Transfer a partition or snapshot from one internal location to another, preserving provenance metadata. This is internal data movement, not external ingestion.

**Key decisions:**
- Copy (preserve origin, create new partition) vs. move (single authoritative copy)
- Whether transformation is allowed during transfer (prefer no, keep transfer pure)
- Metadata continuity: the transferred copy must carry forward source lineage

**When to use:** Promoting data from origin to a staging zone; replicating a curated product to a serving layer with different latency or access characteristics.

**Distinction from Batch Ingestion:** Ingestion is source → platform. Transfer is platform zone → platform zone.

---

### 3. Schema Transform
**Problem:** Source system schemas rarely match the enterprise canonical model. A principled, auditable mapping from source schema to target schema is needed.

**Solution:** Apply a declarative field mapping that converts source field names, types, and structures to the target schema. The mapping is configuration, not bespoke code.

**What it covers:**
- Field renaming (`cust_nm` → `customer_name`)
- Type casting (`varchar(20)` → `date`)
- Struct flattening or nesting
- Null handling (propagate null vs. substitute default)
- Dropping source fields not needed in the target layer

**What it does not cover:** Business logic. If you are computing derived values, that is Calculated Fields. If you are joining to a reference table to resolve a code, that is Data Enrichment. Schema Transform is pure structural mapping with no logic.

**Implementation:** A schema mapping config file (YAML or equivalent) consumed by the framework. Not hand-coded transformation logic.

---

### 4. Calculated Fields
**Problem:** Many downstream fields are not delivered by the source but must be derived, computed from one or more source fields using business rules.

**Solution:** Apply business rule expressions to input records to produce new derived fields. Rules are externalised as configuration or a rule file, not embedded in pipeline code.

**Examples:**
- `full_name = first_name + ' ' + last_name`
- `is_active = (status_code IN ('A', 'P') AND end_date IS NULL)`
- `transaction_quarter = DATE_TRUNC('quarter', transaction_date)`
- `risk_band = CASE WHEN score > 800 THEN 'low' WHEN score > 600 THEN 'medium' ELSE 'high' END`

**Key discipline:** Calculated fields should be idempotent. Given the same input record, they always produce the same output. No lookups, no joins, no external calls. For anything requiring a reference table, use Data Enrichment.

**Versioning:** Business rules change. The rule version must be captured in pipeline metadata so historical records can be recomputed when rules change.

---

### 5. Data Enrichment / Lookups
**Problem:** Source records carry codes or keys that need to be resolved to human-readable values, or enriched with attributes from reference data.

**Solution:** Join incoming records against a reference dataset to add or decode attributes. The reference dataset is itself a governed data product (typically a curated layer reference product).

**Examples:**
- Resolve a country code to country name and region
- Expand a product code to product name, category, and pricing tier
- Add customer segment from a segment assignment table
- Resolve an instrument ISIN to currency, asset class, and domicile

**Key decisions:**
- Broadcast join vs. sorted merge join (performance implications at scale)
- What to do when a lookup key has no match: fail, warn, substitute, or propagate null?
- Reference data freshness: if the reference data is updated daily, enrichment results reflect the reference data at time of pipeline run, not at time of original event

**The temporal trap:** If you are building a historical feature for ML and the reference data is slowly changing, you must join to the reference data as it existed at the time of the event, not as it exists now. Look up the Twine algorithm (from Anchor Modeling) for an efficient approach to temporal joins.

---

### 6. Data Filtering
**Problem:** Not all records from a source are relevant to a given data product. Irrelevant, invalid, or out-of-scope records need to be excluded in a documented, auditable way.

**Solution:** Apply explicit filter predicates as configuration. Records that do not pass filters are routed to a filter log so the decision is auditable, never silently dropped.

**Examples:**
- Include only records from specific business units
- Exclude test accounts, internal accounts, or dummy records
- Include only transactions in a defined currency set
- Exclude records before a platform cutover date

**The audit requirement:** Filtering must be transparent. "We have 10 million source records but only 8 million in the product" must be answerable by consulting the filter log: what predicates were applied, how many records matched each, and why.

**Anti-pattern:** Filtering embedded silently in SQL WHERE clauses without documentation. When the filter logic is wrong (and it will be), there is no way to audit what was excluded.

---

### 7. Data Validation
**Problem:** Detecting and handling data that violates expected quality constraints before it contaminates downstream products.

**Solution:** Apply a set of validation rules against incoming data. Records (or batches) that fail rules are quarantined, not silently passed through. Failures generate alerts.

**Rule types:**

| Type | Example |
|------|---------|
| **Completeness** | `customer_id` must be non-null |
| **Format/type** | `email_address` matches RFC-5321 pattern |
| **Range** | `transaction_amount` between -1M and +1M |
| **Referential** | `product_code` exists in product reference table |
| **Uniqueness** | No duplicate `transaction_id` within a batch |
| **Distribution** | Record count within ±20% of 30-day rolling average |
| **Consistency** | `end_date >= start_date` always |

**Quarantine model:** Failed records are routed to a quarantine zone with the failing rule, the record content, and a timestamp. They are held for investigation and reprocessing. The main pipeline continues with passing records (unless failure rate exceeds the threshold, in which case the pipeline aborts).

**Threshold vs. abort:** Define a maximum acceptable failure rate per product. Below threshold: quarantine and continue. Above threshold: abort the run, do not publish partial data, alert.

---

### 8. Data Aggregation
**Problem:** Curated and consumption products often need pre-computed summaries rather than row-level detail. Aggregates must be correct, reproducible, and maintainable.

**Solution:** Apply grouping and aggregation expressions as a configuration-driven step, operating on validated data. Aggregation logic is versioned.

**Examples:**
- Daily transaction count and volume by account
- Monthly portfolio NAV and return by fund
- Rolling 30-day average transaction value by customer
- Exposure by counterparty and asset class

**Key decisions:**
- Pre-aggregation (materialised in the product) vs. query-time aggregation (pushed to consumer): pre-aggregation improves query performance but adds storage and update complexity
- Temporal grain: what period does each aggregate row represent?
- Late arrival handling: what happens when a transaction arrives after the daily aggregate has been computed?

**Late arrival pattern:** Maintain a watermark. When a late record arrives, recompute the affected aggregate periods and republish. The framework must support deterministic recomputation.

---

### 9. Data Curation
**Problem:** Curated data products are assembled from multiple sources, sometimes with overlapping or conflicting representations of the same entity. Producing a single, authoritative, curated record requires explicit rules.

**Solution:** Apply business curation rules to resolve conflicts, apply entity resolution, and produce a golden record per entity. Curation decisions are explicit and auditable.

**What curation covers:**
- Entity resolution: matching records from different sources that represent the same real-world entity (deterministic by key, probabilistic by attributes)
- Golden record selection: when sources conflict on an attribute value, which source wins?
- Merge rules: for multi-valued attributes, which values to carry forward?
- Survivorship rules: when should a record be suppressed (e.g. flagged as duplicate)?

**Source precedence:** Each attribute should have a defined source precedence. "If CRM has a phone number and CBS has a phone number, prefer CRM unless null, then fall back to CBS." This is configuration, not code.

**The distinction from Validation:** Validation detects problems. Curation resolves them (or explicitly marks them as unresolvable).

---

### 10. Data Contracts (Pattern)
**Problem:** Consumers of a data product need confidence in what they will receive. The interface between a data product and its consumers must be formalised.

**Solution:** Define a versioned contract document that specifies schema, quality guarantees, freshness SLA, lineage, and access rules. The contract is the authoritative specification. The pipeline enforces it, not the other way around.

**See:** Full contract model in [data-product-framework.md](data-product-framework.md#data-contracts).

**In the pipeline context:** A contract validation step at the end of the pipeline confirms that the output product conforms to its published contract before the product is marked as "available." If the output violates the contract (e.g. a required field is null for >0.1% of records), the product is not published.

---

### 11. Lineage Capture
**Problem:** When something goes wrong with a data product, the problem must be traceable back to its source. When a source changes, affected downstream products must be identifiable.

**Solution:** Record the lineage graph at execution time, not just at design time. Track which products and sources each pipeline step reads from and writes to.

**Levels of lineage:**

| Level | Granularity | Example |
|-------|-------------|---------|
| **Product-level** | Which products depend on which | `curated.customer` ← `origin.crm_customer`, `origin.cbs_customer` |
| **Field-level** | Which output field derives from which input field | `full_name` ← `first_name` + `last_name` from `origin.crm_customer` |
| **Run-level** | Which specific batch produced which output | Run ID, timestamp, input record counts, output record counts |

**Minimum viable lineage:** Product-level is essential. Field-level is valuable for impact analysis and regulatory reporting. Run-level is essential for debugging.

**The framework responsibility:** The framework captures lineage automatically from pipeline configuration. Teams do not write lineage capture code. Every pipeline run emits a lineage record by default.

---

### 12. Metadata Capture
**Problem:** Data products are only useful if they are discoverable. Every product must be described, findable, and understandable without requiring teams to manually maintain documentation.

**Solution:** At pipeline execution, automatically publish metadata to a data catalog from the pipeline configuration and contract. The config is the documentation.

**What is captured:**
- Product name, version, type, owner, domain
- Schema (field names, types, descriptions, PII flags)
- Quality metrics from the run (completeness rates, record counts, SLA status)
- Lineage (upstream / downstream)
- Run timestamp and freshness status

**The automation principle:** Metadata should not require a separate documentation step. If teams have to write docs separately from building pipelines, documentation will drift from reality. The pipeline config file and contract YAML are the source of truth; the catalog is populated from them automatically.

---

### 13. Schema Publish
**Problem:** Consumers need to know the schema of a data product before they can use it, and they need to be notified when the schema changes.

**Solution:** At pipeline execution, publish the current schema version to a schema registry. Schema evolution is governed: additive changes (new nullable fields) are non-breaking; deletions and type changes are breaking and require a version increment and deprecation notice.

**Schema versioning rules:**

| Change type | Version impact | Consumer impact |
|-------------|---------------|----------------|
| Add nullable field | Minor increment | Non-breaking |
| Add required field | Major increment | Breaking |
| Remove field | Major increment | Breaking |
| Change field type | Major increment | Breaking |
| Rename field | Major increment | Breaking |

**The registry role:** Consumers subscribe to a schema version. They receive a notification (not a silent failure) when a breaking change occurs, giving them a deprecation window to update.

---

### 14. Data Publish
**Problem:** Once a data product pipeline has run, validated, and curated its output, the product must become available to consumers through a controlled publication step.

**Solution:** A publication step that atomically makes the new version of the product available: updates the "current" pointer (so consumers see the new version), registers the SLA as met, and notifies subscribed consumers.

**Atomicity requirement:** Publication must be all-or-nothing. A partially-written product visible to consumers, even briefly, can cause silent data quality issues downstream. Use partition swaps, table snapshots, or equivalent atomic replacement mechanisms.

**SLA recording:** When a product is successfully published, record: publish timestamp, record count, quality metrics summary. This feeds the SLA dashboard.

**Downstream notification:** Consumers who have registered a dependency on this product receive a notification that a new version is available. They do not need to poll.

---

### 15. Checkpoint & Retries
**Problem:** Pipelines fail: network timeouts, source system unavailability, transient errors. Pipelines need to recover from failure without re-running from scratch or producing duplicate data.

**Solution:** Checkpoint the pipeline at defined intervals. Record what has been successfully completed and what has not. On retry, resume from the last successful checkpoint, not from the beginning.

**Checkpoint granularity:**
- At the step level: completed steps are not re-executed; failed step is re-tried
- At the record level: for large datasets, record which partitions have been successfully processed

**Idempotency requirement:** Every step must be idempotent. Running it twice must produce the same result as running it once. If a step is not idempotent (e.g. it appends to a table), make it idempotent through upsert, truncate-and-reload, or partition-replace.

**Retry policy:** Define maximum retries, backoff strategy (linear, exponential), and alerting threshold. Do not retry indefinitely. Unresolvable failures must surface for human intervention.

---

## Pattern Composition by Product Type

Each product type in the layered framework uses a defined composition of these patterns. This is what the config-driven framework encodes.

### Origin Data Products

**Mandatory patterns:**
1. Batch Ingestion (or Streaming equivalent)
2. Data Validation (schema + completeness checks at landing)
3. Checkpoint & Retries
4. Lineage Capture
5. Metadata Capture
6. Data Publish

**Optional patterns:**
- Batch Transfer (if multi-zone architecture)
- Data Filtering (only for known exclusion rules, e.g. exclude test records)

**Not applicable at this layer:** Schema Transform, Calculated Fields, Data Enrichment, Data Aggregation, Data Curation. Origin products do not transform.

---

### Curated Data Products (Base)

**Core pattern sequence:**
1. Batch Transfer (from origin)
2. Data Validation (pre-transformation checks)
3. Schema Transform (source → canonical model mapping)
4. Calculated Fields (derived attributes from business rules)
5. Data Enrichment (resolve codes, join reference data)
6. Data Curation (entity resolution, golden record)
7. Data Validation (post-transformation checks)
8. Data Contracts (contract compliance check)
9. Lineage Capture
10. Metadata Capture
11. Schema Publish
12. Data Publish
13. Checkpoint & Retries (across all steps)

**Optional:**
- Data Filtering (scope restriction, e.g. a curated product covering one geography)
- Data Aggregation (rare at base CDP level, only for pre-computed summary attributes)

---

### Feature Data Products (Consolidated)

**Core pattern sequence:**
1. Batch Transfer (from base CDPs)
2. Data Validation (input quality checks)
3. Calculated Fields (feature computation: derived signals, scores, flags)
4. Data Enrichment (cross-entity attribute joins)
5. Data Aggregation (rolling windows, time-period summaries)
6. Data Validation (output quality checks)
7. Data Contracts
8. Lineage Capture
9. Metadata Capture
10. Schema Publish
11. Data Publish
12. Checkpoint & Retries

**Key addition vs. base CDP:** Aggregation is central here. Feature data products produce signals, not records.

**Temporal correctness requirement:** Point-in-time joins for historical training data must use the Twine algorithm or equivalent. Do not use BETWEEN-style joins at scale.

---

### Consumption Data Products

**Core pattern sequence:**
1. Batch Transfer (from curated data products)
2. Data Validation (input checks)
3. Calculated Fields (consumer-specific derived fields)
4. Data Filtering (consumer-specific scope)
5. Data Aggregation (consumer-specific grain)
6. Data Curation (consumer-specific merge/presentation rules)
7. Data Contracts
8. Lineage Capture
9. Metadata Capture
10. Data Publish
11. Checkpoint & Retries

**Note:** Consumption products do not publish to a schema registry in the same way. They serve a specific consumer and evolve with that consumer's needs, not on a broadly-subscribed schema.

---

## Pattern Dependency Map

```
Batch Ingestion / Batch Transfer
         │
         ▼
  Data Filtering ──────────────► Filter Log
         │
         ▼
  Data Validation ─────────────► Quarantine Zone
  (pre-transform)                 Failure Alerts
         │
         ▼
  Schema Transform
         │
         ├──► Calculated Fields
         │
         ├──► Data Enrichment ──► Reference Products
         │
         ├──► Data Curation
         │
         └──► Data Aggregation
              │
              ▼
       Data Validation
       (post-transform)
              │
              ▼
       Data Contracts ──────────► Contract Violation Alert
       (compliance check)
              │
              ├──► Lineage Capture ──► Lineage Graph
              ├──► Metadata Capture ──► Data Catalog
              ├──► Schema Publish ──► Schema Registry
              └──► Data Publish ──► Consumer Notification
```

Checkpoint & Retries wraps the entire sequence. Any step can fail and resume.

---

## Configuration Model

The framework maps pattern names to implementations. A pipeline configuration references patterns by name; the framework resolves them to the correct implementation class and executes them in order.

```yaml
pipeline:
  name: "curated.customer"
  type: "curated-base"

steps:
  - pattern: "batch_transfer"
    config:
      sources: ["origin.crm_customer", "origin.cbs_customer"]

  - pattern: "data_validation"
    stage: "pre"
    config:
      rules_file: "configs/customer_pre_validation.yaml"
      error_threshold: 0.001

  - pattern: "schema_transform"
    config:
      mapping_file: "configs/customer_schema_map.yaml"

  - pattern: "calculated_fields"
    config:
      rules_file: "configs/customer_business_rules.yaml"

  - pattern: "data_enrichment"
    config:
      lookups:
        - source: "ref.customer_segments"
          join_key: "customer_id"
          fields: ["segment_code", "segment_name"]

  - pattern: "data_curation"
    config:
      entity_key: "customer_id"
      resolution_strategy: "deterministic"
      survivorship_rules: "configs/customer_survivorship.yaml"

  - pattern: "data_validation"
    stage: "post"
    config:
      rules_file: "configs/customer_post_validation.yaml"

  - pattern: "data_contracts"
    config:
      contract_file: "contracts/curated.customer.yaml"

  - pattern: "lineage_capture"
  - pattern: "metadata_capture"
  - pattern: "schema_publish"
  - pattern: "data_publish"

checkpointing:
  enabled: true
  granularity: "step"
  max_retries: 3
  backoff: "exponential"
```

Each `pattern` entry resolves to a concrete implementation (DataTransformation subclass) registered in the framework's pattern registry. Adding a new pattern means: define the abstract interface, implement it, register it by name. Teams that want to extend the framework contribute new pattern implementations through the inner source process.

---

## TODO

- [ ] Streaming equivalents: map each batch pattern to its streaming counterpart (Change Data Capture, event stream validation, streaming aggregation with watermarks)
- [ ] Pattern testing model: how to unit test pattern implementations without a full pipeline execution environment
- [ ] Reference data management: lifecycle, versioning, and temporal handling for enrichment reference products
- [ ] Late arrival handling: formalise the watermark and recompute model for aggregation
- [ ] Pattern registry: how the framework discovers and loads pattern implementations at runtime
