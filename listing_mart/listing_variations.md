---
id: listing_variations
title: etsy-data-warehouse-prod.listing_mart.listing_variations
sidebar_label: listing_variations
---

# listing_variations

Summary of listing variations, one row per listing, containing variation counts, pricing ranges, and linking information.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_variations`

**Source Script:** `Rollups/auto/p2/daily/listing_mart.sql`

**Primary Key**: `listing_id`

**Clustering**: Clustered by `listing_id`

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `etsy_shard.listing_inventory_channel_summary` | From inventory summary (channel=1, state=1, latest by update_date) | Unique listing identifier. Primary key |
| `is_active` | INT64 | `listing_mart.listings` | Direct from listings | Active listing indicator |

### Variation Configuration

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `variation_count` | INT64 | `etsy_shard.listing_variations` | Count of distinct property_id (is_deleted!=1). Set to 0 if count is 1-3 but products_count=1. NULL becomes 0 | Number of variation types. Set to 0 if listing has 1-3 variation types but only 1 product |
| `quantity_linking_type` | INT64 | `etsy_shard.listing_inventory_channel_summary` | From inventory summary. NULL if variation_count = 0 | How quantity is linked across variations. NULL if variation_count = 0 |
| `price_linking_type` | INT64 | `etsy_shard.listing_inventory_channel_summary` | From inventory summary. NULL if variation_count = 0 | How price is linked across variations. NULL if variation_count = 0 |
| `sku_linking_type` | INT64 | `etsy_shard.listing_inventory_channel_summary` | From inventory summary. NULL if variation_count = 0 | How SKU is linked across variations. NULL if variation_count = 0 |
| `price_linking_id` | INT64 | `etsy_shard.listing_inventory_channel_summary` | Direct from inventory summary | Price linking identifier |
| `quantity_linking_id` | INT64 | `etsy_shard.listing_inventory_channel_summary` | Direct from inventory summary | Quantity linking identifier |
| `sku_linking_id` | INT64 | `etsy_shard.listing_inventory_channel_summary` | Direct from inventory summary | SKU linking identifier |
| `products_count` | INT64 | `etsy_shard.listing_inventory_channel_summary` | From inventory summary, defaults to 1 if NULL | Number of product offerings. Defaults to 1 if NULL |

### Pricing - Original Currency

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `currency_code` | STRING | `etsy_shard.listing_inventory_channel_summary` | Direct from inventory summary | ISO currency code |
| `min_price` | INT64 | `etsy_shard.listing_inventory_channel_summary` | Direct from inventory summary | Minimum price across variations. **STORED IN CENTS** |
| `max_price` | INT64 | `etsy_shard.listing_inventory_channel_summary` | Direct from inventory summary | Maximum price across variations. **STORED IN CENTS** |
| `market_rate` | INT64 | `materialized.exchange_rates` | Latest exchange rate before current_date for the currency | Exchange rate to USD. **Stored as rate*1e7**. Divide by 10,000,000 to get actual rate |

### Pricing - USD Converted

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `min_price_usd` | INT64 | Calculated from `min_price` and `market_rate` | Multiply min_price by (market_rate/1e7), cast to int64 | Minimum price in USD. **STORED IN CENTS**. Divide by 100 for dollars |
| `max_price_usd` | INT64 | Calculated from `max_price` and `market_rate` | Multiply max_price by (market_rate/1e7), cast to int64 | Maximum price in USD. **STORED IN CENTS**. Divide by 100 for dollars |

---

## Data Quality

### Data Sources

- **Primary**: `etsy_shard.listing_inventory_channel_summary` (channel = 1)
- **Variation Count**: `etsy_shard.listing_variations` (non-deleted)
- **Listings**: `listing_mart.listings`
- **Exchange Rates**: `materialized.exchange_rates`

---

## Business Logic

### Variation Count Logic

```
IF listing has 1-3 variation types AND only 1 product THEN
    variation_count = 0  (not considered variations)
ELSE IF listing has variations THEN
    variation_count = COUNT(DISTINCT property_id)
ELSE
    variation_count = 0
END IF
```

**Example**: A listing with Size and Color variations, but only one combination available (e.g., "Small/Red") would have variation_count = 0.

### Products Count

- Sourced from `listing_inventory_channel_summary.products_count`
- Defaults to 1 if NULL (no inventory data)

### Linking Types

When variation_count > 0, these fields indicate how properties are linked:
- `quantity_linking_type` - How inventory is shared across variations
- `price_linking_type` - How pricing differs across variations
- `sku_linking_type` - How SKUs are assigned to variations

Set to NULL when variation_count = 0.

---

## Common Use Cases

### Analytics Queries

**Listings with variations:**
```sql
SELECT
    listing_id,
    variation_count,
    products_count,
    min_price_usd / 100.0 AS min_price_dollars,
    max_price_usd / 100.0 AS max_price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listing_variations`
WHERE variation_count > 0
AND is_active = 1
```

**Price range analysis:**
```sql
SELECT
    listing_id,
    (max_price_usd - min_price_usd) / 100.0 AS price_range_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listing_variations`
WHERE max_price_usd > min_price_usd
ORDER BY price_range_dollars DESC
LIMIT 100
```

**Listings with many product variations:**
```sql
SELECT
    listing_id,
    products_count,
    variation_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_variations`
WHERE products_count > 10
AND is_active = 1
ORDER BY products_count DESC
```

---

## Display Formatting Rules

### Money Fields
- `min_price`, `max_price`, `min_price_usd`, `max_price_usd` → **Divide by 100** for dollars
- Format with 2 decimal places: `$XX,XXX.XX`
- Display as range: `$X.XX - $Y.YY`

### Exchange Rates
- `market_rate` → **Divide by 10,000,000** (1e7) to get actual rate

### Counts
- `variation_count`, `products_count` - Display as whole numbers
- 0 variations = "No variations" or "Single variant"

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listing_variations` for:**
- Getting variation summary (count, price range) per listing
- Analyzing price ranges across variations
- Understanding variation configuration (linking types)
- Counting products per listing
- **One row per listing** summary queries

**Don't use this table for:**
- Individual variation details (Size: Large, Color: Red) → Use `listing_mart.listing_variation_attributes`
- Product-level data with SKUs → Use `listing_mart.listing_products`
- Listings without variations → This table only includes listings with inventory data

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_shard.listing_inventory_channel_summary` - This IS that data (channel=1, deduped)
- `etsy_shard.listing_variations` - Variation count already calculated from this

**✅ Do join to these mart tables**:
- `listing_mart.listings` - For shop info, status, base listing details
- `listing_mart.listing_variation_attributes` - For individual variation property values
- `listing_mart.listing_products` - For product-level details

### Performance Tips

1. **Clustered by `listing_id`** - Filter on listing_id for best performance
2. **variation_count logic is nuanced** - 0 can mean "no variations" OR "has variation properties but only 1 product". Check `products_count` to distinguish
3. **Price range filtering** - Use `min_price_usd` and `max_price_usd` for USD filtering (values in cents)
4. **Not all listings are here** - Only listings with inventory data. Join to `listing_mart.listings` to include all listings

### Common Query Patterns

**Listings with variations and price ranges:**
```sql
SELECT
  listing_id,
  variation_count,
  products_count,
  min_price_usd / 100.0 AS min_price_dollars,
  max_price_usd / 100.0 AS max_price_dollars,
  (max_price_usd - min_price_usd) / 100.0 AS price_range_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listing_variations`
WHERE is_active = 1
  AND variation_count > 0
ORDER BY price_range_dollars DESC
```

**Listings with many products:**
```sql
SELECT listing_id, variation_count, products_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_variations`
WHERE is_active = 1
  AND products_count > 10
ORDER BY products_count DESC
```

**Price range in specific currency:**
```sql
SELECT
  listing_id,
  currency_code,
  min_price / 100.0 AS min_price_original,
  max_price / 100.0 AS max_price_original
FROM `etsy-data-warehouse-prod.listing_mart.listing_variations`
WHERE is_active = 1
  AND currency_code = 'EUR'
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listings` - Core listing data (price, availability, shop info, status flags)
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_attributes` - Attributes, taxonomy, recipient, color
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_variation_attributes` - Individual variation details (multiple rows per listing)
- `listing_mart.listing_products` - Product-level inventory, SKUs, country-specific pricing
- `listing_mart.listing_current_discounts` - Active promotions
- `listing_mart.listing_all_attributes` - Comprehensive attribute data

### Reference Tables
- `listing_mart.all_properties` - Property ID to name lookup
- `listing_mart.all_property_values` - Value ID to value lookup
- `listing_mart.all_custom_property_names` - Custom variation names

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listing_inventory_channel_summary` - Inventory data (this table IS that data, deduped for channel=1)
- `etsy_shard.listing_variations` - Variation properties (variation_count already calculated from this)
- `materialized.exchange_rates` - Exchange rates (already converted in *_usd fields)
