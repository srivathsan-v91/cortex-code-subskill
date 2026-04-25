
# Data Vault Pipeline Development Skill

> **Note:** Cortex Code skills do not require a README file. This document
> exists solely to provide orientation for humans browsing the GitHub
> repository. The agent reads `SKILL.md`, `SKILL_datavault.md`, and
> `SKILL_pipeline.md` directly — it never consults this file.

## What This Skill Does

A three-file Cortex Code skill that onboards a new data source into
Snowflake end-to-end:

1. **Profiles** the source dataset (CSV, Parquet, JSON)
2. **Generates** Data Vault 2.0 DDL (hubs, links, satellites)
3. **Generates** ingestion pipeline SQL (staging, COPY INTO, vault load)

All generated SQL is compile-only validated — nothing is executed without
explicit user approval at mandatory stopping points.

## File Structure

```
datavault-pipeline-development-skill/
├── SKILL.md              # Base skill — source profiling + delegation
├── SKILL_datavault.md    # Sub-skill — Data Vault model DDL generation
├── SKILL_pipeline.md     # Sub-skill — Pipeline SQL generation
└── docs/
    └── README.md         # This file (for GitHub only)
```

## Workflow

```
SKILL.md (Entry Point)
  Step 1: Gather inputs (source path, schema, preferences)  --> STOP
  Step 2: Profile source data, classify DV roles             --> STOP
  Step 2b: Write source profile dictionary                   --> STOP
    |
    ├──> SKILL_datavault.md (if DV model requested)
    |      Step 3: Generate hub/link/satellite DDL
    |      Step 4: Validate & present model                  --> STOP
    |      Step 4b: Write DDL to file                        --> STOP
    |
    └──> SKILL_pipeline.md (if pipeline requested)
           Step 5: Generate staging DDL, COPY INTO, vault load
           Step 6: Validate & present pipeline               --> STOP
           Step 6b: Write pipeline SQL to file               --> STOP
```

Seven mandatory stopping points ensure the user reviews and approves
every intermediate artifact before the skill proceeds.

## Key Concepts

- **Source Profile Dictionary** — 12-field structured contract between
  the base skill and sub-skills (column name, inferred type, DV role,
  hub entity, quality risk, etc.)
- **Data Vault 2.0** — Hubs (`HUB_`), Links (`LNK_`), Satellites
  (`SAT_`), Hierarchical Links (`HLNK_`) with hash keys via
  `MD5(UPPER(TRIM(COALESCE(...))))`
- **Safe Casting** — Always `TRY_TO_*` functions, never raw `CAST()`
- **Pipeline Layers** — Two-layer (staging to vault) or three-layer
  (staging to hash staging to vault)

## How to Use

1. Place this folder under your Cortex Code skills directory
   (e.g., `~/.snowflake/cortex/skills/`)
2. In Cortex Code, the skill activates on triggers like:
   *"onboard data"*, *"profile source"*, *"data vault"*,
   *"ingestion pipeline"*, *"COPY INTO"*, *"staging table"*
3. Follow the prompted workflow — the skill will ask for inputs and
   pause at each stopping point for your approval

## Requirements

- Snowflake account with a configured Cortex Code connection
- Source data file (CSV, Parquet, or JSON) or an existing Snowflake stage
- Target schema for generated objects
