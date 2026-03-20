---
id: listing_titles
title: etsy-data-warehouse-prod.listing_mart.listing_titles
sidebar_label: listing_titles
---

# listing_titles

Contains listing titles and descriptions, with automatic fallback to translated versions when original is empty.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_titles`

**Source Script:** `Rollups/auto/p2/daily/listing_mart.sql`

**Primary Key**: `listing_id`

**Clustering**: Clustered by `listing_id`

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `listing_mart.listings` | Direct from listings | Unique listing identifier. Primary key |
| `is_active` | INT64 | `listing_mart.listings` | Direct from listings | Active listing indicator from listings table |
| `listing_table_title` | STRING | `etsy_shard.listings` | Original title with HTML entities decoded: `&#39;` → `'`, `&quot;` → `"` | Original listing title from listings table with HTML entities decoded |
| `listing_table_description` | STRING | `etsy_shard.listings` | First 100 characters of description field | Original listing description truncated to first 100 characters |
| `title` | STRING | `etsy_shard.listings` (primary), `etsy_shard.listing_translations` (fallback) | Uses listing_table_title if not empty; falls back to translated_title (language=5 for English) if available; otherwise uses listing_table_title even if empty | Display title. Prefers original, falls back to English translation if original is empty |
| `description` | STRING | `etsy_shard.listings` (primary), `etsy_shard.listing_translations` (fallback) | Uses listing_table_description if not empty; falls back to translated_description (language=5) if available; otherwise uses listing_table_description even if empty | Display description. Prefers original, falls back to English translation if original is empty |
| `is_translated` | INT64 | Derived from title sources | Returns 1 if listing_table_title is empty AND translated_title exists, else 0 | Translation indicator. 1 if title came from translation, else 0 |

---

## Data Quality

### Assertions

- **Uniqueness**: Table enforces unique listing_id

### Data Sources

- **Primary**: `listing_mart.listings`
- **Original Titles**: `etsy_shard.listings` (title, description)
- **Translations**: `etsy_shard.listing_translations` (language=5, English translations)

---

## Business Logic

### Title Selection Logic

```
IF listing_table_title is empty THEN
    IF translated_title exists and is not empty THEN
        USE translated_title
    ELSE
        USE listing_table_title (empty)
    END IF
ELSE
    USE listing_table_title
END IF
```

### Description Selection Logic

Same logic as title - prioritizes original, falls back to translation.

### HTML Entity Decoding

- `&#39;` → `'` (apostrophe)
- `&quot;` or `&Quot;` → `"` (quotation mark)

---

## Common Use Cases

### Analytics Queries

**Get active listings with titles:**
```sql
SELECT
    listing_id,
    title,
    description,
    CASE WHEN is_translated = 1 THEN 'Translated' ELSE 'Original' END AS title_source
FROM `etsy-data-warehouse-prod.listing_mart.listing_titles`
WHERE is_active = 1
```

**Find listings using translated titles:**
```sql
SELECT
    listing_id,
    title,
    listing_table_title AS original_title
FROM `etsy-data-warehouse-prod.listing_mart.listing_titles`
WHERE is_translated = 1
```

**Search listings by title text:**
```sql
SELECT
    listing_id,
    title
FROM `etsy-data-warehouse-prod.listing_mart.listing_titles`
WHERE LOWER(title) LIKE '%search term%'
AND is_active = 1
```

---

## Display Formatting Rules

### Text Fields
- `title` - Display as-is, HTML entities already decoded
- `description` - Truncated to 100 characters, display with ellipsis if needed
- `listing_table_title` / `listing_table_description` - Raw originals, use `title`/`description` for display

### Translation Indicator
- `is_translated`: 1 = Title from translation, 0 = Original title
- Display with language badge if translated

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listing_titles` for:**
- Getting listing titles and descriptions for display
- Full-text search on titles
- Finding listings with translated titles
- Any query where you need to show or filter on title/description text

**Don't use this table for:**
- Price, availability, or other listing metadata → Use `listing_mart.listings`
- Attributes like color, recipient, taxonomy → Use `listing_mart.listing_attributes`

### Avoid These Joins

**❌ Don't join to these tables** (data already denormalized here):
- `etsy_shard.listings` - Titles already extracted with HTML entity decoding
- `etsy_shard.listing_translations` - Translation fallback logic already applied

**✅ Do join to these mart tables**:
- `listing_mart.listings` - For price, availability, shop info (join on listing_id)
- `listing_mart.listing_attributes` - For taxonomy/categorization (join on listing_id)

### Performance Tips

1. **Clustered by `listing_id`** - Always include listing_id in WHERE clause when possible
2. **For text search, use `LOWER(title) LIKE '%term%'`** - Case-insensitive search on the `title` field (not `listing_table_title`)
3. **Description is truncated to 100 chars** - Don't rely on it for full text, it's just a preview
4. **Use `is_active = 1`** - Filter to active listings first to reduce scan size

### Common Query Patterns

**Search listings by title:**
```sql
SELECT listing_id, title, is_translated
FROM `etsy-data-warehouse-prod.listing_mart.listing_titles`
WHERE is_active = 1
  AND LOWER(title) LIKE '%jewelry%'
LIMIT 100
```

**Get titles with prices:**
```sql
SELECT
  t.listing_id,
  t.title,
  l.price_usd / 100.0 AS price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listing_titles` t
JOIN `etsy-data-warehouse-prod.listing_mart.listings` l USING (listing_id)
WHERE t.is_active = 1
  AND l.is_active = 1
```

**Find translated titles:**
```sql
SELECT listing_id, title, listing_table_title AS original_title
FROM `etsy-data-warehouse-prod.listing_mart.listing_titles`
WHERE is_translated = 1
  AND is_active = 1
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listings` - Core listing data (price, availability, shop info, status flags)
- `listing_mart.listing_attributes` - Attributes, taxonomy, recipient, color
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_variations` - Variation summary
- `listing_mart.listing_variation_attributes` - Individual variation details
- `listing_mart.listing_products` - Product-level inventory and SKUs
- `listing_mart.listing_current_discounts` - Active promotions
- `listing_mart.listing_all_attributes` - Comprehensive attribute data

### Reference Tables
- `listing_mart.all_properties` - Property ID to name lookup
- `listing_mart.all_property_values` - Value ID to value lookup
- `listing_mart.all_custom_property_names` - Custom variation names

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listings` - Raw titles (HTML entities already decoded here)
- `etsy_shard.listing_translations` - Translations (fallback logic already applied here)
