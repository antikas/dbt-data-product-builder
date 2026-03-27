# Export Guide

**Purpose:** How to package the dbt FDP Build prompt pack for different AI tools.

---

## Principle: Single Source of Truth

The markdown files in this repository are the SSOT. Every export format is a view of this content, not a copy.

When you update the prompt pack:
1. Edit the source files in this directory
2. Re-export to each target format
3. Never edit the exports directly

---

## Export Formats

### 1. Claude Code - Slash Command Skills

See [claude-code-skill.md](claude-code-skill.md) for the full template.

**What you get:** `/fdp-build` skill that accepts a YAML config and runs the full generate + review cycle.

**How to set up:**
1. Copy the skill file to your dbt project's `.claude/commands/` directory
2. The skill inlines the key parts of config-spec, generator, and orchestrator
3. Reviewer prompts are referenced as sub-agent calls within the orchestrator

### 2. Claude.ai - Project Knowledge

See [claude-ai-project.md](claude-ai-project.md) for instructions.

**What you get:** A Claude.ai project pre-loaded with the full prompt pack as knowledge.

**How to set up:**
1. Create a new Claude.ai project
2. Upload the content files in the specified order
3. Start a conversation with your YAML config

### 3. GitHub Copilot - Custom Instructions

See [copilot-instructions.md](copilot-instructions.md) for the template.

**What you get:** Copilot references the generator patterns when you write dbt models.

**Limitations:** Copilot cannot orchestrate multi-step workflows. The export focuses on the generator prompt only. Run reviewers manually via Claude or another tool.

---

## Which Format for Which Use Case

| Use case | Best format |
|----------|------------|
| Daily dbt development in VS Code | Claude Code skill |
| One-off model generation | Claude.ai project |
| Team-wide conventions | Copilot instructions - manual review |
| CI/CD integration | Claude Code skill (scriptable) |
