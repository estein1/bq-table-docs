# Listing Mart Materialized Views

Active-only materialized views for listing_mart tables, providing faster query performance for active listings.

## Source Script

**Script:** `Rollups/auto/p2/daily/listing_mart_mvs.sql`

**Owner:** estein@etsy.com

**Owner Team:** cdata-alerts@etsy.com

## Overview

All materialized views in this folder follow the same pattern:
- **Filter:** `WHERE is_active = 1` (active listings only)
- **Clustering:** `listing_id`
- **Refresh:** Disabled (`enable_refresh = false`) - manually refreshed by script
- **Purpose:** Faster queries when you only need active listings

## Available Materialized Views

All views are in the `etsy-data-warehouse-prod.listing_mart` schema:

| Materialized View | Base Table | Description |
|-------------------|------------|-------------|
| `listings_active` | [listings](../listings.md) | Active listings with pricing and status |
| `listing_titles_active` | [listing_titles](../listing_titles.md) | Active listing titles and descriptions |
| `listing_attributes_active` | [listing_attributes](../listing_attributes.md) | Active listing attributes (digital, vintage, categories) |
| `listing_all_attributes_active` | [listing_all_attributes](../listing_all_attributes.md) | All active listing attributes (comprehensive) |
| `listing_counts_active` | [listing_counts](../listing_counts.md) | Count aggregates for active listings |
| `listing_current_discounts_active` | [listing_current_discounts](../listing_current_discounts.md) | Active listing discounts and sales |
| `listing_gms_active` | [listing_gms](../listing_gms.md) | Active listing GMS and sales metrics |
| `listing_images_active` | [listing_images](../listing_images.md) | Active listing images |
| `listing_variation_attributes_active` | [listing_variation_attributes](../listing_variation_attributes.md) | Active listing variation attributes |
| `listing_variations_active` | [listing_variations](../listing_variations.md) | Active listing variations |
| `listing_post_purch_active` | [listing_post_purch](../listing_post_purch.md) | Active listing post-purchase data |
| `listing_indicators_active` | [listing_indicators](../listing_indicators.md) | Active listing indicators (POD, how-it's-made, etc.) |

## When to Use Materialized Views

**Use the `_active` materialized view when:**
- You only need active listings (`is_active = 1`)
- You want faster query performance
- You're running common queries on active listings

**Use the base table when:**
- You need both active AND inactive listings
- You need to filter on `is_active = 0` (inactive)
- You're analyzing listing lifecycle (active → inactive)

## Query Performance Benefits

Materialized views are pre-filtered and pre-clustered, providing:
- **Faster scans** - Smaller data size (only active listings)
- **Better clustering** - Optimized for `listing_id` lookups
- **Reduced costs** - Less data scanned per query

## Example Usage

**Using base table (slower for active-only queries):**
```sql
SELECT *
FROM `etsy-data-warehouse-prod.listing_mart.listings`
WHERE is_active = 1
  AND listing_id IN (123, 456, 789);
```

**Using materialized view (faster):**
```sql
SELECT *
FROM `etsy-data-warehouse-prod.listing_mart.listings_active`
WHERE listing_id IN (123, 456, 789);
```

## Common Patterns

**Join multiple active views:**
```sql
SELECT
    l.listing_id,
    l.price_usd / 100.0 AS price_dollars,
    lt.title,
    la.top_category,
    li.listing_image_url,
    g.total_gms,
    g.past_year_orders
FROM `etsy-data-warehouse-prod.listing_mart.listings_active` l
JOIN `etsy-data-warehouse-prod.listing_mart.listing_titles_active` lt USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_attributes_active` la USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_images_active` li USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_gms_active` g USING (listing_id)
WHERE l.listing_id = 123456789;
```

**Active listings with indicators:**
```sql
SELECT
    l.listing_id,
    lt.title,
    i.print_on_demand.is_pod,
    i.how_its_made.label,
    i.flagship_segments.is_personalizable_customizable
FROM `etsy-data-warehouse-prod.listing_mart.listings_active` l
JOIN `etsy-data-warehouse-prod.listing_mart.listing_titles_active` lt USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_indicators_active` i USING (listing_id)
WHERE i.print_on_demand.is_pod = 1
LIMIT 100;
```

## Refresh Schedule

All materialized views are refreshed daily by the `listing_mart_mvs.sql` script using:
```sql
CALL BQ.REFRESH_MATERIALIZED_VIEW("etsy-data-warehouse-prod.listing_mart.<view_name>");
```

Views have `enable_refresh = false`, so they do not auto-refresh and require manual refresh by the script.

## Access Permissions

Most views grant `roles/bigquery.dataViewer` to:
- `group:gcp-etsy-employees@etsy.com`
- `group:bigquery-public-role@etsy.com`

Note: `listing_gms_active` inherits permissions from the listing_mart schema (bucket 3 access).
