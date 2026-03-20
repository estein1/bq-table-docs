---
id: listings
title: etsy-data-warehouse-prod.listing_mart.listings
sidebar_label: listings
---

# listings

Core listing mart table containing current state of all Etsy listings with pricing, availability, and shop information.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listings`

**Source Script:** `Rollups/auto/p2/daily/listing_mart.sql`

**Purpose**: Core listing mart table containing current state of all Etsy listings with pricing, availability, and shop information.

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
| `listing_id` | INT64 | `etsy_shard.listings` | `l.listing_id` | Unique listing identifier. Primary key; duplicates are filtered out during creation |
| `shop_id` | INT64 | `etsy_shard.listings` | `l.shop_id` | Shop identifier. References shop_data.shop_id |
| `shop_name` | STRING | `etsy_shard.shop_data` | `s.name` | Shop name |
| `user_id` | INT64 | `etsy_shard.listings` | `l.user_id` | Seller user ID |

### Status & Availability

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_active` | INT64 | Derived from multiple fields in `etsy_shard.listings` | Returns 1 if ALL conditions met: admin_flags=0, state=0, is_available=1, is_displayable=1, is_searchable=1. Otherwise 0 | Active listing indicator. Must meet ALL conditions: admin_flags=0, state=0, is_available=1, is_displayable=1, is_searchable=1 |
| `state` | INT64 | `etsy_shard.listings` | Direct from listings table | Listing state code. 0 = active state |
| `is_available` | INT64 | `etsy_shard.listings` | Direct from listings table, cast to int64 | Listing is available for purchase |
| `is_displayable` | INT64 | `etsy_shard.listings` | Direct from listings table, cast to int64 | Listing can be displayed |
| `is_searchable` | INT64 | `etsy_shard.listings` | Direct from listings table, cast to int64 | Listing appears in search |
| `seller_flags` | INT64 | `etsy_shard.listings` | Direct from listings table | Bitwise seller flags. See Seller Flags section below |
| `admin_flags` | INT64 | `etsy_shard.listings` | Direct from listings table | Bitwise admin flags |
| `featured_rank` | INT64 | `etsy_shard.listings` | Direct from listings table | Featured ranking |
| `quantity` | INT64 | `etsy_shard.listings` | Direct from listings table | Available quantity |

### Pricing - Original Currency

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `price` | INT64 | `etsy_shard.listing_inventory_channel_summary` (primary), `etsy_shard.listings` (fallback) | Prefers min_price from inventory summary (channel=1, state=1, latest update_date), falls back to listings.price | Listing price in original currency. **STORED IN CENTS**. Prefers inventory summary data, falls back to listings table if not available |
| `currency_code` | STRING | `etsy_shard.listing_inventory_channel_summary` (primary), `etsy_shard.listings` (fallback) | Prefers currency_code from inventory summary, falls back to listings.currency_code | ISO currency code (e.g., 'USD', 'EUR') |
| `market_rate` | INT64 | `materialized.exchange_rates` | Latest exchange rate before current_date for the currency, defaults to 1 if not found | Exchange rate to USD. **Stored as rate*1e7**. Divide by 10,000,000 to get actual rate. Defaults to 1 for missing rates |

### Pricing - USD Converted

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `price_usd` | INT64 | Calculated from `market_rate` and `price` | Multiply price by (market_rate/1e7), cast to int64. Uses market_rate from exchange_rates and price from inventory summary or listings | Price converted to USD. **STORED IN CENTS**. Calculation: (market_rate/1e7) * price. For display, divide by 100 to get dollars |

### Pricing - USA Specific

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `price_usa` | INT64 | `etsy_shard.listing_inventory_channel_summary_prices` | From min_price where channel=1, state=1, country_id=209, created >= 2025-10-20 | USA-specific price if set. **STORED IN CENTS**. NULL if listing doesn't have USA-specific pricing. Only includes prices created since October 20, 2025 |
| `price_usa_usd` | INT64 | Calculated from `price_usa` and `market_rate_usa` | Multiply price_usa by (market_rate_usa/1e7), cast to int64 | USA price converted to USD. **STORED IN CENTS**. For display, divide by 100 |
| `currency_code_usa` | STRING | `etsy_shard.listing_inventory_channel_summary_prices` | Currency code from USA-specific pricing record | Currency code for USA pricing |
| `market_rate_usa` | INT64 | `materialized.exchange_rates` | Returns 1 if currency is USD, otherwise latest exchange rate before current_date | Exchange rate for USA pricing. Returns 1 if currency_code_usa='USD', otherwise uses rate from exchange_rates |

### Dates & Timing

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `original_create_date` | INT64 | `etsy_shard.listings` | Direct from listings table | Original listing creation timestamp. **Unix timestamp (seconds)** |
| `create_date` | INT64 | `etsy_shard.listings` | Direct from listings table | Most recent listing creation timestamp. **Unix timestamp (seconds)**. May differ from original_create_date if listing was renewed |
| `ending_date` | INT64 | `etsy_shard.listings` | Direct from listings table | Listing expiration timestamp. **Unix timestamp (seconds)** |
| `update_date` | INT64 | `etsy_shard.listings` | Direct from listings table | Last update timestamp. **Unix timestamp (seconds)** |
| `days_since_original_create` | INT64 | Calculated from `original_create_date` | Date difference between current_date and original_create_date plus 1 | Days since original creation. Current date minus original_create_date + 1 |
| `days_since_create` | INT64 | Calculated from `create_date` | Date difference between current_date and create_date plus 1 | Days since most recent creation. Current date minus create_date + 1 |
| `new_listing_flag` | INT64 | Derived from `original_create_date` | Returns 1 if original_create_date is yesterday or later, else 0 | New listing indicator. 1 if original_create_date is yesterday or later, else 0 |

### Images & Media

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `image_key` | STRING | `etsy_shard.listings` | `image_key` | Primary image identifier. Key for fetching listing image |

### Listing Type Indicators

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_vacation` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 0 | Shop on vacation. Extracted from seller_flags bit 0 |
| `is_closed` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 1 | Shop is closed. Extracted from seller_flags bit 1 |
| `is_reserved` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 2 | Listing is reserved. Extracted from seller_flags bit 2 |
| `is_wholesale` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 3 | Wholesale listing. Extracted from seller_flags bit 3 |
| `is_notretail` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 4 | Not retail listing. Extracted from seller_flags bit 4 |
| `is_fundonetsy` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 5 | Fund on Etsy listing. Extracted from seller_flags bit 5 |
| `is_download` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 6 | Digital download. Extracted from seller_flags bit 6 |
| `is_subscription` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 7 | Subscription listing. Extracted from seller_flags bit 7 |
| `is_customshop` | INT64 | `etsy_shard.listings` | Bitwise check of seller_flags bit 8 | Custom shop listing. Extracted from seller_flags bit 8 |
| `is_mature` | INT64 | `etsy_shard.suppression_listing_restrictions` | Returns 1 if listing has restriction='mature', is_deleted=0, max(is_restricted)=1, and no admin un-restrictions | Mature content listing. 1 if listing in suppression_listing_restrictions with restriction='mature' and is_restricted=1 with no admin overrides |
| `has_inventory_data` | INT64 | `etsy_shard.listing_inventory_channel_summary` | Returns 1 if listing exists in inventory summary (channel=1, state=1, latest by update_date) | Has inventory summary data. 1 if listing exists in listing_inventory_channel_summary |
| `has_products` | INT64 | `etsy_shard.listing_inventory_channel_summary` | Returns 1 if products_count > 0 from inventory summary, defaults to 0 | Has product variations. 1 if products_count > 0 from inventory summary |
| `is_gift_card` | INT64 | `transaction_mart.receipts_gms` | Returns 1 if seller has any receipts with is_gift_card=1 | Is a gift card listing. 1 if seller has sold gift cards |

---

## Data Quality

### Assertions

- **Uniqueness**: Table enforces unique listing_id (duplicates from source are filtered)
- **Date Filter**: Only includes listings where original_create_date < current_date

### Data Sources

- **Primary**: `etsy_shard.listings`
- **Pricing**: `etsy_shard.listing_inventory_channel_summary` (preferred), falls back to listings.price
- **USA Pricing**: `etsy_shard.listing_inventory_channel_summary_prices` (channel=1, country_id=209, created since 2025-10-20)
- **Exchange Rates**: `materialized.exchange_rates` (latest rate before current_date)
- **Shop Info**: `etsy_shard.shop_data`
- **Mature Listings**: `etsy_shard.suppression_listing_restrictions`
- **Gift Cards**: `transaction_mart.receipts_gms`

---

## Common Use Cases

### Analytics Queries

**Active listings with price in dollars:**
```sql
SELECT
    listing_id,
    shop_name,
    price_usd / 100.0 AS price_dollars,
    currency_code
FROM `etsy-data-warehouse-prod.listing_mart.listings`
WHERE is_active = 1
```

**New listings created yesterday:**
```sql
SELECT *
FROM `etsy-data-warehouse-prod.listing_mart.listings`
WHERE new_listing_flag = 1
```

**Digital download listings:**
```sql
SELECT *
FROM `etsy-data-warehouse-prod.listing_mart.listings`
WHERE is_download = 1
AND is_active = 1
```

---

## Display Formatting Rules

### Money Fields
- `price`, `price_usd`, `price_usa`, `price_usa_usd` → **Divide by 100** to display as dollars
- Format with 2 decimal places: `$XX,XXX.XX`

### Exchange Rates
- `market_rate`, `market_rate_usa` → **Divide by 10,000,000** (1e7) to get actual rate
- Display with 4-6 decimal places

### Dates
- All date fields are **Unix timestamps in seconds**
- Convert using `TIMESTAMP_SECONDS()` in BigQuery or equivalent in your tool
- Display in ISO 8601 format or user's preferred timezone

### Boolean Flags
- All `is_*` fields: 1 = Yes/True, 0 = No/False
- Display as checkmarks (✓/✗) or Yes/No

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listings` for:**
- Getting core listing data (prices, status, shop info) in a single query
- Filtering active/available listings
- Price analysis (original currency or USD)
- Seller flag analysis (vacation, wholesale, digital downloads, etc.)
- Gift card seller identification
- Any query where you need listing-level aggregates

**Don't use this table for:**
- Listing titles/descriptions → Use `listing_mart.listing_titles`
- Detailed variation data → Use `listing_mart.listing_variations` or `listing_mart.listing_variation_attributes`
- Individual product SKUs/inventory → Use `listing_mart.listing_products`
- Taxonomy/category details → Use `listing_mart.listing_attributes`
- Image URLs → Use `listing_mart.listing_images`

### Avoid These Joins

**❌ Don't join to these tables** (data already denormalized here):
- `etsy_shard.listings` - This mart IS the listings table with enhancements
- `etsy_shard.shop_data` - Shop name already included via `shop_name` column
- `etsy_shard.listing_inventory_channel_summary` - Price already sourced from here
- `materialized.exchange_rates` - Already converted to USD in `price_usd` columns

**✅ Do join to these mart tables** for additional details:
- `listing_mart.listing_titles` - For title and description
- `listing_mart.listing_attributes` - For recipient, color, taxonomy, digital/personalizable flags
- `listing_mart.listing_variations` - For variation summary
- `listing_mart.listing_images` - For image URLs

### Performance Tips

1. **Always filter on `listing_id` when possible** - Table is clustered by listing_id for optimal performance
2. **Use `is_active = 1` early in WHERE clause** - Dramatically reduces row count (most listings are inactive)
3. **Price fields are already in cents** - Divide by 100 for display, but keep as INT64 for filtering to avoid float precision issues
4. **Bitwise flag extraction is already done** - Use `is_vacation`, `is_download`, etc. instead of bitwise operations on `seller_flags`
5. **For large scans, use `new_listing_flag = 1`** - Much smaller subset for recent listing analysis

### Common Query Patterns

**Get active listings with converted prices:**
```sql
SELECT listing_id, shop_name, price_usd / 100.0 AS price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listings`
WHERE is_active = 1
  AND listing_id IN (12345, 67890)  -- Use listing_id for clustering
```

**Digital downloads in a price range:**
```sql
SELECT listing_id, price_usd / 100.0 AS price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listings`
WHERE is_active = 1
  AND is_download = 1
  AND price_usd BETWEEN 500 AND 2000  -- $5 to $20
```

**Shop-level aggregation:**
```sql
SELECT
  shop_id,
  shop_name,
  COUNT(*) AS listing_count,
  AVG(price_usd) / 100.0 AS avg_price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listings`
WHERE is_active = 1
GROUP BY shop_id, shop_name
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_attributes` - Attributes, taxonomy, recipient, color, digital/personalizable flags
- `listing_mart.listing_images` - Image URLs (primary and up to 20 additional)
- `listing_mart.listing_variations` - Variation summary (one row per listing)
- `listing_mart.listing_variation_attributes` - Individual variation details (multiple rows per listing)
- `listing_mart.listing_products` - Product-level inventory, SKUs, country-specific pricing
- `listing_mart.listing_current_discounts` - Active promotions and sales
- `listing_mart.listing_all_attributes` - Comprehensive attribute data (all properties)

### Reference Tables
- `listing_mart.all_properties` - Property ID to attribute name lookup
- `listing_mart.all_property_values` - Value ID to attribute value lookup
- `listing_mart.all_custom_property_names` - Custom variation names per listing

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listings` - Raw listings table (use this mart instead)
- `etsy_shard.shop_data` - Shop information (shop_name already included here)
- `etsy_shard.listing_inventory_channel_summary` - Inventory data (price already sourced from here)
- `materialized.exchange_rates` - Exchange rates (already converted in price_usd fields)
- `transaction_mart.receipts_gms` - Used for is_gift_card flag (already derived here)
