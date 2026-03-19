---
id: all_custom_property_names
title: listing_mart.all_custom_property_names
sidebar_label: all_custom_property_names
---

# listing_mart.all_custom_property_names

## Table Overview

**Purpose**: Maps custom property names for listing variations. Sellers can create custom variation labels (e.g., "Pattern" instead of standard "Design").

**Primary Key**: Composite key (`listing_id`, `property_id`)

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `etsy_shard.listing_variation_properties` | From listing_variation_properties, most recent by update_date per listing/property | Listing identifier. Part of composite primary key |
| `property_id` | INT64 | `etsy_shard.listing_variation_properties` | From listing_variation_properties | Property identifier. Part of composite primary key |
| `custom_name` | STRING | `etsy_index.custom_property_names` | Joined on property_name_id, cast from bytes to string | Custom property name. Seller-defined name for this variation property. Cast from bytes to string |

---

## Data Quality

### Assertions

- **Uniqueness**: Table enforces unique combination of (listing_id, property_id)

### Data Sources

- **Primary**: `etsy_shard.listing_variation_properties`
- **Names**: `etsy_index.custom_property_names`

---

## Business Logic

### Custom Name Selection

- Deduplicates by listing_id and property_id, taking the most recent (by update_date)
- Only includes properties with custom names (property_name_id exists)
- Converts byte-encoded names to strings

---

## Common Use Cases

### Analytics Queries

**Get custom names for a listing:**
```sql
SELECT
    listing_id,
    property_id,
    custom_name
FROM `etsy-data-warehouse-prod.listing_mart.all_custom_property_names`
WHERE listing_id = 12345
```

**Most common custom property names:**
```sql
SELECT
    custom_name,
    COUNT(DISTINCT listing_id) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.all_custom_property_names`
GROUP BY custom_name
ORDER BY listing_count DESC
LIMIT 100
```

**Listings with custom variation names:**
```sql
SELECT DISTINCT
    listing_id
FROM `etsy-data-warehouse-prod.listing_mart.all_custom_property_names`
```

---

## Display Formatting Rules

### Text Fields
- `custom_name` - Display as-is, seller-provided text
- May contain special characters or emojis

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.all_custom_property_names` for:**
- Finding seller-defined custom variation names
- Understanding which listings use custom property labels
- **Reference table** - Usually joined, rarely queried directly

**Don't use this table for:**
- Standard property names → Use `listing_mart.all_properties`
- Property values → Use `listing_mart.all_property_values`

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_shard.listing_variation_properties` - Already the source
- `etsy_index.custom_property_names` - Already joined

**✅ Do join to these mart tables**:
- `listing_mart.listing_variation_attributes` - Already uses this table (custom_name preferred over attribute_name)
- Use this when you specifically need to know if a listing has custom names

### Performance Tips

1. **Small table** - Only listings with custom names are included
2. **Already incorporated in listing_variation_attributes** - Usually don't need to join this directly
3. **Composite key (listing_id, property_id)** - Filter on both for best performance

### Common Query Patterns

**Get custom names for a listing:**
```sql
SELECT property_id, custom_name
FROM `etsy-data-warehouse-prod.listing_mart.all_custom_property_names`
WHERE listing_id = 12345
```

**Most popular custom names:**
```sql
SELECT
  custom_name,
  COUNT(DISTINCT listing_id) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.all_custom_property_names`
GROUP BY custom_name
ORDER BY listing_count DESC
LIMIT 100
```

**Listings with custom variation labels:**
```sql
SELECT DISTINCT listing_id
FROM `etsy-data-warehouse-prod.listing_mart.all_custom_property_names`
```

---

## Related Tables

### Listing Mart Tables (Use This Table With)
- `listing_mart.all_properties` - Standard property names (this provides custom overrides)
- `listing_mart.all_property_values` - Value ID to value lookup
- `listing_mart.listing_variation_attributes` - Uses this table for custom attribute names
- `listing_mart.listing_variations` - Variation summary
- `listing_mart.listing_products` - Product-level data with variations

### Other Mart Tables
- `listing_mart.listings` - Core listing data
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_attributes` - Attributes, taxonomy, recipient, color
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_current_discounts` - Active promotions
- `listing_mart.listing_all_attributes` - Comprehensive attribute data

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listing_variation_properties` - Source for listing-property pairs (already processed here)
- `etsy_index.custom_property_names` - Custom names (already joined and cast to string)
