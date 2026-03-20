---
id: listing_attributes
title: etsy-data-warehouse-prod.listing_mart.listing_attributes
sidebar_label: listing_attributes
---

# listing_attributes

Key listing attributes including recipient, color, digital/personalizable flags, and taxonomy categorization. One row per listing.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_attributes`

**Source Script:** `Rollups/auto/p2/daily/listing_mart.sql`

**Primary Key**: `listing_id`

**Clustering**: Clustered by `listing_id`

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

### Identifiers & Status

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `listing_mart.listings` | Direct from listings | Unique listing identifier. Primary key |
| `is_active` | INT64 | `listing_mart.listings` | Direct from listings | Active listing indicator from listings table |

### Listing Properties

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `recipient` | STRING | `etsy_shard.listings_properties_values`, `etsy_index.property_options` | From property_id=266817057, joins to property_options for name. Most recent by modified date | Who the item is for. E.g., "Women", "Men", "Kids", "Unisex Adult". NULL if not set |
| `color` | STRING | `etsy_shard.listing_attributes`, `etsy_index.property_option_inputs` | From property_id=200 (color attribute), state=1, language=0. Most recent by update_date | Primary color. NULL if not set |
| `is_digital` | INT64 | `etsy_shard.listings_properties_values` | From property_id=2, value between '0' and '9'. Most recent by modified date. Defaults to 0 | Digital download listing. 1 if digital download, 0 otherwise |
| `is_personalizable` | INT64 | `etsy_shard.listings_properties_values` | From property_id=54. Most recent by modified date. Defaults to 0 | Can be personalized. 1 if personalizable, 0 otherwise |

### Taxonomy

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `taxonomy_id` | INT64 | `materialized.listing_categories_taxonomy` | Direct from taxonomy table | Taxonomy ID. Current taxonomy classification |
| `top_category` | STRING | `materialized.listing_categories_taxonomy` | Direct from top_level_cat_new field | Top-level category. First level of taxonomy path (e.g., "Jewelry", "Home & Living") |
| `second_level_category` | STRING | `materialized.listing_categories_taxonomy` | Direct from second_level_cat_new field | Second-level category. Second level of taxonomy path (e.g., "Necklaces", "Home Decor") |
| `third_level_category` | STRING | `materialized.listing_categories_taxonomy` | Direct from third_level_cat_new field | Third-level category. More specific subcategory |
| `full_path` | STRING | `materialized.listing_categories_taxonomy` | Direct from full_path field | Complete taxonomy path. Full taxonomy hierarchy, dot-separated |
| `taxo_flags` | INT64 | `materialized.listing_categories_taxonomy` | Direct from taxo_flags field | Taxonomy flags. Bitwise flags for taxonomy-related properties |

---

## Data Quality

### Data Sources

- **Primary**: `listing_mart.listings`
- **Recipient**: `etsy_shard.listings_properties_values` (property_id=266817057) joined with `etsy_index.property_options`
- **Color**: `etsy_shard.listing_attributes` (property_id=200, state=1) joined with `etsy_index.property_option_inputs`
- **Digital**: `etsy_shard.listings_properties_values` (property_id=2)
- **Personalizable**: `etsy_shard.listings_properties_values` (property_id=54)
- **Taxonomy**: `materialized.listing_categories_taxonomy`

---

## Business Logic

### Property Selection

For properties that can change over time (recipient, color, digital, personalizable):
- Takes the most recent value based on `modified` or `update_date`
- Deduplicates using row_number() partitioned by listing_id

### Default Values

- `is_digital`: Defaults to 0 if not set
- `is_personalizable`: Defaults to 0 if not set
- `recipient`, `color`: NULL if not set

### Taxonomy Path

- `full_path` format: "Top Category.Second Category.Third Category"
- Derived from current taxonomy classification
- Categories may be NULL if not classified at that level

---

## Common Use Cases

### Analytics Queries

**Digital download listings:**
```sql
SELECT
    listing_id,
    top_category,
    second_level_category
FROM `etsy-data-warehouse-prod.listing_mart.listing_attributes`
WHERE is_digital = 1
AND is_active = 1
```

**Personalizable jewelry:**
```sql
SELECT
    listing_id,
    second_level_category,
    recipient,
    color
FROM `etsy-data-warehouse-prod.listing_mart.listing_attributes`
WHERE is_personalizable = 1
AND top_category = 'Jewelry'
AND is_active = 1
```

**Listings by recipient:**
```sql
SELECT
    recipient,
    COUNT(*) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_attributes`
WHERE is_active = 1
AND recipient IS NOT NULL
GROUP BY recipient
ORDER BY listing_count DESC
```

**Color distribution in a category:**
```sql
SELECT
    color,
    COUNT(*) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_attributes`
WHERE second_level_category = 'Necklaces'
AND is_active = 1
AND color IS NOT NULL
GROUP BY color
ORDER BY listing_count DESC
LIMIT 20
```

**Taxonomy path search:**
```sql
SELECT
    listing_id,
    full_path
FROM `etsy-data-warehouse-prod.listing_mart.listing_attributes`
WHERE LOWER(full_path) LIKE '%vintage%'
AND is_active = 1
```

---

## Display Formatting Rules

### Boolean Flags
- `is_digital`, `is_personalizable`: 1 = Yes/True, 0 = No/False
- Display as badges: "Digital Download", "Personalizable"

### Text Fields
- `recipient`, `color` - Display as-is, title case preferred
- `top_category`, `second_level_category`, `third_level_category` - Display as breadcrumbs
- `full_path` - Can split on `.` for hierarchical display

### Categories
- Display as breadcrumb navigation: Top > Second > Third
- May truncate third_level_category if too long

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listing_attributes` for:**
- Filtering by recipient (Women, Men, Kids, etc.)
- Filtering by color
- Category/taxonomy analysis
- Finding digital or personalizable listings
- Full taxonomy path navigation

**Don't use this table for:**
- Comprehensive attribute search → Use `listing_mart.listing_all_attributes` (has ALL attributes)
- Variation-specific attributes → Use `listing_mart.listing_variation_attributes`
- Property values in general → This only has specific curated properties (recipient, color, digital, personalizable, taxonomy)

### Avoid These Joins

**❌ Don't join to these tables** (data already here or redundant):
- `etsy_shard.listings_properties_values` - Already extracted for recipient, is_digital, is_personalizable
- `etsy_shard.listing_attributes` - Color already extracted
- `materialized.listing_categories_taxonomy` - Taxonomy already denormalized here

**✅ Do join to these mart tables**:
- `listing_mart.listings` - For price, availability, shop info
- `listing_mart.listing_titles` - For title/description
- `listing_mart.listing_all_attributes` - For ALL other attributes not in this table

### Performance Tips

1. **Clustered by `listing_id`** - Include listing_id in WHERE when possible
2. **Taxonomy filtering** - Use `top_category`, `second_level_category` for broad filtering; `full_path LIKE '%term%'` for specific searches
3. **NULL handling** - `recipient` and `color` can be NULL; use `IS NOT NULL` if you need them populated
4. **Digital vs is_download** - `is_digital` here is from property_id=2; `is_download` in listings table is from seller_flags bit 6. They should match but prefer `is_download` from listings for consistency

### Common Query Patterns

**Filter by category and attributes:**
```sql
SELECT listing_id, recipient, color, second_level_category
FROM `etsy-data-warehouse-prod.listing_mart.listing_attributes`
WHERE is_active = 1
  AND top_category = 'Jewelry'
  AND recipient = 'Women'
  AND color IS NOT NULL
```

**Personalizable items by category:**
```sql
SELECT listing_id, second_level_category, full_path
FROM `etsy-data-warehouse-prod.listing_mart.listing_attributes`
WHERE is_active = 1
  AND is_personalizable = 1
  AND top_category = 'Home & Living'
```

**Taxonomy distribution:**
```sql
SELECT
  top_category,
  second_level_category,
  COUNT(*) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_attributes`
WHERE is_active = 1
GROUP BY top_category, second_level_category
ORDER BY listing_count DESC
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listings` - Core listing data (price, availability, shop info, status flags)
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_variations` - Variation summary
- `listing_mart.listing_variation_attributes` - Individual variation details
- `listing_mart.listing_products` - Product-level inventory and SKUs
- `listing_mart.listing_current_discounts` - Active promotions
- `listing_mart.listing_all_attributes` - Comprehensive attribute data (includes all attributes, not just curated ones)

### Reference Tables
- `listing_mart.all_properties` - Property ID to name lookup
- `listing_mart.all_property_values` - Value ID to value lookup
- `listing_mart.all_custom_property_names` - Custom variation names

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listings_properties_values` - Properties (recipient, is_digital, is_personalizable already extracted)
- `etsy_shard.listing_attributes` - Attributes (color already extracted)
- `materialized.listing_categories_taxonomy` - Taxonomy (already denormalized here)
