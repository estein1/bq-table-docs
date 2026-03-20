---
id: listing_mart_overview
title: listing_mart Schema Overview
sidebar_label: Overview
---

# listing_mart Schema Overview

## Purpose

The `listing_mart` schema contains denormalized, analytics-ready tables for Etsy listing data. These tables are designed for efficient querying and analysis, with pricing, variations, attributes, and images all readily available.

**Schema**: `etsy-data-warehouse-prod.listing_mart`

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

**Update Frequency**: Daily (p2 rollup)

**Source Scripts**:
- `Rollups/auto/p2/daily/listing_mart.sql` - Core tables
- `Rollups/auto/p2/daily/listing_indicators.sql` - Indicators (POD, how-it's-made, etc.)
- `Rollups/auto/p2/daily/listing_mart_2.sql` - Ngrams and queries
- `Rollups/auto/p2/daily/listing_mart_bucket3.sql` - GMS and post-purchase
- `Rollups/auto/p2/daily/listing_mart_post_analytics.sql` - Counts (depends on analytics)
- `Rollups/auto/p2/daily/listing_mart_mvs.sql` - Materialized views

---

## Table Index

### Core Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [listings](listings.md) | Core listing data with pricing and status | `listing_id` | Main listing info, prices, flags |
| [listing_titles](listing_titles.md) | Titles and descriptions with translation fallback | `listing_id` | Display text, search |
| [listing_attributes](listing_attributes.md) | Key attributes (color, recipient, digital, taxonomy) | `listing_id` | Filtering, categorization |
| [listing_images](listing_images.md) | Image URLs (up to 20 per listing) | `listing_id` | Display images |

### Variation & Product Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [listing_variations](listing_variations.md) | Variation summary per listing | `listing_id` | Variation counts, price ranges |
| [listing_variation_attributes](listing_variation_attributes.md) | Detailed variation attributes | `listing_id`, `listing_variation_id` | Specific variation options (Size: Large, Color: Red) |
| [listing_products](listing_products.md) | Product-level data with inventory | `product_id` | Inventory, country-specific pricing |

### Attribute Reference Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [all_properties](all_properties.md) | Property ID to name mapping | `property_id` | Decode property IDs to names |
| [all_property_values](all_property_values.md) | Value ID to name mapping | `value_id` | Decode value IDs to readable values |
| [all_custom_property_names](all_custom_property_names.md) | Custom variation names by listing | `listing_id`, `property_id` | Seller-defined variation labels |

### Comprehensive Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [listing_all_attributes](listing_all_attributes.md) | ALL listing attributes (many rows per listing) | `listing_id`, `attribute_name`, `attribute_value` | Complete attribute analysis |
| [listing_current_discounts](listing_current_discounts.md) | Active sales/discounts | `listing_id` | Current promotions |

### Indicators & Classification

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [listing_indicators](listing_indicators.md) | POD classification, how-it's-made, flagship segments, personalization | `listing_id` | POD detection, seller creation method, segments |

### Text Analysis & Search

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [listing_ngrams](listing_ngrams.md) | Tokens, bigrams, trigrams from titles/tags/styles (with duplicates) | None | NLP, text analysis, word frequency |
| [listing_ngrams_combined](listing_ngrams_combined.md) | Deduplicated ngrams per listing | `listing_id`, `ngram` | Simpler ngram queries |
| [listing_all_queries](listing_all_queries.md) | Search queries that drove clicks/carts/purchases | `listing_id`, `query` | Search analysis, SEO |

### Performance & Metrics

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [listing_gms](listing_gms.md) | GMS, sales, orders (total, past day, past year, normalized) | `listing_id` | Revenue analysis, sales velocity |
| [listing_post_purch](listing_post_purch.md) | Returns, refunds, ratings, cases | `listing_id` | Quality, customer satisfaction |
| [listing_counts](listing_counts.md) | Counts of attributes, tags, materials, views, favorites, images | `listing_id` | Metadata completeness, engagement |

### Materialized Views (Active Listings Only)

**📂 [Materialized Views](materialized_views/README.md)** - 11 `_active` views for faster queries on active listings

---

## Common Query Patterns

### Simple Listing Query

For basic listing information, use the core tables:

```sql
SELECT
    l.listing_id,
    lt.title,
    l.price_usd / 100.0 AS price_dollars,
    la.top_category,
    la.second_level_category,
    li.listing_image_url
FROM `etsy-data-warehouse-prod.listing_mart.listings` l
JOIN `etsy-data-warehouse-prod.listing_mart.listing_titles` lt USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_attributes` la USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_images` li USING (listing_id)
WHERE l.is_active = 1
LIMIT 100
```

### Listings with Variations

```sql
SELECT
    l.listing_id,
    lt.title,
    lv.variation_count,
    lv.min_price_usd / 100.0 AS min_price_dollars,
    lv.max_price_usd / 100.0 AS max_price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listings` l
JOIN `etsy-data-warehouse-prod.listing_mart.listing_titles` lt USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_variations` lv USING (listing_id)
WHERE l.is_active = 1
AND lv.variation_count > 0
```

### Specific Variation Details

```sql
SELECT
    lva.listing_id,
    lva.attribute_name,
    lva.attribute_value,
    lva.min_price_avail_usd / 100.0 AS available_price_dollars,
    lva.is_available
FROM `etsy-data-warehouse-prod.listing_mart.listing_variation_attributes` lva
WHERE lva.listing_id = 12345
ORDER BY lva.attribute_name, lva.attribute_value
```

---

## Important Formatting Rules

### Money Fields

**Price fields in listing tables are stored in CENTS** - divide by 100 for display:
- `price`, `price_usd`, `price_usa`, `price_usa_usd` (listings)
- `min_price*`, `max_price*` (listing_variations, listing_variation_attributes)
- All prices in `csp_prices` array (listing_products)

```sql
-- CORRECT
price_usd / 100.0 AS price_dollars

-- WRONG
price_usd AS price_dollars  -- This shows cents, not dollars!
```

**GMS/revenue fields are already in USD DOLLARS** - no conversion needed:
- All `*_gms` fields in `listing_gms` (total_gms, past_year_gms, etc.)
- `transaction_refund_amount` in `listing_post_purch`

```sql
-- CORRECT - already in dollars
SELECT total_gms FROM listing_mart.listing_gms

-- WRONG - don't divide GMS fields!
SELECT total_gms / 100.0 FROM listing_mart.listing_gms
```

### Exchange Rates

**Market rates are stored as rate/1e7** - divide by 10,000,000:
- `market_rate`, `market_rate_usa` (listings)
- `market_rate` (listing_variations)

```sql
-- CORRECT
market_rate / 10000000.0 AS actual_rate

-- WRONG
market_rate AS actual_rate  -- This is 10 million times too large!
```

### Dates

**All date fields are Unix timestamps in seconds**:
- Convert using `TIMESTAMP_SECONDS()` in BigQuery
- Examples: `original_create_date`, `create_date`, `update_date`, `start_date`, `end_date`

```sql
-- CORRECT
TIMESTAMP_SECONDS(create_date) AS created_at

-- For display
DATE(TIMESTAMP_SECONDS(create_date)) AS created_date
```

---

## Data Freshness

- **Daily updates**: All tables refresh daily via p2 rollup (runs at 1:30 AM UTC)
- **Historical data**: Includes all listings where `original_create_date < current_date()`
- **Active listings**: Filter using `is_active = 1`
- **Current discounts**: Only shows promotions active today

---

## Schema Evolution

### Recent Changes

- **Jan 2026**: Pricing now sourced from inventory summary (falls back to shop_listings)
- **Oct 2025**: USA-specific pricing added (`price_usa`, `price_usa_usd`) for listings created after 2025-10-20
- **Ongoing**: Taxonomy and attribute mappings continuously updated

### Clustering

Tables are clustered for query performance:
- Most tables: Clustered by `listing_id`
- `listing_all_attributes`: Clustered by `attribute_name` (per Google recommendation)

---

## Best Practices

### Performance

1. **Always filter on `is_active = 1`** if you only want active listings
2. **Use clustering fields** in WHERE clauses for better performance
3. **Limit result sets** during exploration
4. **Avoid `listing_all_attributes`** for simple queries (use `listing_attributes` instead)

### Data Quality

1. **Check for NULLs** - Many optional fields can be NULL
2. **Join carefully** - Some tables only contain rows with data (e.g., `listing_current_discounts` only has listings with active discounts)
3. **Understand cardinality**:
   - `listings`, `listing_titles`, `listing_attributes`, `listing_images`, `listing_variations`: 1 row per listing
   - `listing_variation_attributes`: Many rows per listing (one per variation option)
   - `listing_products`: Many rows per listing (one per product)
   - `listing_all_attributes`: MANY rows per listing (one per attribute)

### Common Mistakes

❌ **Don't forget to divide prices by 100**
```sql
-- WRONG - This shows cents!
SELECT price_usd FROM listing_mart.listings
```

✅ **Do this instead**
```sql
-- RIGHT - This shows dollars
SELECT price_usd / 100.0 AS price_dollars FROM listing_mart.listings
```

---

## Support

- **Questions**: Contact #bigquery Slack channel
- **Data Issues**: Contact estein@etsy.com or data-marts@etsy.pagerduty.com
- **Script Source**: [Rollups/auto/p2/daily/listing_mart.sql](https://github.com/etsy/Rollups/blob/main/auto/p2/daily/listing_mart.sql)
- **Detailed Docs**: [go/rollup-docs](https://docs.etsycorp.com/bq-docs/docs/bq-rollups)
