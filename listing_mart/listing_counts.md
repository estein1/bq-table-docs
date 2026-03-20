---
id: listing_counts
title: etsy-data-warehouse-prod.listing_mart.listing_counts
sidebar_label: listing_counts
---

# listing_counts

Count aggregates for listing attributes, tags, materials, products, views, favorites, images, and variations.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_counts`

**Source Script:** `Rollups/auto/p2/daily/listing_mart_post_analytics.sql`

**Primary Key:** `listing_id`

**Clustering:** `listing_id`

**Update Frequency:** Daily (full refresh)

**Owner:** estein@etsy.com

**Owner Team:** cdata-alerts@etsy.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `listing_mart.listings` | Primary Key | Listing id. Primary Key |
| `is_active` | INT64 | `listing_mart.listings` | Direct | 1 if listing is active, 0 if inactive |
| `attribute_count` | INT64 | `etsy_shard.listing_attributes` + `etsy_index.properties` | Count distinct ott_attribute_id where state = 1 | Number of distinct attributes (OTT attributes only, state = 1) |
| `tag_count` | INT64 | `etsy_shard.listings_tags` | Count distinct sequence | Number of listing tags |
| `material_count` | INT64 | `etsy_shard.listings_materials` | Count distinct sequence | Number of listing materials |
| `products_count_active` | INT64 | `etsy_shard.listing_products` + `etsy_shard.product_offerings` | Count distinct product_id where state = 1 | Number of active product offerings |
| `products_count` | INT64 | Multiple sources | Prefers listing_inventory_channel_summary in some cases | Number of products (adjusted for data quality) |
| `view_count` | INT64 | `analytics.listing_views` | Count all listing views (all-time) | **All-time listing view count** |
| `favorite_count` | INT64 | `etsy_shard.users_favoritelistings` | Count all favorites | All-time count of users who favorited this listing |
| `image_count` | INT64 | `listing_mart.listing_images` | From pre-calculated table | Number of listing images (up to 20) |
| `variation_count` | INT64 | `listing_mart.listing_variations` | From pre-calculated table | Number of listing variations (0, 1, or 2) |

## Query Guidance

### When to Use This Table

- Quick aggregates for listing metadata counts
- Filtering listings by number of attributes, tags, images, etc.
- Analyzing listing completeness (tags, materials, images)
- Understanding engagement (views, favorites)
- Product and variation analysis

### Avoid These Joins

- ❌ Don't join to `etsy_shard.listing_attributes` - count already here
- ❌ Don't join to `etsy_shard.listings_tags` - count already here
- ❌ Don't join to `analytics.listing_views` - count already here
- ❌ Don't join to `etsy_shard.users_favoritelistings` - count already here

### Performance Tips

- **Always filter on `listing_id`** (clustered column) for best performance
- Filter on `is_active = 1` for active listings only
- Use specific count columns to filter (e.g., `WHERE image_count >= 5`)

### Important Notes

**View count is all-time:**
- `view_count` includes ALL listing views up to current date
- Limited by partition expiration of `analytics.listing_views`
- Not filtered by date range

**Product count adjustment:**
- `products_count` may differ from `products_count_active` in some cases
- Uses inventory summary data when appropriate for data quality

**Depends on analytics table:**
- This table runs later than other listing_mart tables due to `analytics.listing_views` dependency
- Named "post_analytics" in the script for this reason

### Common Query Patterns

**Listings with many images:**
```sql
SELECT
    listing_id,
    image_count,
    view_count,
    favorite_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_counts`
WHERE is_active = 1
  AND image_count >= 10
ORDER BY view_count DESC
LIMIT 100;
```

**Listings with no tags or materials:**
```sql
SELECT
    listing_id,
    tag_count,
    material_count,
    attribute_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_counts`
WHERE is_active = 1
  AND (tag_count = 0 OR material_count = 0)
LIMIT 100;
```

**Most favorited listings:**
```sql
SELECT
    listing_id,
    favorite_count,
    view_count,
    SAFE_DIVIDE(favorite_count, view_count) AS favorite_rate
FROM `etsy-data-warehouse-prod.listing_mart.listing_counts`
WHERE is_active = 1
  AND view_count > 100  -- Minimum views for meaningful rate
ORDER BY favorite_count DESC
LIMIT 100;
```

**Listings with variations:**
```sql
SELECT
    listing_id,
    variation_count,
    products_count_active,
    products_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_counts`
WHERE is_active = 1
  AND variation_count > 0
ORDER BY products_count_active DESC
LIMIT 100;
```

**Metadata completeness analysis:**
```sql
SELECT
    listing_id,
    tag_count,
    material_count,
    attribute_count,
    image_count,
    CASE
        WHEN tag_count >= 13 AND image_count >= 5 AND attribute_count >= 3 THEN 'Complete'
        WHEN tag_count >= 5 AND image_count >= 1 THEN 'Partial'
        ELSE 'Minimal'
    END AS completeness_tier
FROM `etsy-data-warehouse-prod.listing_mart.listing_counts`
WHERE is_active = 1
LIMIT 100;
```

**High view count but low favorites:**
```sql
SELECT
    listing_id,
    view_count,
    favorite_count,
    SAFE_DIVIDE(favorite_count, view_count) AS favorite_rate
FROM `etsy-data-warehouse-prod.listing_mart.listing_counts`
WHERE is_active = 1
  AND view_count >= 1000
  AND SAFE_DIVIDE(favorite_count, view_count) < 0.01  -- Less than 1% favorite rate
ORDER BY view_count DESC
LIMIT 100;
```

## Related Tables

### listing_mart Tables (Join on listing_id)

- [listings](listings.md) - Main listing data
- [listing_images](listing_images.md) - Individual image details
- [listing_variations](listing_variations.md) - Variation details
- [listing_attributes](listing_attributes.md) - Attribute details
- All other listing_mart tables

### Reference Tables

- None

### Source Tables (Avoid - Data Already in Mart)

- `etsy_shard.listing_attributes` - Attributes already counted
- `etsy_shard.listings_tags` - Tags already counted
- `etsy_shard.listings_materials` - Materials already counted
- `etsy_shard.listing_products` - Products already counted
- `etsy_shard.product_offerings` - Products already counted
- `analytics.listing_views` - Views already counted
- `etsy_shard.users_favoritelistings` - Favorites already counted
