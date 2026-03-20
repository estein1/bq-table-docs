---
id: user_visit_stats
title: etsy-data-warehouse-prod.user_mart.user_visit_stats
sidebar_label: user_visit_stats
---

# user_visit_stats

Lifetime (LTD) user visit statistics aggregated across all time

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_visit_stats`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_visit_stats.sql`

**Primary Key**: `user_id`

**Time Window**: Lifetime (since 2010-01-01)

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Purpose

Aggregated visit statistics for users including:
- Visit counts by platform (desktop, iOS, Android)
- Visit counts by device (tablet, handset)
- Top categories browsed
- Top seller countries purchased from
- Conversion metrics

## Key Metrics

- **Total visits** and **converting visits**
- **Platform breakdowns** (desktop/mobile)
- **Device breakdowns** (tablet/handset)
- **Top browsed categories** and **purchased categories**
- **Top seller countries**
- **Visit day counts**

## Query Guidance

### When to Use This Table

- Pre-aggregated visit statistics for lifetime (since 2010-01-01)
- Faster than aggregating from user_visit_daily
- User segmentation by visit behavior
- Top category/seller analysis

### Common Query Pattern

```sql
SELECT
    user_id,
    total_visits,
    total_conv_visits,
    desktop_visits,
    mw_app_visits,
    top_browse_category,
    top_purch_category
FROM `etsy-data-warehouse-prod.user_mart.user_visit_stats`
WHERE user_id = 123456789;
```

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `user_mart.user_visit_daily` | Primary Key | User ID. Primary Key |
| `from_date` | INT64 | Hardcoded | Unix timestamp for 2010-01-01 | From date (unix format) for LTD window |
| `to_date` | INT64 | Calculated | `UNIX_DATE(current_date - 1) * 86400` | To date (unix format) |
| `from_dt` | DATE | Hardcoded | '2010-01-01' | From date (date format) |
| `to_dt` | DATE | Calculated | `current_date - 1` | To date (date format) |
| `top_visit_device` | STRING | Temp: `top_visit_devices` | CASE: greatest of desktop/tablet/handset/undefined visit counts | Top visit device (desktop, tablet, handset, or undefined) |
| `visit_days` | INT64 | `user_mart.user_visit_daily` | `COUNT(DISTINCT _date)` | Number of days visited |
| `browse_taxonomies` | INT64 | `browse.user_taxonomy_metrics` | `COUNT(DISTINCT taxonomy_id)` | Number of taxonomies browsed |
| `browse_categories` | INT64 | `browse.user_taxonomy_metrics` | `COUNT(DISTINCT category)` | Number of categories browsed |
| `browse_days` | INT64 | `browse.user_taxonomy_metrics` | `COUNT(DISTINCT _date)` | Number of days browsed |
| `top_browse_taxonomy` | INT64 | Temp: `top_browse_taxonomies` | Taxonomy with highest listing views (ROW_NUMBER = 1) | Top browse listing taxonomy |
| `top_browse_category` | STRING | Temp: `top_browse_categories` | Category with highest listing views (ROW_NUMBER = 1) | Top browse listing category |
| `top_browse_subcategory` | STRING | Temp: `top_browse_categories` | Subcategory with highest listing views (ROW_NUMBER = 1) | Top browse listing subcategory |
| `last_browse_taxonomy` | INT64 | Temp: `last_browse` | Latest taxonomy browsed (most recent _date, highest views) | Latest taxonomy browsed |
| `last_browse_category` | STRING | Temp: `last_browse` | Latest category browsed (most recent _date, highest views) | Latest category browsed |
| `last_browse_subcategory` | STRING | Temp: `last_browse` | Latest subcategory browsed (most recent _date, highest views) | Latest subcategory browsed |

## Related Tables

- [user_visit_daily](user_visit_daily.md) - Source data for aggregation
- Other user_visit_stats tables with different time windows
- [user_purch_stats](user_purch_stats.md) - Purchase statistics
