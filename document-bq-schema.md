# Document BigQuery Schema

Creates comprehensive documentation for BigQuery schemas with source lineage, business logic, and query guidance.

## Usage

User provides:
- Schema name (e.g., `transaction_mart`)
- One or more SQL script paths/URLs containing CREATE TABLE statements
- Optional: GitHub repo path (defaults to `/Users/estein/bq-table-docs`)

## Process

### 1. Download and Parse SQL Scripts

For each SQL script:
- Download from GitHub/local path
- Parse all `CREATE TABLE` or `CREATE OR REPLACE TABLE` statements
- Extract table names, column definitions, source tables, and business logic
- Identify temp tables and trace their lineage back to source tables

### 2. Generate Table Documentation

For each table, create `{table_name}.md` with:

#### Front Matter
```yaml
---
id: {table_name}
title: {schema}.{table_name}
sidebar_label: {table_name}
---
```

#### Table Overview
- Purpose
- Primary Key
- Clustering
- Update Frequency
- Owner/Owner Team

#### Column Reference

Format:
```markdown
| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
```

**Guidelines:**
- **Source Table**: Actual BigQuery tables (e.g., `etsy_shard.listings`), NOT temp table names
- **Business Logic**: High-level summary of transformations, NOT SQL code
  - Examples: "Prefers min_price from inventory summary, falls back to listings.price"
  - NOT: `coalesce(ics.min_price, l.price)` with temp table references
- **Description**: Human-readable explanation with formatting rules (e.g., **STORED IN CENTS**)

For derived/calculated fields:
- Show what it's calculated from
- Explain the logic clearly
- Include special handling (CASE statements, defaults, etc.)

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

### 4. Update Root README

Update `/Users/estein/bq-table-docs/README.md` to include new schema section.

### 5. Commit and Push to GitHub

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
│   └── ...
└── transaction_mart/
    ├── README.md
    ├── receipts_gms.md
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
- ✅ All 12 tables in schema documented
- ✅ Related Tables section lists ALL tables in schema
- ✅ Query Guidance has real SQL examples
- ✅ Schema README has table index
- ✅ Root README updated with new schema

## Example Invocation

**User:** "Document the transaction_mart schema. The tables are created in these scripts:
- Rollups/auto/p2/daily/transaction_mart_receipts.sql
- Rollups/auto/p2/daily/transaction_mart_transactions.sql"

**Claude:**
1. Downloads both SQL scripts
2. Parses all CREATE TABLE statements
3. Creates docs for each table (receipts_gms, transactions_gms, etc.)
4. Creates transaction_mart/README.md
5. Updates root README.md
6. Commits and pushes to GitHub

## Notes

- Always simplify temp table references to original sources
- Use high-level business logic descriptions, not code
- Include comprehensive Query Guidance for every table
- Maintain consistent formatting across all docs
- Test that internal links work (table names in Related Tables)
