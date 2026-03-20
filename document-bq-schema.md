# Document BigQuery Schema

Creates comprehensive documentation for BigQuery schemas with source lineage, business logic, and query guidance.

## Usage

User provides:
- Schema name (e.g., `transaction_mart`)
- One or more SQL script paths/URLs containing CREATE TABLE statements
- Optional: GitHub repo path (defaults to `/Users/estein/bq-table-docs`)

**IMPORTANT**: Always ask the user for the exact source script paths. Do NOT use wildcards (e.g., `transaction_mart_*.sql`) or generic patterns unless explicitly provided by the user. Each table should reference the specific script that creates it.

## Process

### 1. Download and Parse SQL Scripts

For each SQL script:
- Download from GitHub/local path using the exact path provided by the user
- Parse all `CREATE TABLE` or `CREATE OR REPLACE TABLE` statements
- Extract table names, column definitions, source tables, and business logic
- Identify temp tables and trace their lineage back to source tables
- **Map each table to its creating script** - do not use wildcards or generic patterns

### 2. Generate Table Documentation

For each table, create `{table_name}.md` with:

#### Front Matter
```yaml
---
id: {table_name}
title: {dataset}.{schema}.{table_name}
sidebar_label: {table_name}
---
```

#### Document Header
```markdown
# {table_name}

Brief one-line description of the table.
```

#### Table Overview
- **Full Table Name**: `` `{dataset}.{schema}.{table_name}` ``
- **Source Script**: `Rollups/auto/{priority}/daily/{exact_folder_path}/{exact_script_name}.sql`
  - **MUST be the exact script path**, not a wildcard pattern
  - If unknown, ask the user for the correct script path
- Purpose (detailed)
- Primary Key
- Clustering
- Update Frequency
- Owner/Owner Team

#### Column Reference

**REQUIRED FORMAT** - ALL columns MUST be documented with this 5-column table:

```markdown
| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `etsy_shard.listings` | Primary Key | Listing ID. Primary Key |
| `price_usd` | INT64 | `etsy_shard.listings`, temp: `inventory_summary` | Prefers min_price from inventory (channel=1, state=1), falls back to listings.price | **Price in USD cents**. Divide by 100 for dollars |
```

**CRITICAL REQUIREMENTS:**
- **ALL columns** must be documented (no grouped descriptions or shortcuts)
- **MUST use the 5-column format** exactly as shown above
- Do NOT use simplified 3-column tables or grouped descriptions
- Do NOT use section headers like "### Core Metrics" followed by partial columns

**Column-by-Column Guidelines:**

**Column**: Actual column name with backticks (e.g., `` `listing_id` ``)

**Type**: BigQuery data type (INT64, STRING, NUMERIC, FLOAT64, BOOL, DATE, TIMESTAMP, etc.)

**Source Table**:
- Actual BigQuery tables (e.g., `etsy_shard.listings`), NOT temp table names
- For joins: list multiple sources separated by commas
- For temp tables: show as `table_name`, temp: `temp_table_name` then trace in Business Logic
- For calculated fields: "Calculated" or the calculation source
- Examples:
  - `transaction_mart.all_transactions`
  - `etsy_index.users_index`, `etsy_shard.user_details`
  - Temp table: `buyatt_mart.analytics_clv`

**Business Logic**:
- High-level summary of transformations, NOT SQL code
- Examples:
  - "Prefers min_price from inventory summary, falls back to listings.price"
  - "Sum where live = 1"
  - "`ntile(10)` over expectedgms104 DESC"
  - "CASE WHEN platform = 'desktop' THEN 1 END, summed"
  - "Direct" (for pass-through columns)
  - "Primary Key"
- NOT: `coalesce(ics.min_price, l.price)` with temp table references

**Description**:
- Human-readable explanation with formatting rules
- Include important notes in **bold** (e.g., **STORED IN CENTS**, **PII**, **USD dollars**)
- Be concise but complete
- Examples:
  - "**Price in USD cents**. Divide by 100 for dollars"
  - "**PII**: User email (lowercase, trimmed)"
  - "Desktop visits for the day"

**For derived/calculated fields:**
- Show what it's calculated from
- Explain the logic clearly
- Include special handling (CASE statements, defaults, etc.)
- Example:
  ```
  | `account_age` | INT64 | Calculated | `Floor((UNIX_SECONDS(current_date) - join_date) / 86400)` | Days since account creation |
  ```

**For tables with many similar columns (platform breakdowns, etc.):**
- Still document EVERY column individually
- Use clear patterns but don't group
- You can add section comment rows for readability:
  ```markdown
  | **Platform OS Metrics** | | | | **Desktop, iOS, Android, Other, Undefined** |
  | `desktop_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'desktop' THEN 1 END)` | Desktop visits |
  | `ios_os_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'ios' THEN 1 END)` | iOS visits |
  ```

**Examples of INCORRECT formats to AVOID:**

❌ **Don't use simplified 3-column tables:**
```markdown
| Column | Type | Description |
|--------|------|-------------|
```

❌ **Don't use grouped descriptions:**
```markdown
### Platform Metrics (Desktop, iOS, Android)
- `{platform}_visits` - Visit count by platform
- `{platform}_conv_visits` - Converting visits by platform
```

❌ **Don't skip columns with "see above" or "similar to"**

✅ **Always use the complete 5-column format for EVERY column**

#### Query Guidance Section

**When to Use This Table:**
- Specific use cases (bullet points)
- What NOT to use this table for (point to alternatives)

**Avoid These Joins:**
- ❌ Don't join - Tables where data is already denormalized
- ✅ Do join - Recommended mart tables to combine with

**Performance Tips:**
- Clustering column guidance
- Query optimization hints
- Common gotchas
- Data type handling tips

**Common Query Patterns:**
- 3-5 real SQL examples
- Copy-paste ready
- Cover typical use cases

#### Related Tables Section

Organized into 3 groups:

**{Schema} Tables (Join on {key}):**
- List ALL tables in the same schema
- Brief description of each

**Reference Tables:**
- Lookup tables used to build this table

**Source Tables (Avoid - Data Already in Mart):**
- Raw tables to avoid joining (data already here)

### 3. Generate Schema README

Create `{schema}/README.md` with:
- Schema overview and purpose
- Table index (organized by category)
- Common query patterns
- Important formatting rules (money, dates, etc.)
- Best practices
- Support information
- Source scripts section listing all rollup scripts

### 4. Document Materialized Views (if applicable)

For schemas with materialized views:
- Create `{schema}/materialized_views/README.md`
- Document each materialized view with:
  - Base table reference
  - Filter logic (e.g., `WHERE is_active = 1`)
  - Performance benefits
  - When to use vs base table
- Include comparison examples

### 5. Update Root README

Update `/Users/estein/bq-table-docs/README.md` to include:
- New schema section with all tables listed by category
- Links to all table documentation
- Links to materialized views (if applicable)

### 6. Update Existing Documentation Format

When updating existing table documentation:
- Add **Full Table Name** field with complete `dataset.schema.table` path
- Add **Source Script** field showing which rollup creates the table
- Update frontmatter `title:` to include dataset prefix
- Simplify h1 heading to just table name (not schema.table)
- Maintain consistency with newly created docs

### 7. Commit and Push to GitHub

```bash
cd /Users/estein/bq-table-docs
git add {schema}/
git commit -m "Add {schema} documentation"
git push origin main
```

## SQL Parsing Guidelines

### Identify Source Tables

**Trace through temp tables:**
1. Find final table creation: `CREATE TABLE schema.table_name`
2. For each column, trace back through:
   - Direct table references (e.g., `l.listing_id` from `listings l`)
   - Temp table references (e.g., `ics.min_price` from temp table)
   - For temp tables, recursively trace to original source
3. Report the ORIGINAL source table, not the temp table name

**Example:**
```sql
-- Temp table
CREATE TEMP TABLE inventory_summary AS
  SELECT listing_id, min_price FROM etsy_shard.listing_inventory_channel_summary;

-- Final table
CREATE TABLE mart.listings AS
  SELECT ics.min_price FROM inventory_summary ics;
```

**Correct documentation:**
- Source Table: `etsy_shard.listing_inventory_channel_summary`
- Business Logic: "From min_price in inventory summary (channel=1, state=1)"

**Wrong documentation:**
- Source Table: `inventory_summary` ❌ (this is a temp table)

### Simplify Business Logic

**Transform SQL to summaries:**

❌ **Don't write:**
```
`coalesce(ics.min_price, l.price)` where ics from inventory_summary temp table
```

✅ **Do write:**
```
Prefers min_price from inventory summary (channel=1, state=1, latest by update_date), falls back to listings.price
```

**For CASE statements:**
```
Returns 1 if ALL conditions met: admin_flags=0, state=0, is_available=1. Otherwise 0
```

**For calculations:**
```
Multiply price by (market_rate/1e7), cast to int64
```

## Output Format

### Directory Structure
```
bq-table-docs/
├── README.md
├── listing_mart/
│   ├── README.md
│   ├── listings.md
│   ├── listing_indicators.md
│   ├── listing_gms.md
│   ├── materialized_views/
│   │   └── README.md
│   └── ...
└── transaction_mart/
    ├── README.md
    ├── receipts_gms.md
    ├── transaction_obt.md
    └── ...
```

### File Naming
- Use snake_case matching table names
- One file per table
- Schema README always named `README.md`

## Quality Checks

Before committing:
- ✅ All source tables traced to original sources (no temp tables)
- ✅ Business logic is readable summaries, not code
- ✅ All tables in schema documented
- ✅ Full table names include dataset prefix (e.g., `etsy-data-warehouse-prod.listing_mart.table`)
- ✅ Source script referenced in each table doc
- ✅ Frontmatter title includes full dataset.schema.table path
- ✅ H1 heading uses simple table name only
- ✅ Related Tables section lists ALL tables in schema
- ✅ Query Guidance has real SQL examples
- ✅ Schema README has table index organized by category
- ✅ Schema README lists all source scripts
- ✅ Materialized views documented (if applicable)
- ✅ Root README updated with new schema and tables

## Example Invocations

### Example 1: Document New Schema

**User:** "Document the transaction_mart schema. The tables are created in these scripts:
- Rollups/auto/p2/daily/transaction_mart_receipts.sql
- Rollups/auto/p2/daily/transaction_mart_transactions.sql"

**Claude:**
1. Downloads both SQL scripts from GitHub
2. Parses all CREATE TABLE statements
3. Creates docs for each table with full table names and source scripts
4. Creates transaction_mart/README.md with table index
5. Updates root README.md
6. Commits and pushes to GitHub

### Example 2: Add Tables to Existing Schema

**User:** "Add documentation for these listing_mart tables from these scripts:
- Rollups/auto/p2/daily/listing_indicators.sql
- Rollups/auto/p2/daily/listing_mart_bucket3.sql"

**Claude:**
1. Downloads SQL scripts
2. Parses CREATE TABLE statements
3. Creates new table docs (listing_indicators.md, listing_gms.md, etc.)
4. Updates listing_mart/README.md to include new tables in appropriate categories
5. Updates root README.md
6. Commits and pushes to GitHub

### Example 3: Update Existing Documentation Format

**User:** "Update all existing table docs to include full table names and source scripts"

**Claude:**
1. Reads all existing .md files in schema folders
2. Updates each file:
   - Adds **Full Table Name** field
   - Adds **Source Script** field
   - Updates frontmatter title
   - Simplifies h1 heading
3. Commits all changes with descriptive message
4. Pushes to GitHub

### Example 4: Document Materialized Views

**User:** "Document the listing_mart materialized views from listing_mart_mvs.sql"

**Claude:**
1. Downloads listing_mart_mvs.sql
2. Identifies all CREATE MATERIALIZED VIEW statements
3. Creates materialized_views/README.md
4. Documents each view with base table, filter, and usage guidance
5. Updates listing_mart/README.md to link to materialized views
6. Commits and pushes to GitHub

## Notes

- Always simplify temp table references to original sources
- Use high-level business logic descriptions, not code
- Include comprehensive Query Guidance for every table
- Maintain consistent formatting across all docs
- Test that internal links work (table names in Related Tables)
- **CRITICAL**: Never use wildcard patterns for source scripts (e.g., `transaction_mart_*.sql`)
  - Always use the exact script path that creates the table
  - If you don't know the exact script, ask the user instead of guessing
  - Download and parse scripts to map tables to their creating scripts
