---
id: all_properties
title: listing_mart.all_properties
sidebar_label: all_properties
---

# listing_mart.all_properties

## Table Overview

**Purpose**: Reference table mapping property IDs to human-readable attribute names. Used for decoding listing variations and attributes.

**Primary Key**: `property_id`

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `property_id` | INT64 | `etsy_index.properties` | Direct from properties table | Unique property identifier. Primary key |
| `ott_attribute_id` | INT64 | `etsy_index.properties` | Direct from properties table | OTT (Organic Taxonomy Tool) attribute ID. May be NULL or 0 if not mapped to OTT |
| `attribute_name` | STRING | `structured_data.attributes`, `etsy_index.properties` | If OTT mapped: uses attributes.name with 'ListingMetadata_Property_' removed. Otherwise: uses properties.name with 'ListingMetadata_Variation_' removed and 'TeeShirtSize' → 'Size' | Human-readable attribute name. Cleaned with prefixes removed |

---

## Data Quality

### Assertions

- **Uniqueness**: Table enforces unique property_id

### Data Sources

- **Primary**: `etsy_index.properties`
- **OTT Attributes**: `structured_data.attributes` (when ott_attribute_id is set)

---

## Business Logic

### Attribute Name Derivation

The `attribute_name` is cleaned using these transformations:

1. **If OTT attribute exists** (ott_attribute_id is not NULL and not 0):
   - Use `attributes.name` with prefix `'ListingMetadata_Property_'` removed

2. **If no OTT attribute**:
   - Use `properties.name` with transformations:
     - Remove prefix: `'ListingMetadata_Variation_'`
     - Replace: `'TeeShirtSize'` → `'Size'`

**Examples:**
- `'ListingMetadata_Property_Color'` → `'Color'`
- `'ListingMetadata_Variation_Size'` → `'Size'`
- `'ListingMetadata_Variation_TeeShirtSize'` → `'Size'`

---

## Common Use Cases

### Analytics Queries

**Get all property mappings:**
```sql
SELECT
    property_id,
    attribute_name,
    ott_attribute_id
FROM `etsy-data-warehouse-prod.listing_mart.all_properties`
ORDER BY attribute_name
```

**Find properties for a specific attribute:**
```sql
SELECT
    property_id,
    attribute_name
FROM `etsy-data-warehouse-prod.listing_mart.all_properties`
WHERE LOWER(attribute_name) = 'color'
```

**Properties with OTT mapping:**
```sql
SELECT
    property_id,
    attribute_name,
    ott_attribute_id
FROM `etsy-data-warehouse-prod.listing_mart.all_properties`
WHERE ott_attribute_id IS NOT NULL
AND ott_attribute_id != 0
```

---

## Display Formatting Rules

### Text Fields
- `attribute_name` - Display as-is, already cleaned and human-readable
- Use title case for display: e.g., "Color", "Size", "Material"

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.all_properties` for:**
- Looking up property_id → attribute_name mappings
- Finding which properties are OTT-mapped
- Decoding property IDs from raw data
- **Reference table** - Usually joined, rarely queried directly

**Don't use this table for:**
- Getting property values → Use `listing_mart.all_property_values`
- Listing-specific data → This is just a lookup table

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_index.properties` - Already the source
- `structured_data.attributes` - Already joined for OTT names

**✅ Do join to these mart tables**:
- `listing_mart.listing_variation_attributes` - Already uses this table
- `listing_mart.listing_all_attributes` - Already uses this table
- Use this when working with raw `etsy_shard.listing_variations` or `etsy_shard.listing_attributes`

### Performance Tips

1. **Small reference table** - Fast to scan, safe to join
2. **Use for decoding only** - Don't filter on attribute_name and then join; filter after joining
3. **Property names are cleaned** - No need to remove prefixes yourself

### Common Query Patterns

**Look up attribute name:**
```sql
SELECT property_id, attribute_name, ott_attribute_id
FROM `etsy-data-warehouse-prod.listing_mart.all_properties`
WHERE property_id = 200
```

**Find property ID by name:**
```sql
SELECT property_id, attribute_name
FROM `etsy-data-warehouse-prod.listing_mart.all_properties`
WHERE LOWER(attribute_name) = 'color'
```

**OTT-mapped properties:**
```sql
SELECT property_id, attribute_name, ott_attribute_id
FROM `etsy-data-warehouse-prod.listing_mart.all_properties`
WHERE ott_attribute_id IS NOT NULL
  AND ott_attribute_id != 0
ORDER BY attribute_name
```

---

## Related Tables

### Listing Mart Tables (Use This Table With)
- `listing_mart.all_property_values` - Value ID to value lookup (used together for property_id → name, value_id → value)
- `listing_mart.all_custom_property_names` - Custom property names per listing
- `listing_mart.listing_variation_attributes` - Uses this table for attribute_name
- `listing_mart.listing_all_attributes` - Uses this table for attribute_name
- `listing_mart.listing_attributes` - Uses this for color attribute name
- `listing_mart.listing_variations` - Variation counts reference properties
- `listing_mart.listing_products` - Product variations reference properties

### Other Mart Tables
- `listing_mart.listings` - Core listing data
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_current_discounts` - Active promotions

### Source Tables (Avoid - Data Already in Mart)
- `etsy_index.properties` - Raw property definitions (already here with cleaned names)
- `structured_data.attributes` - OTT attributes (already joined for ott_attribute_id mappings)
