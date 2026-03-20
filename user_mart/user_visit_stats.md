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

## Related Tables

- [user_visit_daily](user_visit_daily.md) - Source data for aggregation
- Other user_visit_stats tables with different time windows
- [user_purch_stats](user_purch_stats.md) - Purchase statistics
