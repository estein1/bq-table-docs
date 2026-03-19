---
id: listing_images
title: listing_mart.listing_images
sidebar_label: listing_images
---

# listing_mart.listing_images

## Table Overview

**Purpose**: Listing image URLs, with up to 20 images per listing pivoted into columns. Provides easy access to primary listing image and additional images.

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
| `is_active` | INT64 | `listing_mart.listings` | Direct from listings | Active listing indicator |

### Image Counts

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `image_count` | INT64 | Calculated from image records | Total images minus 1 (excludes primary image from count) | Number of additional images beyond the primary image. Primary image (rank 0) is not included in this count |

### Primary Image

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_image_id` | INT64 | `etsy_shard.listing_images`, `listings.image_key` | From listing_images rank=0, or extracted from image_key via regex if not in listing_images | Primary image ID. Image with rank = 0. Falls back to regex extraction from image_key if needed |
| `listing_image_url` | STRING | `etsy_shard.images` | Constructs URL from image_id, storage_volume, version, shop_id, and secret. Format: il_170x135 thumbnail | Primary image URL. 170x135 pixel thumbnail URL for rank 0 image |

### Additional Images

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `image_url1` | STRING | `etsy_shard.images` | Pivoted from rank=1 image, same URL construction as primary | First additional image URL. 170x135 pixel thumbnail. NULL if listing has only 1 image |
| `image_url2` | STRING | `etsy_shard.images` | Pivoted from rank=2 image | Second additional image URL. 170x135 pixel thumbnail. NULL if listing has ≤2 images |
| `image_url3` | STRING | `etsy_shard.images` | Pivoted from rank=3 image | Third additional image URL. 170x135 pixel thumbnail. NULL if listing has ≤3 images |
| `image_url4` - `image_url20` | STRING | `etsy_shard.images` | Pivoted from rank=4-20 images | Additional image URLs. 170x135 pixel thumbnails. Up to 20 additional images supported. NULL if not present |

---

## Data Quality

### Data Sources

- **Primary**: `etsy_shard.listing_images` (filtered to latest by update_date, rank-based ordering)
- **Fallback**: `listing_mart.listings.image_key` (used for rank 0 if no listing_images row)
- **Image Details**: `etsy_shard.images` (for constructing URLs)

---

## Business Logic

### Image Ranking

- **Rank 0**: Primary listing image (always present)
- **Rank 1-20**: Additional images in display order
- Images are deduped by listing_id and rank, taking most recent by update_date

### URL Construction

Image URLs are constructed using this format:
```
http://img{image_id % 2}.etsystatic.com/{storage_volume}/{version}/{shop_id}/il_170x135.{image_id}_{secret}.jpg
```

- **Size**: `il_170x135` = 170x135 pixel thumbnail
- **Server**: `img0` or `img1` based on image_id (mod 2)
- **Secret**: First 30 characters of secret, omitted if NULL or empty

### Image Count

- Calculated as total images - 1 (excludes primary image)
- 0 = Only primary image exists
- Maximum of 20 additional images

---

## Common Use Cases

### Analytics Queries

**Listings with multiple images:**
```sql
SELECT
    listing_id,
    image_count,
    listing_image_url,
    image_url1,
    image_url2
FROM `etsy-data-warehouse-prod.listing_mart.listing_images`
WHERE image_count > 0
AND is_active = 1
LIMIT 100
```

**Get all image URLs for a listing:**
```sql
SELECT
    listing_id,
    listing_image_url AS primary_image,
    ARRAY_CONCAT_AGG([
        image_url1, image_url2, image_url3, image_url4, image_url5,
        image_url6, image_url7, image_url8, image_url9, image_url10,
        image_url11, image_url12, image_url13, image_url14, image_url15,
        image_url16, image_url17, image_url18, image_url19, image_url20
    ]) AS additional_images
FROM `etsy-data-warehouse-prod.listing_mart.listing_images`
WHERE listing_id = 12345
```

**Distribution of image counts:**
```sql
SELECT
    image_count,
    COUNT(*) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_images`
WHERE is_active = 1
GROUP BY image_count
ORDER BY image_count
```

**Listings with 10+ images:**
```sql
SELECT
    listing_id,
    image_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_images`
WHERE image_count >= 10
AND is_active = 1
ORDER BY image_count DESC
```

---

## Display Formatting Rules

### Image URLs
- All URLs are 170x135 pixel thumbnails
- To get different sizes, modify the URL pattern:
  - `il_170x135` → Small thumbnail
  - `il_570xN` → Medium size
  - `il_fullxfull` → Full size
- Example: Replace `il_170x135` with `il_570xN` in the URL

### Image Count
- Display as: "X additional images" or "X photos"
- 0 = "1 photo" (only primary)
- 1 = "2 photos" (primary + 1)

### Missing Images
- NULL image_urlX columns mean the listing doesn't have that many images
- Don't display broken image icons for NULL URLs

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listing_images` for:**
- Getting primary listing image URL
- Counting how many images a listing has
- Accessing up to 20 additional image URLs
- Image availability analysis

**Don't use this table for:**
- More than 20 images per listing (rare, but possible in source data)
- Image metadata beyond URLs → Join to `etsy_shard.images` if needed

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_shard.listing_images` - URLs already constructed and pivoted
- `etsy_shard.images` - Storage details already used to build URLs

**✅ Do join to these mart tables**:
- `listing_mart.listings` - For listing details (join on listing_id)
- `listing_mart.listing_titles` - For title to display with images

### Performance Tips

1. **Clustered by `listing_id`** - Always filter on listing_id when possible
2. **Image URLs are pre-constructed** - No need to build them yourself
3. **Use `image_count` for filtering** - More efficient than checking if image_url1 IS NOT NULL
4. **All URLs are 170x135 thumbnails** - Modify the URL pattern (change `il_170x135` to `il_570xN` or `il_fullxfull`) for different sizes

### Common Query Patterns

**Get images for specific listings:**
```sql
SELECT
  listing_id,
  listing_image_url,
  image_count,
  image_url1,
  image_url2
FROM `etsy-data-warehouse-prod.listing_mart.listing_images`
WHERE listing_id IN (12345, 67890, 11111)
```

**Listings with multiple images:**
```sql
SELECT listing_id, image_count, listing_image_url
FROM `etsy-data-warehouse-prod.listing_mart.listing_images`
WHERE is_active = 1
  AND image_count >= 5
```

**Get all images as array (for listings with many images):**
```sql
SELECT
  listing_id,
  [listing_image_url, image_url1, image_url2, image_url3, image_url4,
   image_url5, image_url6, image_url7, image_url8, image_url9, image_url10] AS all_image_urls
FROM `etsy-data-warehouse-prod.listing_mart.listing_images`
WHERE listing_id = 12345
```

**Change image size in URL:**
```sql
SELECT
  listing_id,
  REPLACE(listing_image_url, 'il_170x135', 'il_570xN') AS medium_image_url,
  REPLACE(listing_image_url, 'il_170x135', 'il_fullxfull') AS full_image_url
FROM `etsy-data-warehouse-prod.listing_mart.listing_images`
WHERE listing_id = 12345
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listings` - Core listing data (price, availability, shop info)
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_attributes` - Attributes, taxonomy, recipient, color
- `listing_mart.listing_variations` - Variation summary
- `listing_mart.listing_variation_attributes` - Individual variation details
- `listing_mart.listing_products` - Product-level inventory and SKUs
- `listing_mart.listing_current_discounts` - Active promotions
- `listing_mart.listing_all_attributes` - Comprehensive attribute data

### Reference Tables
- `listing_mart.all_properties` - Property ID to name lookup
- `listing_mart.all_property_values` - Value ID to value lookup
- `listing_mart.all_custom_property_names` - Custom variation names

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listing_images` - Raw image rankings (URLs already constructed and pivoted here)
- `etsy_shard.images` - Image storage details (already used to build URLs here)
