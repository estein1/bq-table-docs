---
id: listing_current_discounts
title: listing_mart.listing_current_discounts
sidebar_label: listing_current_discounts
---

# listing_mart.listing_current_discounts

## Table Overview

**Purpose**: Current active discounts/sales for listings. Shows the best (highest percentage) discount currently applicable to each listing. One row per listing with active promotion.

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
| `shop_id` | INT64 | `etsy_shard.seller_marketing_promotion` | From promotion table | Shop identifier |
| `is_active` | INT64 | `listing_mart.listings` | Direct from listings | Active listing indicator |

### Promotion Details

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `promotion_id` | INT64 | `etsy_shard.seller_marketing_promotion` | Direct from seller_marketing_promotion | Promotion identifier. Filtered to discoverability_type=2, promotion_type in (1,2,4,5), no minimum purchase requirements, active date range |
| `promotion_type` | INT64 | `etsy_shard.seller_marketing_promotion` | Direct from seller_marketing_promotion | Type of promotion. **Values**: 1, 2, 4, or 5 |

### Discount

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `pct_discount` | NUMERIC | `etsy_shard.seller_marketing_promotion` | For shop-wide: reward_percent_discount_on_order/100. For listing-specific: reward_percent_discount_on_items_in_set/100 | Discount percentage. **Stored as decimal** (e.g., 0.20 = 20% off). Highest discount is selected when multiple promotions apply |
| `entire_shop` | INT64 | Derived from promotion scope | 1 if promotion has no listing-specific entries in seller_marketing_promotion_listing, 0 if promotion has specific listings | Shop-wide promotion indicator. **Values**: 1 = Applies to entire shop, 0 = Specific to this listing |

### Date Range

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `start_date` | INT64 | `etsy_shard.seller_marketing_promotion` | Direct from seller_marketing_promotion | Promotion start timestamp. **Unix timestamp (seconds)**. Filtered to current_date >= start_date |
| `end_date` | INT64 | `etsy_shard.seller_marketing_promotion` | Direct from seller_marketing_promotion | Promotion end timestamp. **Unix timestamp (seconds)**. Filtered to end_date >= current_date |

---

## Data Quality

### Data Sources

- **Primary**: `etsy_shard.seller_marketing_promotion`
- **Listing-Specific Promotions**: `etsy_shard.seller_marketing_promotion_listing`
- **Listings**: `listing_mart.listings`

---

## Business Logic

### Promotion Eligibility

Promotions included if they meet ALL of these criteria:
- `discoverability_type = 2` (sale/discount)
- `promotion_type IN (1, 2, 4, 5)`
- `condition_min_total_order_price = 0` (no minimum purchase)
- `condition_min_num_order_items = 0` (no item quantity requirement)
- `condition_min_num_items_from_set = 0` (no set requirement)
- `timestamp(current_date()) >= timestamp_seconds(start_date)` (has started)
- `timestamp_seconds(end_date) >= timestamp(current_date())` (not ended)
- `legacy_coupon_id IS NULL` (not a legacy coupon)

### Best Discount Selection

When multiple promotions apply to a listing:
1. **Highest discount percentage** wins (pct_discount DESC)
2. **Tiebreaker**: Latest end_date (end_date DESC)
3. Only one promotion per listing (row_number = 1)

### Shop-Wide vs Listing-Specific

**Shop-Wide (entire_shop = 1)**:
- Applies to all listings in shop
- No entry in seller_marketing_promotion_listing
- Uses `reward_percent_discount_on_order`

**Listing-Specific (entire_shop = 0)**:
- Listed in seller_marketing_promotion_listing
- Uses `reward_percent_discount_on_items_in_set`

---

## Common Use Cases

### Analytics Queries

**Listings with active discounts:**
```sql
SELECT
    listing_id,
    shop_id,
    pct_discount * 100 AS discount_percent,
    CASE WHEN entire_shop = 1 THEN 'Shop-wide' ELSE 'Listing-specific' END AS promotion_scope,
    TIMESTAMP_SECONDS(end_date) AS sale_ends
FROM `etsy-data-warehouse-prod.listing_mart.listing_current_discounts`
WHERE is_active = 1
ORDER BY pct_discount DESC
```

**Average discount by promotion type:**
```sql
SELECT
    promotion_type,
    AVG(pct_discount) * 100 AS avg_discount_percent,
    COUNT(*) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_current_discounts`
WHERE is_active = 1
GROUP BY promotion_type
ORDER BY promotion_type
```

**Shop-wide sales:**
```sql
SELECT
    shop_id,
    COUNT(DISTINCT listing_id) AS listings_on_sale,
    MAX(pct_discount) * 100 AS max_discount_percent
FROM `etsy-data-warehouse-prod.listing_mart.listing_current_discounts`
WHERE entire_shop = 1
AND is_active = 1
GROUP BY shop_id
ORDER BY listings_on_sale DESC
```

**Expiring soon (within 7 days):**
```sql
SELECT
    listing_id,
    pct_discount * 100 AS discount_percent,
    TIMESTAMP_SECONDS(end_date) AS sale_ends
FROM `etsy-data-warehouse-prod.listing_mart.listing_current_discounts`
WHERE is_active = 1
AND TIMESTAMP_SECONDS(end_date) <= TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
ORDER BY end_date
```

---

## Display Formatting Rules

### Discount Percentage
- `pct_discount` → **Multiply by 100** for display
- **Stored as decimal**: 0.15 = 15% off
- Format: "15% off", "20% off", etc.
- Round to nearest integer or 1 decimal place

### Dates
- `start_date`, `end_date` → **Unix timestamps in seconds**
- Convert using `TIMESTAMP_SECONDS()` in BigQuery
- Display as: "Sale ends [date]"
- Show countdown for ending soon: "Ends in 3 days"

### Promotion Scope
- `entire_shop`:
  - 1 = "Shop-wide sale" or "Entire shop on sale"
  - 0 = "On sale" or "Listing discount"

### Promotion Type
- Display as generic "Sale" or "Discount" unless you have type mappings
- Types 1, 2, 4, 5 are included (specific meanings may vary)

---

## Important Notes

### Missing Listings

- **Only includes listings with active promotions**
- Listings without discounts will NOT appear in this table
- Use LEFT JOIN when combining with other listing tables if you need all listings

### Discount Application

- Discount percentages are the maximum discount available
- Actual discount at checkout may depend on other factors
- This table shows seller-created sales, not marketplace-wide promotions

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listing_current_discounts` for:**
- Finding listings currently on sale
- Getting discount percentages
- Identifying shop-wide sales vs listing-specific promotions
- **Only includes active promotions** - No historical data

**Don't use this table for:**
- Historical promotions → This only has current/active discounts
- Listings without discounts → Not all listings are in this table
- Complex promotion rules → Only simple percentage discounts with no minimum purchase requirements

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_shard.seller_marketing_promotion` - Already filtered and processed
- `etsy_shard.seller_marketing_promotion_listing` - Listing scope already determined

**✅ Do join to these mart tables**:
- `listing_mart.listings` - For listing price, availability, shop info
- `listing_mart.listing_titles` - For listing title

### Performance Tips

1. **Clustered by `listing_id`** - Filter on listing_id for best performance
2. **Not all listings have discounts** - Use LEFT JOIN from listings if you need all listings
3. **Best discount is selected** - If multiple promotions apply, highest percentage is chosen
4. **Current date filtering already applied** - All promotions in this table are currently active
5. **pct_discount is decimal** - 0.20 = 20% off

### Common Query Patterns

**Active sales:**
```sql
SELECT
  listing_id,
  pct_discount * 100 AS discount_percent,
  entire_shop,
  end_date
FROM `etsy-data-warehouse-prod.listing_mart.listing_current_discounts`
WHERE is_active = 1
ORDER BY pct_discount DESC
```

**Calculate sale price:**
```sql
SELECT
  l.listing_id,
  l.price_usd / 100.0 AS original_price_dollars,
  d.pct_discount * 100 AS discount_percent,
  (l.price_usd * (1 - d.pct_discount)) / 100.0 AS sale_price_dollars
FROM `etsy-data-warehouse-prod.listing_mart.listings` l
JOIN `etsy-data-warehouse-prod.listing_mart.listing_current_discounts` d USING (listing_id)
WHERE l.is_active = 1
  AND l.listing_id IN (12345, 67890)
```

**Shop-wide sales:**
```sql
SELECT
  shop_id,
  COUNT(*) AS discounted_listings,
  AVG(pct_discount) * 100 AS avg_discount_percent
FROM `etsy-data-warehouse-prod.listing_mart.listing_current_discounts`
WHERE is_active = 1
  AND entire_shop = 1
GROUP BY shop_id
```

**Expiring soon:**
```sql
SELECT
  listing_id,
  pct_discount * 100 AS discount_percent,
  TIMESTAMP_SECONDS(end_date) AS expires_at
FROM `etsy-data-warehouse-prod.listing_mart.listing_current_discounts`
WHERE is_active = 1
  AND end_date < UNIX_SECONDS(TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY))
ORDER BY end_date
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listings` - Core listing data (price, availability, shop info, status flags)
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_attributes` - Attributes, taxonomy, recipient, color
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_variations` - Variation summary
- `listing_mart.listing_variation_attributes` - Individual variation details
- `listing_mart.listing_products` - Product-level inventory and SKUs
- `listing_mart.listing_all_attributes` - Comprehensive attribute data

### Reference Tables
- `listing_mart.all_properties` - Property ID to name lookup
- `listing_mart.all_property_values` - Value ID to value lookup
- `listing_mart.all_custom_property_names` - Custom variation names

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.seller_marketing_promotion` - All promotions (already filtered to active, simple discounts)
- `etsy_shard.seller_marketing_promotion_listing` - Listing-specific promotions (already used to determine entire_shop flag)
