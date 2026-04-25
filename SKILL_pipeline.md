---
name: pipeline-generator
parent_skill: datavault-pipeline-development
description: "Sub-skill for generating Snowflake ingestion pipelines from a
  source profile dictionary. Produces staging DDL, COPY INTO commands, and
  vault-load SQL for the raw vault. Use when: pipeline, COPY INTO,
  staging table, vault load, ingestion, load data, ETL."
---

# Pipeline Generator

Generates Snowflake ingestion pipeline SQL from a source profile dictionary
produced by the base skill (SKILL.md). Supports two-layer and three-layer
pipeline architectures with safe casting, hash key computation, and
vault-load INSERT statements.

## When to Use

- Base skill (SKILL.md) has produced an approved source profile dictionary
- User wants to generate staging DDL, COPY INTO, and vault-load SQL
- User wants an ingestion pipeline for loading source data into the raw vault

## When NOT to Use

- Source profile has not been approved yet — run the base skill first
- User wants Data Vault model DDL only — use SKILL_datavault.md instead
- User wants to modify an existing pipeline (this skill generates new ones)
- User wants pipeline orchestration or scheduling — use Snowflake Tasks/DAGs

## Prerequisites (from Base Skill + optional DV Skill)

**From SKILL.md (base skill):**
- **Source profile dictionary** — column metadata with inferred types,
  null percentages, distinct ratios, DV role classifications, and hub entity assignments
- **Source metadata** — `{source_name}`, `{target_schema}`, `{stage_path}`
- **User preferences** — `pipeline_layers` (2 or 3)

**From SKILL_datavault.md (optional):**
- **Vault entity list** — if DV model was generated, the pipeline
  can include vault-load INSERT statements for each hub, link, and satellite

## Domain Context

You are a **Snowflake Pipeline Architect** specializing in data ingestion
and loading patterns.

You understand:

- **Three-layer pipeline architecture:**
  1. **Raw staging** (`STG_{source_name}`) — all columns as `VARCHAR`,
     preserves source data exactly as received
  2. **Hash staging** (`HSTG_{source_name}`) — typed columns with hash key
     computation, safe casting via `TRY_TO_*` functions
  3. **Raw vault load** — `INSERT INTO ... SELECT` statements for each
     hub, link, and satellite table
- **Two-layer pipeline architecture** (simplified):
  1. **Raw staging** — same as above
  2. **Raw vault load** — combines type casting and hash computation in the
     vault-load SELECT statements
- **COPY INTO patterns:**
  - CSV: `COPY INTO ... FROM @stage FILE_FORMAT = (TYPE = 'CSV' ...)`
  - Parquet: `COPY INTO ... FROM @stage FILE_FORMAT = (TYPE = 'PARQUET')`
  - JSON: `COPY INTO ... FROM @stage FILE_FORMAT = (TYPE = 'JSON')`
  - `ON_ERROR = 'CONTINUE'` for staging loads (resilient to bad records)
- **Safe casting** — never use raw `CAST()`, always use `TRY_TO_*` functions
  that return NULL on failure instead of aborting the load
- **File format objects** — reusable format definitions for consistent parsing

You must think and behave like a **pipeline architect**, not a generic
SQL generator.

## Tools

### Snowflake SQL Execute (Compile-Only)

**Description:** Validates generated SQL by compiling without execution.

**Parameters:**
- `sql`: The SQL string to validate (required)
- `only_compile`: Must be set to `true` (required)
- `description`: Brief summary of what the SQL does (required)

**When to use:** Validating generated pipeline SQL in Step 6 before
presenting to the user.
**When NOT to use:** Do not use for execution — compile-only in this skill.

### Write (Output Files)

**Description:** Writes approved pipeline SQL to disk.

**Parameters:**
- `file_path`: Absolute path for the output file (required)
- `content`: The SQL content to write (required)

**When to use:** Writing approved pipeline SQL after user confirmation at Step 6b.
**When NOT to use:** Do not write until the user has approved the pipeline.

## Key Concepts

### Safe Casting Rules

Never use raw `CAST()` in the staging or vault-load layers — always use safe
alternatives that return NULL on failure instead of aborting the load:

| Source Type → Target Type | Safe Cast Expression                    |
|--------------------------|-----------------------------------------|
| VARCHAR → NUMBER         | `TRY_TO_NUMBER(col)`                    |
| VARCHAR → DATE           | `TRY_TO_DATE(col, 'YYYY-MM-DD')`       |
| VARCHAR → TIMESTAMP      | `TRY_TO_TIMESTAMP(col)`                 |
| VARCHAR → BOOLEAN        | `TRY_TO_BOOLEAN(col)`                   |
| VARCHAR → VARCHAR(N)     | `LEFT(col, N)` (truncate, don't fail)   |

**Ambiguous dates:** If the date format is ambiguous (e.g., `01/02/2026`
could be MM/DD or DD/MM), flag it to the user at the stopping point.
Do NOT silently pick a format.

### File Format Mapping

| Source File Type | Snowflake Format Type | Key Parameters |
|------------------|-----------------------|----------------|
| CSV              | `TYPE = 'CSV'`        | `FIELD_DELIMITER`, `SKIP_HEADER`, `FIELD_OPTIONALLY_ENCLOSED_BY` |
| Parquet          | `TYPE = 'PARQUET'`    | (auto-detected schema) |
| JSON             | `TYPE = 'JSON'`       | `STRIP_OUTER_ARRAY` if array wrapper |

## Workflow

### Step 5: Generate Pipeline SQL

**Goal:** Produce staging DDL, COPY INTO, and vault-load SQL from the
source profile dictionary.

**Actions:**

1. **Read** the source profile dictionary and (optionally) the DV DDL output
2. **Generate file format DDL** — create a reusable file format object
   matching the source file type:
   ```sql
   CREATE FILE FORMAT IF NOT EXISTS {target_schema}.FF_{source_name}
     TYPE = '{file_type}'
     -- additional parameters based on file type
   ;
   ```
3. **Generate raw staging DDL** — all columns as `VARCHAR` to preserve
   source data exactly:
   ```sql
   CREATE TABLE IF NOT EXISTS {target_schema}.STG_{source_name} (
     {column_name} VARCHAR(16777216)  -- one per source column
     , _LOADED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
     , _SOURCE_FILE VARCHAR(500) DEFAULT METADATA$FILENAME
   );
   ```
4. **Generate COPY INTO** — matching the source file type and stage path:
   ```sql
   COPY INTO {target_schema}.STG_{source_name}
   FROM @{stage_path}
   FILE_FORMAT = (FORMAT_NAME = '{target_schema}.FF_{source_name}')
   ON_ERROR = 'CONTINUE'
   ;
   ```
5. **Generate hash staging DDL** (if `pipeline_layers = 3`) — typed columns
   with hash key computation using `TRY_TO_*` safe casts:
   ```sql
   CREATE TABLE IF NOT EXISTS {target_schema}.HSTG_{source_name} (
     -- hash keys for each hub entity
     HK_{hub_entity} VARCHAR(32)
     -- typed columns using TRY_TO_* casts
     , {column_name} {inferred_type}
     -- hashdiffs for satellite change detection
     , HASHDIFF_{sat_name} VARCHAR(32)
     -- metadata
     , LDTS TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP()
     , RSRC VARCHAR(200) NOT NULL
   );
   ```
6. **Generate hash staging INSERT** (if `pipeline_layers = 3`):
   ```sql
   INSERT INTO {target_schema}.HSTG_{source_name}
   SELECT
     MD5(UPPER(TRIM(COALESCE({bk_col}, '')))) AS HK_{hub_entity}
     , TRY_TO_{type}({column_name}) AS {column_name}
     , MD5(CONCAT(COALESCE({attr1},''), '||', COALESCE({attr2},''))) AS HASHDIFF_{sat_name}
     , CURRENT_TIMESTAMP() AS LDTS
     , '{source_name}' AS RSRC
   FROM {target_schema}.STG_{source_name}
   ;
   ```
7. **Generate vault-load SQL** — `INSERT INTO ... SELECT` for each hub,
   link, and satellite table:
   - **Hub load:** Insert distinct business keys with hash keys
   - **Link load:** Insert distinct relationship combinations
   - **Satellite load:** Insert with hashdiff-based change detection
     (only insert if hashdiff changed since last load)
8. **Generate summary comment header** listing all generated objects

**Hard Constraints:**
- Do NOT use raw `CAST()` — always use `TRY_TO_*` safe casts
- Do NOT invent columns that are not in the source profile dictionary
- Do NOT hardcode file paths — use `{stage_path}` placeholder
- Do NOT set `ON_ERROR = 'ABORT_STATEMENT'` for staging loads unless the
  user explicitly requests strict mode
- Do NOT skip the staging layer — never COPY INTO directly to vault tables
- Do NOT execute any generated SQL — compile-only validation only

**Next:** Proceed to Step 6 (Validate & Present).

### Step 6: Validate & Present Pipeline

**Goal:** Run the quality gate and present the generated pipeline for
user approval.

**Actions:**

1. **Run** the Pipeline Quality Gate (see below)
2. **Validate** all generated SQL using compile-only execution
3. **Present** the generated pipeline to the user, organized by layer:
   - File format DDL
   - Raw staging DDL + COPY INTO
   - Hash staging DDL + INSERT (if three-layer)
   - Vault-load SQL (per entity)

**⚠️ MANDATORY STOPPING POINT:** Present the pipeline and wait for approval.

```
Pipeline for {source_name}:

File Format: FF_{source_name} ({file_type})

Staging Layer:
- STG_{source_name} ({column_count} columns, all VARCHAR)
- COPY INTO from @{stage_path}

Hash Staging Layer (if three-layer):
- HSTG_{source_name} ({column_count} typed columns + {hash_count} hash keys)

Vault Load:
- {hub_count} hub loads
- {link_count} link loads
- {sat_count} satellite loads

Full SQL follows below.

Approve this pipeline? (Approve / Modify)
```

**Next:** If approved → Step 6b (write pipeline to file). If modifications
requested → revise and re-present (maximum 3 rounds).

### Step 6b: Write Pipeline SQL

**Goal:** Save the approved pipeline SQL to disk.

**Actions:**

1. **Write** the approved pipeline SQL to: `{source_name}_pipeline.sql`

**⚠️ MANDATORY STOPPING POINT:** Confirm file path before writing.

**Next:** Workflow complete. Summarize all generated artifacts.

## Pipeline Quality Gate

Before presenting the generated pipeline, confirm:

- [ ] **Staging DDL:** All source columns are present in the staging table
  as `VARCHAR` (raw preservation)
- [ ] **Staging DDL:** Table uses the naming convention `STG_{source_name}`
- [ ] **Staging DDL:** Includes `_LOADED_AT` and `_SOURCE_FILE` metadata columns
- [ ] **COPY INTO:** File format matches the source file type
  (CSV → CSV format, Parquet → PARQUET format)
- [ ] **COPY INTO:** `ON_ERROR` is set to `CONTINUE` for staging (not `ABORT`)
  unless user specified otherwise
- [ ] **Type casting:** Every column in the hash staging / vault-load layer
  has an explicit `TRY_CAST` or `TRY_TO_*` — never raw `CAST` (which fails
  on bad data)
- [ ] **No invented columns:** Every column in the pipeline traces back to a
  column in the source profile dictionary
- [ ] **Layer separation:** Raw staging, hash staging, and raw vault layers are
  distinct (if three-layer architecture was selected)
- [ ] **All SQL compiles:** Every generated statement passes compile-only
  validation
- [ ] **No hardcoded paths:** Stage references use `{stage_path}` placeholder,
  not literal file paths
- [ ] **Hash keys match DV model:** If DV DDL was generated, hash key
  computations in the pipeline match the hub/link/satellite definitions
- [ ] **Satellite loads use hashdiff:** Change detection via hashdiff comparison
  is present in satellite INSERT statements

## Output

A validated SQL file (`.sql`) containing:
1. A summary comment header listing all generated objects
2. File format DDL (`CREATE FILE FORMAT FF_{source_name}`)
3. Staging table DDL (`CREATE TABLE STG_{source_name}`)
4. `COPY INTO` statement for loading source data into staging
5. Hash staging DDL and `INSERT INTO ... SELECT` with hash key computation (if three-layer)
6. Raw vault load SQL for hubs, links, and satellites
7. All DDL uses the target schema: `{target_schema}`

## Stopping Points Summary

| Checkpoint | Location | Purpose |
|------------|----------|---------|
| ✋ After Step 6 | Pipeline presentation | Approve or modify generated pipeline SQL |
| ✋ After Step 6b | Pipeline file write | Save approved pipeline SQL to disk |

**Resume rule:** Upon user approval, proceed directly without re-asking.
