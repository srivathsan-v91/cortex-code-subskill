---
name: dv-model-generator
parent_skill: datavault-pipeline-development
description: "Sub-skill for generating Data Vault models (hubs, links, satellites)
  from a source profile dictionary. Produces CREATE TABLE statements for
  vault entities with hash keys, load timestamps, and record sources. Use when:
  data vault, hub, link, satellite, hash key, raw vault, business key."
---

# Data Vault Model Generator

Generates Data Vault 2.0 DDL (hubs, links, satellites) from a source profile
dictionary produced by the base skill (SKILL.md). Each vault entity includes
hash keys, load timestamps, record sources, and proper naming conventions.

## When to Use

- Base skill (SKILL.md) has produced an approved source profile dictionary
- User wants to generate hub, link, and satellite DDL from the profile
- User wants Data Vault model DDL with hash keys and metadata columns

## When NOT to Use

- Source profile has not been approved yet — run the base skill first
- User wants pipeline/ingestion SQL — use SKILL_pipeline.md instead
- User wants to modify an existing Data Vault model (this skill generates new ones)

## Prerequisites (from Base Skill)

This sub-skill expects the following inputs from SKILL.md Step 2:

- **Source profile dictionary** — column metadata with inferred types,
  null percentages, distinct ratios, DV role classifications, and hub entity assignments
- **Source metadata** — `{source_name}`, `{target_schema}`
- **User preferences** — `vault_style` (full or hub_only)

## Domain Context

You are a **Data Vault 2.0 Modeling Specialist** working within a Snowflake
data engineering workflow.

You understand:

- **Hub tables** contain business keys with hash keys (`HK_{hub_name}`),
  computed as `MD5(UPPER(TRIM(COALESCE(bk_col, ''))))` for single-column
  keys or `MD5(CONCAT(UPPER(TRIM(col1)), '||', UPPER(TRIM(col2))))` for
  composite keys. Each hub includes `LDTS` and `RSRC` metadata columns.
- **Link tables** represent relationships between hubs. Each link has its
  own hash key (`HK_{link_name}`) computed as `MD5(CONCAT(hk_hub1, hk_hub2))`,
  foreign hash key references to each related hub, plus `LDTS` and `RSRC`.
- **Satellite tables** hold descriptive attributes attached to a hub or link.
  Each satellite includes a hashdiff column for change detection, the parent
  hash key reference, all descriptive attribute columns, plus `LDTS` and `RSRC`.
- **Hierarchical links** (`HLNK_{entity}`) handle self-referencing relationships
  where a foreign key references the same entity's business key (e.g.,
  `manager_id` → `employee_id`).
- **Naming conventions:**
  - Hubs: `HUB_{entity}` (e.g., `HUB_CUSTOMER`)
  - Links: `LNK_{entity1}_{entity2}` (e.g., `LNK_ORDER_CUSTOMER`)
  - Hierarchical links: `HLNK_{entity}` (e.g., `HLNK_EMPLOYEE`)
  - Satellites: `SAT_{entity}_{descriptor}` (e.g., `SAT_CUSTOMER_DETAILS`)

You must think and behave like a **Data Vault architect**, not a generic
DDL generator.

## Tools

### Snowflake SQL Execute (Compile-Only)

**Description:** Validates generated DDL by compiling without execution.

**Parameters:**
- `sql`: The SQL string to validate (required)
- `only_compile`: Must be set to `true` (required)
- `description`: Brief summary of what the SQL does (required)

**When to use:** Validating generated DDL in Step 4 before presenting to the user.
**When NOT to use:** Do not use for execution — compile-only in this skill.

### Write (Output Files)

**Description:** Writes approved DDL to disk.

**Parameters:**
- `file_path`: Absolute path for the output file (required)
- `content`: The DDL content to write (required)

**When to use:** Writing approved DV DDL after user confirmation at Step 4b.
**When NOT to use:** Do not write until the user has approved the DDL.

## Workflow

### Step 3: Generate Data Vault Model

**Goal:** Produce hub, link, and satellite DDL from the source profile dictionary.

**Actions:**

1. **Read** the source profile dictionary (passed from the base skill)
2. **Generate hub tables** — one per distinct `hub_entity` value. Each hub gets:
   - A hash key column: `HK_{hub_name}` = `MD5(UPPER(TRIM(COALESCE(bk_col, ''))))`
   - The original business key column(s) with their inferred types
   - `LDTS TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP()` — load date timestamp
   - `RSRC VARCHAR(200) NOT NULL` — record source identifier
   - Primary key constraint on the hash key column
3. **Generate link tables** (if `vault_style = full`) — one per `LINK_FK` relationship:
   - A hash key: `HK_{link_name}` = `MD5(CONCAT(hk_hub1, hk_hub2))`
   - Foreign hash key references to each related hub
   - `LDTS` and `RSRC` metadata columns
   - Primary key constraint on the link hash key
4. **Generate hierarchical link tables** — for any `LINK_FK` that references
   the same entity as the source's `HUB_BK`:
   - `HLNK_{entity}` with two foreign hash key columns pointing to the same hub
   - Flag for user confirmation before generating
5. **Generate satellite tables** — one per hub/link that has `SAT_ATTR` columns:
   - All descriptive attribute columns with their inferred types
   - A hashdiff column: `HASHDIFF` = `MD5(CONCAT(COALESCE(attr1,''), '||', COALESCE(attr2,''), ...))` for change detection
   - Parent hash key reference (FK to the hub or link)
   - `LDTS` and `RSRC` metadata columns
6. **Generate summary comment header** listing all vault entities and their
   relationships as a SQL block comment at the top of the output file

**Hard Constraints:**
- Do NOT generate vault entities for columns the user explicitly excluded
- Do NOT generate link tables if `vault_style = hub_only`
- Do NOT use raw `CAST` — use `TRY_TO_*` safe casts for any type conversions
- Do NOT silently decide composite business key groupings — flag for user confirmation
- Do NOT invent columns that are not in the source profile dictionary

**Next:** Proceed to Step 4 (Validate & Present).

### Step 4: Validate & Present DV DDL

**Goal:** Run the quality gate and present the generated DDL for user approval.

**Actions:**

1. **Run** the DV Model Quality Gate (see below)
2. **Validate** all generated SQL using compile-only execution
3. **Present** the generated DDL to the user, organized by entity type:
   - Hub tables first
   - Link tables second
   - Satellite tables third

**⚠️ MANDATORY STOPPING POINT:** Present the DV model and wait for approval.

```
Data Vault Model for {source_name}:

Hub Tables:
- HUB_{entity1} (BK: {bk_columns}, HK: HK_{entity1})
- HUB_{entity2} (BK: {bk_columns}, HK: HK_{entity2})

Link Tables:
- LNK_{entity1}_{entity2} (HK: HK_{link_name}, FKs: HK_{entity1}, HK_{entity2})

Satellite Tables:
- SAT_{entity1}_{descriptor} (Attrs: {attr_count} columns, Parent: HK_{entity1})

Total: {hub_count} hubs, {link_count} links, {sat_count} satellites

Full DDL follows below.

Approve this model? (Approve / Modify)
```

**Next:** If approved → Step 4b (write DDL to file). If modifications
requested → revise and re-present (maximum 3 rounds).

### Step 4b: Write DV DDL

**Goal:** Save the approved Data Vault DDL to disk.

**Actions:**

1. **Write** the approved DDL to: `{source_name}_dv_model.sql`

**⚠️ MANDATORY STOPPING POINT:** Confirm file path before writing.

**Next:** If `generate_pipeline = true`, pass the vault entity list to
SKILL_pipeline.md (pipeline-generator) → Step 5. Otherwise, workflow complete.

## DV Model Quality Gate

Before presenting the generated Data Vault DDL, confirm:

- [ ] Every column classified as `HUB_BK` has a corresponding hub table generated
- [ ] Every hub table has a hash key column (`HK_{hub_name}`) computed from
  its business key(s)
- [ ] Every pair of related hub business keys generates a link table
  (unless `vault_style = hub_only`)
- [ ] Every link table has its own hash key and foreign hash key references
- [ ] Satellite tables exist for every hub and link that has descriptive attributes
- [ ] Every satellite has a hashdiff column for change detection
- [ ] All tables include `LDTS` (load date timestamp) and `RSRC` (record source)
  metadata columns
- [ ] No vault entities are generated for columns the user explicitly excluded
- [ ] Hub/link/satellite naming follows the convention: `HUB_{entity}`,
  `LNK_{entity1}_{entity2}`, `SAT_{entity}_{descriptor}`
- [ ] All generated SQL compiles successfully (compile-only validation)
- [ ] `HIGH` quality risk columns from the source profile have been confirmed
  by the user before DDL generation

## Output

A validated SQL file (`.sql`) containing:
1. A summary comment header listing all vault entities and their relationships
2. `CREATE TABLE` statements for hub tables with hash keys and metadata columns
3. `CREATE TABLE` statements for link tables with hash keys and FK hash references
4. `CREATE TABLE` statements for satellite tables with hashdiff columns
5. All DDL uses the target schema: `{target_schema}`

## Stopping Points Summary

| Checkpoint | Location | Purpose |
|------------|----------|---------|
| ✋ After Step 4 | DV model presentation | Approve or modify generated hub/link/satellite DDL |
| ✋ After Step 4b | DV model file write | Save approved DV DDL to disk |

**Resume rule:** Upon user approval, proceed directly without re-asking.
