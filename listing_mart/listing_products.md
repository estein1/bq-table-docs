---
id: listing_products
title: etsy-data-warehouse-prod.listing_mart.listing_products
sidebar_label: listing_products
---

# listing_products

Product-level data for listings with inventory, including variations, quantities, and country-specific pricing. One row per product offering.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_products`

**Source Script:** `Rollups/auto/p2/daily/listing_mart.sql`

**Primary Key**: `product_id`

**Clustering**: Clustered by `listing_id`

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `shop_id` | INT64 | `etsy_shard.listing_products` | From listing_products (state!=0) | Shop identifier |
| `listing_id` | INT64 | `etsy_shard.listing_products` | From listing_products | Listing identifier |
| `product_id` | INT64 | `etsy_shard.listing_products` | From listing_products | Unique product identifier. Primary key |
| `product_offering_id` | INT64 | `etsy_shard.product_offerings` | From product_offerings where channel=1, state!=0. Most recent by update_date | Product offering identifier. From product_offerings for channel=1, most recent by update_date |
| `quantity_id` | INT64 | `etsy_shard.offering_quantity` | From offering_quantity (state!=0) linked via product_offering | Quantity identifier. Links to offering_quantity table |

### Status

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_active` | INT64 | `listing_mart.listings` | Direct from listings | Active listing indicator |
| `is_available` | INT64 | `etsy_shard.product_offerings` | Returns 1 if offering_state = 1, else 0 | Product is available. 1 if offering_state = 1 (active offering), else 0 |

### Inventory

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `quantity` | INT64 | `etsy_shard.offering_quantity` | From offering_quantity where state!=0 | Available quantity. May be shared across multiple products (same quantity_id) |

### Variations

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `variations` | ARRAY<STRUCT> | `etsy_shard.product_variations`, `listing_mart.listing_variation_attributes` | Aggregates variation attributes from product_variations join with listing_variation_attributes. Ordered by attribute_name. NULL if no variations | Array of variation attributes. **Structure**: `ARRAY<STRUCT<listing_variation_id INT64, attribute_name STRING, attribute_value STRING>>`. NULL if product has no variations |

### Pricing

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `csp_prices` | ARRAY<STRUCT> | `etsy_shard.product_offerings_prices`, `materialized.exchange_rates` | Aggregates country-specific prices with USD conversions. price_usd calculated as (market_rate/1e7) * price_int. Ordered by country_id. country_id=0 means all countries | Country-specific prices. **Structure**: `ARRAY<STRUCT<country_id INT64, currency_code STRING, market_rate INT64, price_int INT64, price_usd INT64>>`. All prices **STORED IN CENTS**. market_rate stored as rate*1e7 |

---

## Data Quality

### Assertions

- **Uniqueness**: Table enforces unique product_id

### Data Sources

- **Primary**: `etsy_shard.listing_products` (state != 0)
- **Product Offerings**: `etsy_shard.product_offerings` (channel=1, state != 0, latest by update_date)
- **Prices**: `etsy_shard.product_offerings_prices` (state != 0)
- **Quantities**: `etsy_shard.offering_quantity` (state != 0)
- **Variations**: Joined from `etsy_shard.product_variations` and `listing_mart.listing_variation_attributes`
- **Listings**: `listing_mart.listings`
- **Exchange Rates**: `materialized.exchange_rates`

---

## Business Logic

### Variations Array

- Each product can have 0-N variation attributes
- Example: A product might have `[{attribute_name: "Size", attribute_value: "Large"}, {attribute_name: "Color", attribute_value: "Red"}]`
- NULL if product has no variations
- Sorted by attribute_name

### CSP Prices Array

- CSP = Country-Specific Pricing
- Each product can have prices for multiple countries
- `country_id = 0` means "everywhere" (default price)
- Common country_id values: 209 (USA), 77 (UK), etc.
- All price_int and price_usd values are **in cents**

### Availability

- `is_available = 1` means the product offering is active (offering_state = 1)
- Products can exist even if not available (out of stock or inactive)

---

## Common Use Cases

### Analytics Queries

**Get products with variations for a listing:**
```sql
SELECT
    listing_id,
    product_id,
    variations,
    quantity,
    is_available
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`
WHERE listing_id = 12345
AND variations IS NOT NULL
```

**Extract specific variation attributes:**
```sql
SELECT
    listing_id,
    product_id,
    variation.attribute_name,
    variation.attribute_value
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`,
UNNEST(variations) AS variation
WHERE is_active = 1
```

**Get country-specific pricing:**
```sql
SELECT
    listing_id,
    product_id,
    price.country_id,
    price.currency_code,
    price.price_usd / 100.0 AS price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`,
UNNEST(csp_prices) AS price
WHERE listing_id = 12345
```

**Products available for USA:**
```sql
SELECT
    listing_id,
    product_id,
    price.price_usd / 100.0 AS usa_price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`,
UNNEST(csp_prices) AS price
WHERE price.country_id = 209  -- USA
AND is_available = 1
```

**In-stock products:**
```sql
SELECT
    listing_id,
    product_id,
    quantity
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`
WHERE is_available = 1
AND quantity > 0
```

---

## Display Formatting Rules

### Money Fields
- All prices in `csp_prices` array → **Divide by 100** for dollars
- `price_int` and `price_usd` are in cents
- Format with 2 decimal places: `$XX.XX`

### Exchange Rates
- `market_rate` in csp_prices → **Divide by 10,000,000** (1e7) for actual rate

### Quantities
- Display as whole numbers
- 0 or NULL = Out of stock

### Variations
- Flatten array for display
- Format: "Size: Large, Color: Red"
- NULL = No variations (base product)

### Arrays
- Use UNNEST to access array elements in SQL
- In application code, iterate through array structure

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listing_products` for:**
- Product-level inventory and SKUs
- Country-specific pricing (USA vs everywhere else)
- Individual product variation combinations (Size: Large + Color: Red)
- Available quantity per product
- **One row per product** (a listing can have multiple products)

**Don't use this table for:**
- Variation summary → Use `listing_mart.listing_variations`
- Variation properties only → Use `listing_mart.listing_variation_attributes`
- Listings without product variations → This table only includes listings with products

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_shard.listing_products` - Already the source
- `etsy_shard.product_offerings` - Already joined and filtered (channel=1)
- `etsy_shard.product_offerings_prices` - Already aggregated in `csp_prices` array
- `etsy_shard.offering_quantity` - Quantity already here

**✅ Do join to these mart tables**:
- `listing_mart.listings` - For base listing info, shop details
- `listing_mart.listing_titles` - For listing title
- `listing_mart.listing_variation_attributes` - For additional variation details

### Performance Tips

1. **Clustered by `listing_id`** - Always filter on listing_id
2. **variations is an ARRAY** - Use UNNEST to work with individual variation attributes
3. **csp_prices is an ARRAY** - Use UNNEST to filter by country or currency
4. **country_id = 0 means all countries** - Specific country IDs override the default
5. **Check `is_available = 1`** - Many products may be out of stock or discontinued

### Common Query Patterns

**Get products with variations for a listing:**
```sql
SELECT
  product_id,
  variations,
  quantity,
  is_available
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`
WHERE listing_id = 12345
  AND is_active = 1
ORDER BY product_id
```

**Unnest variation combinations:**
```sql
SELECT
  listing_id,
  product_id,
  v.attribute_name,
  v.attribute_value,
  quantity
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`,
  UNNEST(variations) AS v
WHERE listing_id = 12345
```

**Find USA-specific pricing:**
```sql
SELECT
  listing_id,
  product_id,
  price.country_id,
  price.price_usd / 100.0 AS price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`,
  UNNEST(csp_prices) AS price
WHERE listing_id = 12345
  AND price.country_id = 209  -- USA
```

**Products in stock with specific variation:**
```sql
SELECT
  lp.listing_id,
  lp.product_id,
  lp.quantity,
  v.attribute_name,
  v.attribute_value
FROM `etsy-data-warehouse-prod.listing_mart.listing_products` lp,
  UNNEST(lp.variations) AS v
WHERE lp.is_active = 1
  AND lp.is_available = 1
  AND lp.quantity > 0
  AND v.attribute_name = 'Size'
  AND v.attribute_value = 'Large'
```

**Compare default vs country-specific pricing:**
```sql
SELECT
  product_id,
  MAX(IF(price.country_id = 0, price.price_usd, NULL)) AS default_price_cents,
  MAX(IF(price.country_id = 209, price.price_usd, NULL)) AS usa_price_cents
FROM `etsy-data-warehouse-prod.listing_mart.listing_products`,
  UNNEST(csp_prices) AS price
WHERE listing_id = 12345
GROUP BY product_id
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listings` - Core listing data (price, availability, shop info, status flags)
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_attributes` - Attributes, taxonomy, recipient, color
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_variations` - Variation summary (one row per listing)
- `listing_mart.listing_variation_attributes` - Individual variation details (join on listing_variation_id for variation names/values)
- `listing_mart.listing_current_discounts` - Active promotions
- `listing_mart.listing_all_attributes` - Comprehensive attribute data

### Reference Tables
- `listing_mart.all_properties` - Property ID to name lookup
- `listing_mart.all_property_values` - Value ID to value lookup
- `listing_mart.all_custom_property_names` - Custom variation names

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listing_products` - Raw product data (already processed here)
- `etsy_shard.product_offerings` - Product offerings (already joined and filtered for channel=1)
- `etsy_shard.product_offerings_prices` - Product prices (already aggregated in csp_prices array)
- `etsy_shard.offering_quantity` - Quantities (quantity already included here)
- `etsy_shard.product_variations` - Product variation links (already used to build variations array)
- `materialized.exchange_rates` - Exchange rates (already converted in csp_prices.price_usd)
