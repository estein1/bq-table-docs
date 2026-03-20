---
id: listing_variation_attributes
title: etsy-data-warehouse-prod.listing_mart.listing_variation_attributes
sidebar_label: listing_variation_attributes
---

# listing_variation_attributes

Detailed variation attributes for each listing variation, one row per listing_variation_id. Includes property names, values, and pricing for each specific variation option.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes`

**Source Script:** `Rollups/auto/p2/daily/listing_mart.sql`

**Primary Key**: Composite key (`listing_id`, `listing_variation_id`)

**Clustering**: Clustered by `listing_id`

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `etsy_shard.listing_variations` | From listing_variations (is_deleted!=1) | Listing identifier. Part of composite primary key |
| `listing_variation_id` | INT64 | `etsy_shard.listing_variations` | From listing_variations | Variation option identifier. Part of composite primary key. Each variation option gets unique ID |
| `is_active` | INT64 | `listing_mart.listings` | Direct from listings | Active listing indicator |
| `update_date` | INT64 | `etsy_shard.listing_variations` | From listing_variations | Last update timestamp. **Unix timestamp (seconds)** |

### Attribute Details

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `attribute_name` | STRING | `listing_mart.all_custom_property_names`, `listing_mart.all_properties` | Prefers custom_name if seller defined one, otherwise uses standard attribute_name from all_properties | Property name (e.g., "Size", "Color"). Custom name if seller defined one, otherwise standard property name |
| `attribute_value` | STRING | `listing_mart.all_property_values` | Joined on value_id from all_property_values | Property value (e.g., "Large", "Red"). HTML entities already decoded |
| `ott_attribute_id` | INT64 | `listing_mart.all_properties` | From all_properties.ott_attribute_id | OTT attribute identifier. NULL if not mapped to OTT taxonomy |
| `ott_value_id` | INT64 | `listing_mart.all_property_values` | From all_property_values.ott_value_id | OTT value identifier. NULL if not mapped to OTT taxonomy |
| `scale_name` | STRING | `listing_mart.all_property_values` | From all_property_values.scale_name | Measurement scale if applicable. E.g., "Inches", "Centimeters". NULL if no scale applies |

### Availability

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_available` | INT64 | Product data OR `etsy_shard.listing_variations` | Prefers product-level availability if products exist, otherwise from listing_variations | Variation option is available. 1 if available for purchase, 0 if out of stock or unavailable |

### Pricing - Original Currency

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `currency_code` | STRING | Product data OR variation data | Prefers currency from product offerings, falls back to listing variation currency | ISO currency code |
| `min_price` | INT64 | `etsy_shard.product_offerings_prices` OR calculated from `listing_variations.price_diff` | Prefers product prices if available, otherwise calculates from listing variation price_diff + base price | Minimum price for this variation. **STORED IN CENTS**. Sourced from product prices if available, otherwise from listing variation price_diff |
| `max_price` | INT64 | Product prices OR listing variation prices | Same preference logic as min_price | Maximum price for this variation. **STORED IN CENTS** |
| `min_price_avail` | INT64 | Product prices OR listing variation prices | Same logic, filtered to available items only | Min price for available items only. **STORED IN CENTS**. NULL if nothing available |
| `max_price_avail` | INT64 | Product prices OR listing variation prices | Same logic, filtered to available items only | Max price for available items only. **STORED IN CENTS**. NULL if nothing available |

### Pricing - USD Converted

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `min_price_usd` | INT64 | Calculated from `min_price` and `market_rate` | Multiply min_price by (market_rate/1e7), cast to int64 | Minimum price in USD. **STORED IN CENTS**. Divide by 100 for dollars |
| `max_price_usd` | INT64 | Calculated from `max_price` and `market_rate` | Multiply max_price by (market_rate/1e7), cast to int64 | Maximum price in USD. **STORED IN CENTS**. Divide by 100 for dollars |
| `min_price_avail_usd` | INT64 | Calculated from `min_price_avail` and `market_rate` | Multiply min_price_avail by (market_rate/1e7), cast to int64 | Min USD price for available items. **STORED IN CENTS**. NULL if nothing available |
| `max_price_avail_usd` | INT64 | Calculated from `max_price_avail` and `market_rate` | Multiply max_price_avail by (market_rate/1e7), cast to int64 | Max USD price for available items. **STORED IN CENTS**. NULL if nothing available |

---

## Data Quality

### Data Sources

- **Primary**: `etsy_shard.listing_variations` (non-deleted)
- **Product Prices**: Calculated from `product_offerings_prices` via product_variations join
- **Listing Variation Prices**: Calculated from `listing_variations.price_diff` for listings without products
- **Attribute Names**: `listing_mart.all_properties` and `listing_mart.all_custom_property_names`
- **Attribute Values**: `listing_mart.all_property_values`

---

## Business Logic

### Price Calculation

1. **For listings with products**:
   - Prices come from actual product offerings
   - min/max calculated from all products associated with this variation

2. **For listings without products**:
   - Prices calculated from base listing price + variation price_diff
   - Considers all possible combinations of variations

### Attribute Name Priority

1. Custom name (if seller defined one in `all_custom_property_names`)
2. Standard property name from `all_properties`

### Available Pricing

- `*_avail` price fields only include variations where `is_available = 1`
- Used to show accurate pricing for in-stock items

---

## Common Use Cases

### Analytics Queries

**Get all variations for a listing:**
```sql
SELECT
    listing_id,
    attribute_name,
    attribute_value,
    min_price_usd / 100.0 AS price_dollars,
    is_available
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes`
WHERE listing_id = 12345
ORDER BY attribute_name, attribute_value
```

**Most common variation attributes:**
```sql
SELECT
    attribute_name,
    COUNT(DISTINCT listing_id) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes`
WHERE is_active = 1
GROUP BY attribute_name
ORDER BY listing_count DESC
LIMIT 20
```

**Size variations with prices:**
```sql
SELECT
    listing_id,
    attribute_value AS size,
    min_price_avail_usd / 100.0 AS available_price_dollars,
    is_available
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes`
WHERE attribute_name = 'Size'
AND is_active = 1
ORDER BY listing_id, attribute_value
```

**Out of stock variations:**
```sql
SELECT
    listing_id,
    attribute_name,
    attribute_value
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes`
WHERE is_available = 0
AND is_active = 1
```

---

## Display Formatting Rules

### Money Fields
- `min_price`, `max_price`, `*_avail`, `*_usd` → **Divide by 100** for dollars
- Format with 2 decimal places: `$XX.XX`
- Show as range if min != max: `$X.XX - $Y.YY`

### Availability
- `is_available`: 1 = In Stock, 0 = Out of Stock
- Display with badge or icon

### Attributes
- `attribute_name` - Display in title case
- `attribute_value` - Display as-is
- Combine: "Size: Large" or "Color: Red"

### Dates
- `update_date` - **Unix timestamp in seconds**
- Convert using `TIMESTAMP_SECONDS()` for display

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listing_variation_attributes` for:**
- Getting individual variation options (Size: Large, Color: Red)
- Filtering specific variation values
- Analyzing which variation options are available/in stock
- Price per variation option
- **Multiple rows per listing** (one per variation option)

**Don't use this table for:**
- Variation summary per listing → Use `listing_mart.listing_variations` (one row per listing)
- Product-level data → Use `listing_mart.listing_products`
- All listing attributes → Use `listing_mart.listing_all_attributes`

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_shard.listing_variations` - Already used as source
- `listing_mart.all_properties` - Attribute names already joined
- `listing_mart.all_property_values` - Values already joined
- `listing_mart.all_custom_property_names` - Custom names already preferred

**✅ Do join to these mart tables**:
- `listing_mart.listings` - For base listing info
- `listing_mart.listing_titles` - For listing title
- `listing_mart.listing_variation_attributes` to itself - To combine multiple variation properties for the same listing

### Performance Tips

1. **Clustered by `listing_id`** - Always filter on listing_id first
2. **Multiple rows per listing** - A listing with Size (S, M, L) and Color (Red, Blue) will have 2 rows (one for each property type), not 6
3. **Use `is_available = 1`** - Filter to in-stock variations
4. **Custom vs standard names** - `attribute_name` already handles custom seller names; no need to check separately
5. **Scale names** - Only populated for measurements; check `IS NOT NULL` if you need units

### Common Query Patterns

**Get all variation options for a listing:**
```sql
SELECT
  listing_id,
  attribute_name,
  attribute_value,
  is_available,
  min_price_usd / 100.0 AS min_price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes`
WHERE listing_id = 12345
ORDER BY attribute_name, attribute_value
```

**Find listings with specific variation values:**
```sql
SELECT DISTINCT listing_id
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes`
WHERE is_active = 1
  AND is_available = 1
  AND attribute_name = 'Size'
  AND attribute_value = 'Large'
```

**Pivot variations (Size + Color for same listing):**
```sql
SELECT
  listing_id,
  MAX(IF(attribute_name = 'Size', attribute_value, NULL)) AS size,
  MAX(IF(attribute_name = 'Color', attribute_value, NULL)) AS color
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes`
WHERE listing_id IN (12345, 67890)
GROUP BY listing_id
```

**Available variation values in a category:**
```sql
SELECT
  lva.attribute_name,
  lva.attribute_value,
  COUNT(DISTINCT lva.listing_id) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes` lva
JOIN `etsy-data-warehouse-prod.listing_mart.listing_attributes` la USING (listing_id)
WHERE lva.is_active = 1
  AND lva.is_available = 1
  AND la.top_category = 'Jewelry'
GROUP BY lva.attribute_name, lva.attribute_value
ORDER BY listing_count DESC
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listings` - Core listing data (price, availability, shop info, status flags)
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_attributes` - Attributes, taxonomy, recipient, color
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_variations` - Variation summary (one row per listing)
- `listing_mart.listing_products` - Product-level inventory, SKUs (join on listing_id + variations)
- `listing_mart.listing_current_discounts` - Active promotions
- `listing_mart.listing_all_attributes` - Comprehensive attribute data

### Reference Tables (Used to Build This Table)
- `listing_mart.all_properties` - Property ID to name lookup (attribute_name sourced from here)
- `listing_mart.all_property_values` - Value ID to value lookup (attribute_value sourced from here)
- `listing_mart.all_custom_property_names` - Custom variation names (used in attribute_name)

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listing_variations` - Raw variation data (already processed here)
- `etsy_shard.product_offerings_prices` - Product prices (already calculated here if products exist)
- `materialized.exchange_rates` - Exchange rates (already converted in *_usd fields)
