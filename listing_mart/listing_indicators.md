---
id: listing_indicators
title: etsy-data-warehouse-prod.listing_mart.listing_indicators
sidebar_label: listing_indicators
---

# listing_indicators

Listing indicators including print-on-demand classification, how-it's-made labels, flagship segments, and personalization attributes.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_indicators`

**Source Script:** `Rollups/auto/p2/daily/listing_indicators.sql`

**Primary Key:** `listing_id`

**Clustering:** `listing_id`, `is_active`

**Update Frequency:** Daily (full refresh)

**Owner:** ejimenez@etsy.com

**Owner Team:** data-marts@etsy.pagerduty.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `listing_mart.listings` | Direct | Listing id. Primary Key. Includes active and inactive listings |
| `shop_id` | INT64 | `listing_mart.listings` | Direct | Shop id tied to this listing |
| `is_active` | INT64 | `listing_mart.listings` | Direct | 1 if listing is active, 0 if inactive |
| `print_on_demand` | STRUCT | Multiple | Complex POD detection | Print-on-demand classification based on production partners and API logs |
| `print_on_demand.is_pod` | INT64 | Production partners + API logs | Union of production partner keywords and API logs | 1 if listing is POD (strict definition), 0 otherwise |
| `print_on_demand.platform` | ARRAY<STRING> | Production partners + API logs | Aggregated platform names | Array of POD platform/business names (lowercase, trimmed) |
| `print_on_demand_expanded` | STRUCT | Multiple | Extended POD detection | Expanded POD classification including shipping origin and image matching |
| `print_on_demand_expanded.is_pod_expanded` | INT64 | Multiple sources | Max of all POD signals (excluding digital listings) | 1 if any POD indicator is true, 0 otherwise |
| `print_on_demand_expanded.has_pod_api_log` | INT64 | `data_activation.listing_api_logs_historical_agg` | Matches API keys to known POD apps | 1 if POD detected via API logs |
| `print_on_demand_expanded.has_pod_production_partner` | INT64 | `etsy_shard.shop_production_partner` | Keyword/regex matching on strict POD partners | 1 if POD detected via production partner (strict) |
| `print_on_demand_expanded.has_pod_production_partner_expanded` | INT64 | `etsy_shard.shop_production_partner` | Expanded list of POD production partners | 1 if POD detected via production partner (expanded list) |
| `print_on_demand_expanded.has_pod_shipping_origin` | INT64 | `rollups.receipt_shipping_basics` | USPS tracking MID matching to POD apps | 1 if POD detected via shipping origin analysis |
| `print_on_demand_expanded.has_external_pod_image_match` | INT64 | `etsy_shard.image_recognition_web_detect_annotations` | Image URLs matching POD websites | 1 if listing images hosted on POD platforms |
| `how_its_made` | STRUCT | `etsy_shard.listing_how_its_made_classification` | Most recent classification (algorithm v3 or v1) | How-it's-made classification details |
| `how_its_made.label` | STRING | Classification mapping | L1 classification mapped to display label | Display label: "Made by", "Sourced by", "Handpicked by", "Designed by", or "No label" |
| `how_its_made.listing_how_its_made_classification_id` | INT64 | Direct | Most recent classification ID | Classification record ID |
| `how_its_made.l1_classification_type` | STRING | Classification enum | Mapped from numeric type | Level 1 classification: seller_made, seller_designed, seller_curated, seller_sourced, no_label |
| `how_its_made.l2_classification_type` | STRING | Classification enum | Mapped from numeric type | Level 2 sub-classification (hand_crafted, digital, with_ai, buyer_personalized_pod, etc.) |
| `flagship_segments` | STRUCT | `staging.flagship_segments` | Pre-calculated upstream | Flagship segment indicators |
| `flagship_segments.is_home_decor_and_improvement` | INT64 | Pre-calculated | Direct | 1 if listing is in home decor segment |
| `flagship_segments.is_kids_and_baby` | INT64 | Pre-calculated | Direct | 1 if listing is in kids & baby segment |
| `flagship_segments.is_personalizable_customizable` | INT64 | Based on personalization struct | Direct | 1 if listing is personalizable |
| `personalization` | STRUCT | `staging.perso_listing_labels` | Pre-calculated upstream | Personalization details |
| `personalization.is_personalizable` | INT64 | Direct | Direct | 1 if seller enabled personalization box for this listing |
| `personalization.title` | STRING | Direct | Direct | Listing title |
| `personalization.variation_count` | INT64 | Direct | Direct | Count of listing variations (0, 1, or 2) |
| `personalization.perso_label` | STRING | Direct | Direct | Type of personalization (currently only "Personalized") |
| `personalization.labels` | ARRAY<STRUCT> | Categorization logic | Complex categorization | Array of personalization labels |
| `personalization.labels.full_label` | STRING | Category + type | Combined label | Label describing personalization type and category (e.g. "Personalized Image") |
| `personalization.labels.label_value` | INT64 | Binary flag | Direct | 1 if label applies to this listing |
| `run_date` | DATE | System | Current date at runtime | Date table was created |

## Query Guidance

### When to Use This Table

- Identifying print-on-demand listings with strict or expanded definitions
- Analyzing how-it's-made labels and seller creation methods
- Filtering for flagship segment listings (home decor, kids/baby, personalizable)
- Understanding personalization attributes and categories
- Checking if listings have POD indicators from specific sources (API logs, production partners, shipping, images)

### Avoid These Joins

- ❌ Don't join to `etsy_shard.shop_production_partner_listing_association` - POD data already denormalized here
- ❌ Don't join to `etsy_shard.listing_how_its_made_classification` - most recent classification already here
- ❌ Don't join to `staging.flagship_segments` - already included
- ❌ Don't join to `staging.perso_listing_labels` - already included

### Performance Tips

- **Always filter on `listing_id`** (clustered column) for best performance
- Filter on `is_active = 1` to query only active listings (or use materialized view below)
- Use `print_on_demand.is_pod` for strict POD definition, `print_on_demand_expanded.is_pod_expanded` for broader detection
- Access individual POD signals via `print_on_demand_expanded.has_pod_*` fields
- Check `how_its_made.l2_classification_type = 'buyer_personalized_pod'` for personalized POD listings

### Materialized View

**Active listings only:** `etsy-data-warehouse-prod.listing_mart.listing_indicators_active`
- Pre-filtered for `is_active = 1`
- Faster queries when you only need active listings

### Common Query Patterns

**Get POD listings with platform details:**
```sql
SELECT
    listing_id,
    shop_id,
    print_on_demand.is_pod,
    print_on_demand.platform,
    print_on_demand_expanded.is_pod_expanded
FROM `etsy-data-warehouse-prod.listing_mart.listing_indicators`
WHERE is_active = 1
  AND print_on_demand.is_pod = 1
LIMIT 100;
```

**Analyze POD detection methods:**
```sql
SELECT
    listing_id,
    print_on_demand_expanded.has_pod_api_log,
    print_on_demand_expanded.has_pod_production_partner,
    print_on_demand_expanded.has_pod_production_partner_expanded,
    print_on_demand_expanded.has_pod_shipping_origin,
    print_on_demand_expanded.has_external_pod_image_match,
    print_on_demand.platform
FROM `etsy-data-warehouse-prod.listing_mart.listing_indicators_active`
WHERE print_on_demand_expanded.is_pod_expanded = 1
LIMIT 100;
```

**Get listings by how-it's-made label:**
```sql
SELECT
    how_its_made.label,
    how_its_made.l1_classification_type,
    how_its_made.l2_classification_type,
    COUNT(*) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_indicators_active`
GROUP BY 1, 2, 3
ORDER BY listing_count DESC;
```

**Find personalizable listings in flagship segments:**
```sql
SELECT
    listing_id,
    personalization.is_personalizable,
    personalization.perso_label,
    flagship_segments.is_home_decor_and_improvement,
    flagship_segments.is_kids_and_baby
FROM `etsy-data-warehouse-prod.listing_mart.listing_indicators_active`
WHERE flagship_segments.is_personalizable_customizable = 1;
```

**Analyze personalization labels:**
```sql
SELECT
    listing_id,
    label.full_label
FROM `etsy-data-warehouse-prod.listing_mart.listing_indicators_active`,
UNNEST(personalization.labels) AS label
WHERE label.label_value = 1
  AND personalization.is_personalizable = 1
LIMIT 100;
```

## Related Tables

### listing_mart Tables (Join on listing_id)

- [listings](listings.md) - Main listing data with pricing and status
- [listing_attributes](listing_attributes.md) - Listing attributes (digital, vintage, supplies, categories)
- [listing_titles](listing_titles.md) - Listing titles and descriptions
- All other listing_mart tables

### Reference Tables

- `staging.flagship_segments` - Source for flagship segment data
- `staging.perso_listing_labels` - Source for personalization data
- `etsy_shard.listing_how_its_made_classification` - Raw how-it's-made classifications

### Source Tables (Avoid - Data Already in Mart)

- `etsy_shard.shop_production_partner_listing_association` - Production partner associations (POD already calculated)
- `etsy_shard.shop_production_partner` - Production partner details (POD already calculated)
- `data_activation.listing_api_logs_historical_agg` - API logs (POD already calculated)
- `etsy_shard.image_recognition_web_detect_annotations` - Image matching data (POD already calculated)
- `rollups.receipt_shipping_basics` - Shipping data (POD already calculated)

## Important Notes

### POD Definitions

**Strict POD** (`print_on_demand.is_pod`):
- Based on known production partner names and API logs only
- Most conservative definition
- Recommended for official POD metrics

**Expanded POD** (`print_on_demand_expanded.is_pod_expanded`):
- Includes additional signals: shipping origin analysis and image matching
- Broader detection, may include more false positives
- Useful for exploratory analysis

### How-It's-Made Algorithm

- Uses algorithm v3 (most recent) with fallback to v1
- Classifications can change when sellers update listings or when shown to users
- Check `l2_classification_type = 'buyer_personalized_pod'` to identify personalized POD

### Flagship Segments

- Listings can belong to multiple flagship segments
- Segments calculated upstream in `staging.flagship_segments`
- This is the source-of-truth for flagship segment definitions
