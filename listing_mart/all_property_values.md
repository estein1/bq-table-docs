---
id: all_property_values
title: listing_mart.all_property_values
sidebar_label: all_property_values
---

# listing_mart.all_property_values

## Table Overview

**Purpose**: Reference table mapping value IDs to human-readable attribute values. Used for decoding listing variation values (e.g., "Red", "Large", "Cotton").

**Primary Key**: `value_id`

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `value_id` | INT64 | `etsy_index.property_option_inputs` | Direct from property_option_inputs | Unique value identifier. Primary key |
| `attribute_value` | STRING | `structured_data.values`, `etsy_index.property_option_inputs` | If OTT mapped: uses values.name. Otherwise: uses property_option_inputs.value. HTML entities decoded: `&#39;` → `'`, `&quot;` → `"` | Human-readable attribute value. HTML entities decoded. From structured_data.values if OTT mapped, otherwise from property_option_inputs.value |
| `attribute_type` | STRING | Derived from source | Returns 'ott sd' if from structured_data.values, otherwise 'ott'. Truncated to 10 chars | Source type indicator. 'ott sd' = from structured_data, 'ott' = from property_option_inputs |
| `ott_value_id` | INT64 | `etsy_index.property_option_inputs` | From property_option_inputs. Returns NULL if ott_value_id = 0 | OTT (Organic Taxonomy Tool) value ID. NULL if ott_value_id = 0 |
| `scale_name` | STRING | `structured_data.scales` | Joined from scales table on scale_id, or parsed from legacy_scale field | Scale/unit name if applicable. E.g., "Inches", "Centimeters", "Ounces" |

---

## Data Quality

### Assertions

- **Uniqueness**: Table enforces unique value_id

### Data Sources

- **Primary**: `etsy_index.property_option_inputs`
- **OTT Values**: `structured_data.values` (when ott_value_id is set)
- **Scales**: `structured_data.scales` (for measurement units)

---

## Business Logic

### Attribute Value Derivation

1. **If OTT value exists** (ott_value_id is not NULL and not 0):
   - Use `structured_data.values.name`
   - Set `attribute_type = 'ott sd'`

2. **If no OTT value**:
   - Use `property_option_inputs.value` (cast to string)
   - Set `attribute_type = 'ott'`

3. **HTML Entity Decoding**:
   - `&#39;` → `'` (apostrophe)
   - `&quot;` or `&Quot;` → `"` (quotation mark)

### Scale Name Derivation

- Looks up scale from `structured_data.scales` using scale_id
- Falls back to legacy_scale mappings if scale_id is NULL
- Returns NULL if no scale applies

---

## Common Use Cases

### Analytics Queries

**Get all attribute values:**
```sql
SELECT
    value_id,
    attribute_value,
    scale_name,
    attribute_type
FROM `etsy-data-warehouse-prod.listing_mart.all_property_values`
ORDER BY attribute_value
```

**Find values for a specific attribute (e.g., colors):**
```sql
SELECT
    value_id,
    attribute_value
FROM `etsy-data-warehouse-prod.listing_mart.all_property_values`
WHERE LOWER(attribute_value) LIKE '%red%'
   OR LOWER(attribute_value) LIKE '%blue%'
```

**Values with measurement scales:**
```sql
SELECT
    value_id,
    attribute_value,
    scale_name
FROM `etsy-data-warehouse-prod.listing_mart.all_property_values`
WHERE scale_name IS NOT NULL
ORDER BY scale_name, attribute_value
```

**Structured data vs OTT values:**
```sql
SELECT
    attribute_type,
    COUNT(*) AS value_count
FROM `etsy-data-warehouse-prod.listing_mart.all_property_values`
GROUP BY attribute_type
```

---

## Display Formatting Rules

### Text Fields
- `attribute_value` - Display as-is, HTML entities already decoded
- `scale_name` - Display with value if present (e.g., "10 Inches")

### Type Indicators
- `attribute_type`:
  - 'ott sd' = From structured data (more standardized)
  - 'ott' = From property inputs (may be custom)

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.all_property_values` for:**
- Looking up value_id → attribute_value mappings
- Finding measurement scales/units
- Decoding value IDs from raw data
- **Reference table** - Usually joined, rarely queried directly

**Don't use this table for:**
- Getting property names → Use `listing_mart.all_properties`
- Listing-specific data → This is just a lookup table

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_index.property_option_inputs` - Already the source
- `structured_data.values` - Already joined for OTT values
- `structured_data.scales` - Already joined for scale names

**✅ Do join to these mart tables**:
- `listing_mart.listing_variation_attributes` - Already uses this table
- `listing_mart.listing_all_attributes` - Already uses this table
- Use this when working with raw `etsy_shard.listing_variations` or `etsy_shard.listing_attributes`

### Performance Tips

1. **Small reference table** - Fast to scan, safe to join
2. **HTML entities already decoded** - No need to clean values
3. **Use for decoding only** - Don't filter on attribute_value and then join

### Common Query Patterns

**Look up value:**
```sql
SELECT value_id, attribute_value, scale_name, attribute_type
FROM `etsy-data-warehouse-prod.listing_mart.all_property_values`
WHERE value_id = 12345
```

**Find values with measurements:**
```sql
SELECT value_id, attribute_value, scale_name
FROM `etsy-data-warehouse-prod.listing_mart.all_property_values`
WHERE scale_name IS NOT NULL
ORDER BY scale_name, attribute_value
```

**Search for specific values:**
```sql
SELECT value_id, attribute_value, attribute_type
FROM `etsy-data-warehouse-prod.listing_mart.all_property_values`
WHERE LOWER(attribute_value) LIKE '%red%'
```

---

## Related Tables

### Listing Mart Tables (Use This Table With)
- `listing_mart.all_properties` - Property ID to name lookup (used together for property_id → name, value_id → value)
- `listing_mart.all_custom_property_names` - Custom property names per listing
- `listing_mart.listing_variation_attributes` - Uses this table for attribute_value and scale_name
- `listing_mart.listing_all_attributes` - Uses this table for attribute_value
- `listing_mart.listing_attributes` - Uses this for color values
- `listing_mart.listing_variations` - Variation data
- `listing_mart.listing_products` - Product variations

### Other Mart Tables
- `listing_mart.listings` - Core listing data
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_current_discounts` - Active promotions

### Source Tables (Avoid - Data Already in Mart)
- `etsy_index.property_option_inputs` - Raw values (already here with HTML decoded)
- `structured_data.values` - OTT values (already joined for ott_value_id mappings)
- `structured_data.scales` - Scales/units (already joined for scale_name)
