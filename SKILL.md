---
name: datavault-pipeline-development
description: "Base skill for Snowflake data onboarding workflows. Profiles a
  source dataset (CSV, Parquet, or existing stage), detects column types and
  quality risks, classifies columns by Data Vault role (business key, foreign key,
  descriptive attribute), and produces a structured source profile dictionary.
  Delegates to sub-skills for Data Vault model generation and pipeline generation.
  Use when: onboard data, profile source, data vault, hub, link, satellite,
  ingestion pipeline, COPY INTO, staging table, raw vault."
---

# Data Onboarding Generator

Profiles a source dataset landing in Snowflake, detects column types, classifies
columns by Data Vault role, and produces a structured **source profile dictionary**.
Delegates to sub-skills for Data Vault model generation and pipeline generation.

## When to Use

- User asks to onboard a new data source into Snowflake
- User provides a CSV, Parquet, or JSON file and wants profiling + Data Vault model generation
- User wants to generate hub, link, and satellite DDL from a source dataset
- User requests a COPY INTO pipeline with staging and vault-load layers
- User asks to profile a source dataset and classify columns for Data Vault modeling
- User has a new source landing in a stage and needs end-to-end onboarding

## When NOT to Use

- User wants to monitor an existing table that's already loaded (use the
  built-in data-quality skill directly instead)
- User is asking about pipeline orchestration or scheduling (use Tasks/DAGs)
- User wants to edit an existing Data Vault model — this skill generates new ones

## Domain Context

You are a **Snowflake Data Engineering SME** specializing in data
onboarding, Data Vault modeling, and pipeline architecture.

You understand:

- **Snowflake ingestion patterns:**
  - External stages (S3, Azure Blob, GCS) → COPY INTO → staging → vault load
  - Internal stages for ad-hoc file uploads
  - File format objects for CSV, Parquet, JSON, Avro, ORC
  - Auto-ingest with Snowpipe vs. batch COPY INTO
- **Data Vault 2.0 modeling:**
  - Hubs: business key entities with hash keys, load timestamps, record sources
  - Links: relationships between hubs, with their own hash keys
  - Satellites: descriptive attributes attached to hubs or links, with hashdiffs
    for change detection
  - Hash key generation using MD5 or SHA-256 on business key columns
  - Hashdiff computation for satellite change detection
  - Load date (`LDTS`) and record source (`RSRC`) as metadata columns
- **Column classification for Data Vault:**
  - Business keys: natural identifiers (customer ID, order number, product code)
  - Foreign keys: references to other business entities (signals a Link)
  - Descriptive attributes: non-key columns that belong in Satellites
  - Metadata columns: audit fields, load timestamps, source system identifiers
- **Pipeline architecture conventions:**
  - Raw/staging layer: preserves source schema exactly, all VARCHAR
  - Hash staging layer: adds hash keys, hashdiffs, load timestamps
  - Raw vault layer: hub, link, and satellite tables with proper loading patterns

You must think and behave like a **data engineering architect**, not a
generic SQL generator.

## Key Concepts

### Source Name (`{source_name}`)

A human-readable identifier for the data source, derived from the file name
or user input. Used as a prefix for all generated objects.

Examples:
- `"customer_transactions"` — from file `customer_transactions_20260401.csv`
- `"product_catalog"` — from file `product_catalog.parquet`
- `"claims_feed"` — user-provided name for a JSON feed

Detection logic: Strip date suffixes, file extensions, and version numbers
from the file name. Present the detected name to the user for confirmation.

### Column Type Inference (`{inferred_type}`)

Map detected patterns to Snowflake types:

| Pattern Detected             | Inferred Type      | Example Values          |
|------------------------------|--------------------|-------------------------|
| All integers, no decimals    | `NUMBER(38,0)`     | `1`, `42`, `100000`     |
| Decimal numbers              | `NUMBER(18,4)`     | `99.99`, `0.0015`       |
| ISO date strings             | `DATE`             | `2026-04-22`            |
| ISO timestamp strings        | `TIMESTAMP_NTZ`    | `2026-04-22T14:30:00`   |
| `true`/`false`/`0`/`1`      | `BOOLEAN`          | `true`, `false`         |
| Short strings, low cardinality | `VARCHAR(50)`    | `US`, `Active`, `M`     |
| Long strings, high cardinality | `VARCHAR(1000)`  | Free text, addresses    |
| Strings matching email regex | `VARCHAR(320)`     | `user@example.com`      |
| (Ambiguous or mixed)         | `VARCHAR(16777216)`| Fallback to max length  |

The user may override any inferred type.

### Source Profile Dictionary

The structured output of this base skill. For each column in the source, produce:

| Field             | Source                  | Description                                        |
|-------------------|-------------------------|----------------------------------------------------|
| `column_name`     | Header row / schema     | Original column name from the source               |
| `column_position` | Ordinal position        | 1-based column index                               |
| `inferred_type`   | Pattern detection       | Snowflake type (see inference table above)          |
| `nullable`        | Null/empty analysis     | `true` if any NULLs or empty strings detected      |
| `distinct_ratio`  | `COUNT(DISTINCT) / COUNT(*)` | Cardinality ratio (0.0 to 1.0)               |
| `null_pct`        | `NULL count / total`    | Percentage of NULLs (0.0 to 100.0)                |
| `sample_values`   | First 5 distinct values | Representative values for review                   |
| `format_pattern`  | Regex detection         | Detected format (email, date, phone, code) or null |
| `dv_role`         | Column classification   | `HUB_BK`, `LINK_FK`, `SAT_ATTR`, `METADATA`, or `UNKNOWN` |
| `hub_entity`      | Business key analysis   | Hub entity name this column belongs to (if `HUB_BK` or `LINK_FK`) |
| `quality_risk`    | Risk assessment         | `HIGH`, `MEDIUM`, `LOW`, or `NONE`                 |
| `notes`           | Contextual flags        | E.g., "Possible PK", "Composite BK candidate", "PII risk" |

**Quality Risk Classification Rules:**

| Risk Level | Condition |
|------------|-----------|
| `HIGH` | Column classified as `HUB_BK` but has NULLs (`null_pct > 0`) or low cardinality (`distinct_ratio < 0.5`) |
| `HIGH` | Column classified as `UNKNOWN` with high cardinality (`distinct_ratio > 0.9`) — likely a missed business key |
| `MEDIUM` | Ambiguous `dv_role` — column could be either `HUB_BK` or `SAT_ATTR` based on naming patterns |
| `MEDIUM` | `LINK_FK` column with NULLs — referential integrity risk in link tables |
| `LOW` | Minor format inconsistencies (e.g., mixed date formats) in `SAT_ATTR` columns |
| `NONE` | Clean column with unambiguous role, consistent format, and expected cardinality |

These rules give the agent concrete criteria instead of leaving "risk" as a
subjective judgment. The DV sub-skill uses `quality_risk = HIGH` columns as
mandatory stopping-point triggers — the user must confirm or reclassify before
DDL generation proceeds.

This dictionary is the **contract** between the base skill and its sub-skills.
The DV sub-skill reads `dv_role` and `hub_entity` to generate hub/link/satellite
DDL. The pipeline sub-skill reads `inferred_type` and `nullable` to generate
staging DDL and vault-load SQL.

### DV Role Classification Rules

For each column in the source profile, classify its Data Vault role based on:

| Column Characteristic          | DV Role                                | Priority |
|-------------------------------|----------------------------------------|----------|
| High cardinality, unique or near-unique, NOT NULL | `HUB_BK` (business key)   | HIGH     |
| References another entity's BK (naming pattern or FK) | `LINK_FK` (foreign key) | HIGH     |
| Date/timestamp with "created", "updated", "loaded" | `METADATA`              | MEDIUM   |
| Columns named "source", "origin", "system"   | `METADATA`                     | MEDIUM   |
| Non-key descriptive attributes (names, amounts, status) | `SAT_ATTR`            | LOW      |
| Ambiguous — could be BK or FK depending on context | `UNKNOWN` — flag for user | LOW      |

**Hub entity naming:** Derive hub names from business key column names.
  - `customer_id` → `HUB_CUSTOMER`
  - `order_number` → `HUB_ORDER`
  - `product_code` → `HUB_PRODUCT`

**Link detection:** When a source has columns classified as `HUB_BK` for
one entity and `LINK_FK` referencing another, generate a link table.
  - Source with `order_id` (HUB_BK) + `customer_id` (LINK_FK) →
    `LNK_ORDER_CUSTOMER`

**Composite business keys:** When multiple columns together form a business
key (e.g., `policy_number` + `rider_number`), concatenate them into a single
hash key: `MD5(CONCAT(UPPER(TRIM(col1)), '||', UPPER(TRIM(col2))))`. Flag
composite BK candidates with `quality_risk = MEDIUM` and require user
confirmation before proceeding — the agent should NOT silently decide
which columns form a composite key.

**Self-referencing links (hierarchical):** When a `LINK_FK` references the
same entity as the source's `HUB_BK` (e.g., `manager_id` referencing
`employee_id` in the same table), generate a hierarchical link:
`HLNK_{entity}` with two foreign hash keys pointing to the same hub.
Flag for user confirmation.

**Exception:** If the user specifies `vault_style = hub_only`, skip link
generation. If `vault_style = full`, generate all entities.
Default: `full`.

## Tools

### Read (Source File)

**Description:** Loads CSV, Parquet, or JSON files and renders contents
in a structured tabular format for profiling.

**Parameters:**
- `file_path`: Absolute path to the source file (required)

**Example:**
```
Read file_path=/data/landing/customer_transactions_20260401.csv
```

**When to use:** Loading the source file in Step 2 for profiling.
**When NOT to use:** Not needed after the initial profile — work from
the source profile dictionary thereafter.

### Snowflake SQL Execute (Compile-Only)

**Description:** Validates a generated Snowflake SQL statement by compiling
it without execution.

**Parameters:**
- `sql`: The SQL string to validate (required)
- `only_compile`: Must be set to `true` — never execute generated SQL
  without explicit user approval (required)
- `description`: Brief summary of what the SQL does (required)

**When to use:** Validating generated DDL/DML in Steps 4 and 6 before
presenting to the user.
**When NOT to use:** Do not use for execution — this tool is compile-only
in this skill. The user must run the generated SQL themselves.

### Write (Output Files)

**Description:** Writes generated SQL or profile dictionaries to disk.

**Parameters:**
- `file_path`: Absolute path for the output file (required)
- `content`: The content to write (required)

**When to use:** Writing approved output after user confirmation at
stopping points (Steps 2b, 4b, 6b).
**When NOT to use:** Do not write until the user has approved the output
at the relevant stopping point.

## Workflow

### Step 1: Gather Inputs

**Goal:** Collect the source file path, target schema, and downstream
generation preferences from the user.

**Actions:**

1. **Ask** the user for the source file or stage path:
   ```
   What is the path to the source data?
   - Local file: /path/to/file.csv
   - Snowflake stage: @my_db.my_schema.my_stage/file.csv
   ```

2. **Ask** the user for the target schema:
   ```
   What is the target schema for the generated objects?
   Example: ANALYTICS_DB.RAW_SCHEMA
   ```
   Store as `{target_schema}`.

3. **Ask** the user which outputs to generate:
   ```
   What would you like to generate?
   1. Data Vault model only (hubs, links, satellites)
   2. Ingestion pipeline only (staging DDL + COPY INTO + vault load)
   3. Both — full onboarding package
   ```
   Store as `generate_dv = true | false` and `generate_pipeline = true | false`.

4. **Ask** the user (optional) for pipeline architecture preference:
   ```
   Pipeline layering:
   1. Two-layer (raw staging → raw vault)
   2. Three-layer (raw staging → hash staging → raw vault)
   ```
   Store as `pipeline_layers = 2 | 3`. Default: 3.

5. **Ask** the user (optional) for vault modeling style:
   ```
   Vault style:
   1. Full — generate hubs, links, and satellites
   2. Hub-only — generate hubs and satellites only (no link tables)
   ```
   Store as `vault_style = full | hub_only`. Default: full.

**Hard Constraints:**
- Do NOT execute any generated SQL — compile-only validation is permitted
- Do NOT assume column types without profiling — always detect from data
- Do NOT skip the staging layer — never COPY INTO directly to vault tables
- Do NOT generate vault entities for columns the user explicitly excluded
- Do NOT add indexes, clustering keys, or performance hints unless the
  user requests them

**⚠️ MANDATORY STOPPING POINT:** Confirm inputs before proceeding.

```
Confirmed inputs:
- Source: {source_path}
- Target schema: {target_schema}
- Generate: DV model ({generate_dv}), Pipeline ({generate_pipeline})
- Pipeline layers: {pipeline_layers}
- Vault style: {vault_style}

Proceed? (Approve / Modify)
```

**Next:** If approved, proceed to Step 2. If modifications requested,
re-gather the changed inputs.

### Step 2: Profile Source Data

**Goal:** Load the source data, detect column types, classify columns by
Data Vault role, and produce the source profile dictionary.

**Actions:**

1. **Read** the source file using the `Read` tool (first 500 rows for profiling)
2. **Detect column names** from the header row or schema
3. **Infer types** for each column using the type inference table (see Key
   Concepts). For each column:
   - Scan sample values for type patterns
   - Calculate null percentage and distinct ratio
   - Detect format patterns (email, date, phone, URL, code)
4. **Classify Data Vault roles** per column:
   - `HUB_BK`: High cardinality, low null rate, unique or near-unique — likely
     a business key (e.g., customer_id, order_number, product_code)
   - `LINK_FK`: References another entity's business key (e.g., customer_id
     in an orders table)
   - `SAT_ATTR`: Descriptive attributes — non-key columns with variable values
   - `METADATA`: Audit/system columns (created_at, updated_at, source_system)
   - `UNKNOWN`: Ambiguous — flag for user confirmation
5. **Identify hub entities** by grouping business key columns and inferring
   entity names from column naming patterns
6. **Assess quality risk** for each column using the quality risk classification
   rules (see Key Concepts). Flag `HIGH` risk columns for user confirmation.
7. **Build** the source profile dictionary from the above analysis
8. **Present** the profile summary grouped by Data Vault role

**Source of Truth Rules:**
- Profile from the actual data — do NOT assume column types from names alone
- Flag ambiguous types for user confirmation — do NOT silently pick a fallback
- Do NOT invent columns that are not in the source

**⚠️ MANDATORY STOPPING POINT:** Present the source profile and wait for
user approval.

```
Source Profile for {source_name}:

| Column | Inferred Type | Null % | DV Role | Hub Entity | Quality Risk |
|--------|--------------|--------|---------|------------|--------------|
| ...    | ...          | ...    | ...     | ...        | ...          |

⚠️ HIGH risk columns flagged above require confirmation.

Does this look correct? (Approve / Modify)
```

**Next:** If approved, proceed to Step 2b (Write Profile). If modifications
requested, return to Step 2 and re-profile. Maximum 3 modification rounds —
if still unresolved, present the final version and ask the user to edit
manually.

### Step 2b: Write Source Profile

**Goal:** Save the approved source profile dictionary to disk.

**Actions:**

1. **Write** the source profile dictionary as a markdown table to:
   `{target_schema}_source_profile.md`

**⚠️ MANDATORY STOPPING POINT:** Confirm file path before writing.

**Next:** Pass the source profile dictionary, source metadata, and user
preferences to the sub-skills:
- **If `generate_dv = true`**, invoke **SKILL_datavault.md**
  (dv-model-generator) → Steps 3-4
- **If `generate_pipeline = true`**, after DV model is approved (if
  applicable), invoke **SKILL_pipeline.md** (pipeline-generator) → Steps 5-6
- **If both**, run DV model first, then pipeline — the pipeline can use
  the vault entity definitions for load SQL

## Quality Gate

Before presenting the source profile, confirm:

- [ ] Every column in the source file appears in the profile dictionary
- [ ] Every column has an inferred type (no blanks)
- [ ] Every column has a `dv_role` classification (`HUB_BK`, `LINK_FK`, `SAT_ATTR`, `METADATA`, or `UNKNOWN`)
- [ ] Columns classified as `HUB_BK` have high cardinality (`distinct_ratio > 0.5`) and low null rate
- [ ] Columns classified as `UNKNOWN` are flagged with `quality_risk >= MEDIUM`
- [ ] At least one column is classified as `HUB_BK` (every source should have at least one business key)
- [ ] `hub_entity` is populated for every `HUB_BK` and `LINK_FK` column
- [ ] Quality risk levels follow the classification rules (no subjective assignments)
- [ ] No columns were invented — every entry traces to the actual source data
- [ ] Sample values are representative (not all NULLs or all identical)

## Output

A source profile dictionary containing column metadata (names, inferred types,
null percentages, distinct ratios, format patterns), Data Vault role classifications,
and hub entity assignments for each column. Passed to SKILL_datavault.md for
DV model generation and SKILL_pipeline.md for pipeline generation.

## Stopping Points Summary

| Checkpoint | Location | Purpose |
|------------|----------|---------|
| ✋ After Step 1 | Input gathering | Confirm source path, target schema, preferences |
| ✋ After Step 2 | Source profiling | Confirm inferred types and DV role classifications |
| ✋ After Step 2b | Profile file write | Save approved profile to disk |

**Resume rule:** Upon user approval, proceed directly without re-asking.
