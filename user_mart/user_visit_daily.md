---
id: user_visit_daily
title: etsy-data-warehouse-prod.user_mart.user_visit_daily
sidebar_label: user_visit_daily
---

# user_visit_daily

Daily user visit metrics segmented by platform, OS, app, and device. Incremental table updated daily.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_visit_daily`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_incr.sql`

**Primary Key**: `user_id` + `_date`

**Partitioning**: Partitioned by `_date`

**Clustering**: Clustered by `user_id`

**Update Frequency**: Daily (incremental, 1-day lookback)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Column Reference

### Core Metrics
| Column | Type | Description |
|--------|------|-------------|
| `_date` | DATE | Visit date (partition key) |
| `run_date` | INT64 | Unix timestamp of when the rollup ran |
| `user_id` | INT64 | User ID (cluster key) |
| `visits` | INT64 | Total visits for the day |
| `conv_visits` | INT64 | Converting visits (resulted in purchase) |
| `cart_adds` | INT64 | Total cart adds |
| `engaged_visits` | INT64 | Engaged visits |
| `abandoned_carts` | INT64 | Abandoned carts |

### Platform Metrics (Desktop, iOS, Android, Other, Undefined)
- `{platform}_visits` - Visit count by platform
- `{platform}_conv_visits` - Converting visits by platform
- `{platform}_cart_adds` - Cart adds by platform
- `{platform}_duration` - Total duration in seconds by platform
- `{platform}_fav_item` - Favorite item actions by platform (desktop/app only)
- `{platform}_fav_shop` - Favorite shop actions by platform (desktop/app only)

### App Metrics (MW, SOE, BOE, Undefined)
- `{app}_app_visits` - Visits by app
- `{app}_app_conv_visits` - Converting visits by app
- `{app}_app_cart_adds` - Cart adds by app
- `{app}_app_duration` - Duration by app
- `{app}_app_fav_item` - Favorite items by app
- `{app}_app_fav_shop` - Favorite shops by app

### Device Metrics (Tablet, Handset, Undefined)
- `{device}_device_visits` - Visits by device type
- `{device}_device_conv_visits` - Converting visits by device
- `{device}_device_cart_adds` - Cart adds by device
- `{device}_device_duration` - Duration by device

### Site Metrics (Etsy, Pattern, Leo)
- `{site}_visits` - Visits by site
- `{site}_conv_visits` - Converting visits by site
- `{site}_cart_adds` - Cart adds by site
- `{site}_duration` - Duration by site

### Duration & Pages
| Column | Type | Description |
|--------|------|-------------|
| `visit_duration_seconds` | FLOAT64 | Total visit duration in seconds |
| `avg_visit_duration_seconds` | NUMERIC | Average visit duration |
| `pages_seen` | INT64 | Total pages viewed |
| `blog_visits` | INT64 | Blog visits |

## Query Guidance

### When to Use This Table

- Daily user activity analysis
- Platform/device/app breakdown of user behavior
- Conversion analysis by user over time
- Engagement tracking (cart adds, favorites)
- User visit trends and patterns

### Avoid These Joins

- ❌ Don't aggregate manually - use pre-aggregated stats tables instead (`user_visit_stats`, `user_visit_stats_30d`, etc.)

### Performance Tips

- **Always filter on `_date`** (partition key) for best performance
- **Filter on `user_id`** (clustered column) for single-user queries
- Table is incremental - recent dates have complete data
- Use stats tables for lifetime or multi-period aggregations

### Common Query Patterns

**User activity for a specific date:**
```sql
SELECT
    user_id,
    visits,
    conv_visits,
    cart_adds,
    desktop_visits,
    mw_app_visits,
    soe_app_visits
FROM `etsy-data-warehouse-prod.user_mart.user_visit_daily`
WHERE _date = '2026-03-19'
  AND user_id = 123456789;
```

**Top converting users yesterday:**
```sql
SELECT
    user_id,
    visits,
    conv_visits,
    SAFE_DIVIDE(conv_visits, visits) AS conversion_rate,
    cart_adds
FROM `etsy-data-warehouse-prod.user_mart.user_visit_daily`
WHERE _date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND conv_visits > 0
ORDER BY conv_visits DESC
LIMIT 100;
```

**Platform breakdown for a user over past week:**
```sql
SELECT
    _date,
    desktop_visits,
    ios_os_visits,
    android_os_visits,
    mw_app_visits,
    soe_app_visits
FROM `etsy-data-warehouse-prod.user_mart.user_visit_daily`
WHERE _date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND user_id = 123456789
ORDER BY _date DESC;
```

## Related Tables

### user_mart Tables (Join on user_id)

- [user_visit_stats](user_visit_stats.md) - Lifetime visit statistics
- [user_visit_stats_30d](user_visit_stats_30d.md) - 30-day rolling visit stats
- [user_visit_daily_analytic](user_visit_daily_analytic.md) - Analytics-ready daily visit metrics
- [user_profile](user_profile.md) - User profile and demographic data
- [user_purch_daily](user_purch_daily.md) - Daily purchase metrics
- All other user_mart tables

### Source Tables (Avoid - Data Already in Mart)

- Visit data already aggregated from `analytics.visits`
