# Claude.ai Project Export

**Purpose:** Load the dbt FDP Build prompt pack into a Claude.ai project as knowledge.

---

## Setup Instructions

### Option A: Single Combined Document (Recommended)

1. Go to [claude.ai](https://claude.ai) → Projects → Create Project
2. Name it "dbt FDP Build"
3. In Project Knowledge, upload or paste these files **in this order**:
   1. `config-spec.md` - the YAML config specification (most important)
   2. `generator.md` - the generator prompt
   3. `orchestrator.md` - the review orchestrator
   4. `reviewers/dbt-best-practices.md` - Reviewer 1
   5. `reviewers/pattern-compliance.md` - Reviewer 2
   6. `reviewers/contract-conformance.md` - Reviewer 3
   7. `reviewers/ssot-duplication.md` - Reviewer 4
   8. `templates/dbt-project-layout.md` - default project structure
4. Optionally upload the examples for reference:
   - `examples/investment/config.yml`
   - `examples/investment/expected-output.md`

### Option B: Concatenated Single File

If you prefer a single upload, concatenate the files above into one markdown document with clear section headers:

```markdown
# dbt FDP Build - Complete Prompt Pack

## 1. Config Specification
{content of config-spec.md}

## 2. Generator
{content of generator.md}

## 3. Orchestrator
{content of orchestrator.md}

## 4. Reviewer: dbt Best Practices
{content of reviewers/dbt-best-practices.md}

...
```

### Project Instructions

Set these as the project's custom instructions:

```
You are a dbt model generator and reviewer. You help users create Foundation Data Products by:
1. Taking YAML pipeline configs as input
2. Generating complete dbt artefacts (models, tests, schema YAML, docs)
3. Reviewing generated output through 4 review passes
4. Iterating until all errors are resolved

Always follow the config-spec for YAML validation. Always follow the generator rules for code generation. Always run the 4 reviewers in order when reviewing.
```

---

## Usage

Start a conversation with:

> "Generate dbt models from this config:"
> ```yaml
> pipeline:
>   name: "foundation.{entity}"
>   ...
> ```

Or:

> "Review these generated dbt models against this config: [paste both]"

---

## Limitations

- Claude.ai sessions do not persist generated files - copy them before the session ends
- The iteration loop depends on conversation context - very large configs may need to be split across messages
- No file system access - you paste config in and copy output out
