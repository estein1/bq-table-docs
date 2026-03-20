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

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `_date` | DATE | `analytics.visits` | `DATE(TIMESTAMP_SECONDS(visit_date))` | Visit date (partition key) |
| `run_date` | INT64 | `analytics.visits` | Visit date in unix format | Unix timestamp of when the rollup ran |
| `user_id` | INT64 | `analytics.visits` | Primary Key | User ID (cluster key) |
| `visits` | INT64 | `analytics.visits` | `COUNT(DISTINCT visit_id)` | Total visits for the day |
| `conv_visits` | INT64 | `analytics.visits` | `SUM(converted)` | Converting visits (resulted in purchase) |
| `cart_adds` | INT64 | `analytics.visits` | `SUM(total_cart_adds)` | Total cart adds |
| `engaged_visits` | INT64 | `analytics.visits` | Visits with duration >300s OR converted OR cart_add OR fav_item OR fav_shop | Engaged visits |
| `abandoned_carts` | INT64 | `analytics.visits` | `SUM(CASE WHEN cart_adds > 0 AND converted = 0 THEN 1 END)` | Abandoned carts |
| **Platform OS Metrics** | | | | **Desktop, iOS, Android, Other, Undefined** |
| `desktop_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'desktop' THEN 1 END)` | Desktop visits |
| `ios_os_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'ios' THEN 1 END)` | iOS visits |
| `android_os_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'android' THEN 1 END)` | Android visits |
| `other_os_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'other' THEN 1 END)` | Other OS visits |
| `undefined_os_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'undefined' THEN 1 END)` | Undefined OS visits |
| `desktop_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'desktop' AND converted = 1 THEN 1 END)` | Desktop converting visits |
| `ios_os_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'ios' AND converted = 1 THEN 1 END)` | iOS converting visits |
| `android_os_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'android' AND converted = 1 THEN 1 END)` | Android converting visits |
| `other_os_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'other' AND converted = 1 THEN 1 END)` | Other OS converting visits |
| `undefined_os_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'undefined' AND converted = 1 THEN 1 END)` | Undefined OS converting visits |
| `desktop_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'desktop' THEN cart_adds END)` | Desktop cart adds |
| `ios_os_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'ios' THEN cart_adds END)` | iOS cart adds |
| `android_os_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'android' THEN cart_adds END)` | Android cart adds |
| `other_os_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'other' THEN cart_adds END)` | Other OS cart adds |
| `undefined_os_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'undefined' THEN cart_adds END)` | Undefined OS cart adds |
| `desktop_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'desktop' THEN duration END)` | Desktop duration in seconds |
| `ios_os_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'ios' THEN duration END)` | iOS duration in seconds |
| `android_os_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'android' THEN duration END)` | Android duration in seconds |
| `other_os_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'other' THEN duration END)` | Other OS duration in seconds |
| `undefined_os_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'undefined' THEN duration END)` | Undefined OS duration in seconds |
| `desktop_fav_item` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'desktop' THEN fav_item_count END)` | Desktop favorite item actions |
| `desktop_fav_shop` | INT64 | `analytics.visits` | `SUM(CASE WHEN platform = 'desktop' THEN fav_shop_count END)` | Desktop favorite shop actions |
| **App Metrics** | | | | **Mobile Web (MW), Sell on Etsy (SOE), Buy on Etsy (BOE), Undefined** |
| `mw_app_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'mobile_web' THEN 1 END)` | Mobile web visits |
| `soe_app_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'soe' THEN 1 END)` | Sell on Etsy app visits |
| `boe_app_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'boe' THEN 1 END)` | Buy on Etsy app visits |
| `undefined_app_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'undefined' THEN 1 END)` | Undefined app visits |
| `mw_app_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'mobile_web' AND converted = 1 THEN 1 END)` | Mobile web converting visits |
| `soe_app_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'soe' AND converted = 1 THEN 1 END)` | SOE app converting visits |
| `boe_app_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'boe' AND converted = 1 THEN 1 END)` | BOE app converting visits |
| `undefined_app_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'undefined' AND converted = 1 THEN 1 END)` | Undefined app converting visits |
| `mw_app_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'mobile_web' THEN cart_adds END)` | Mobile web cart adds |
| `soe_app_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'soe' THEN cart_adds END)` | SOE app cart adds |
| `boe_app_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'boe' THEN cart_adds END)` | BOE app cart adds |
| `undefined_app_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'undefined' THEN cart_adds END)` | Undefined app cart adds |
| `mw_app_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'mobile_web' THEN duration END)` | Mobile web duration in seconds |
| `soe_app_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'soe' THEN duration END)` | SOE app duration in seconds |
| `boe_app_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'boe' THEN duration END)` | BOE app duration in seconds |
| `undefined_app_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'undefined' THEN duration END)` | Undefined app duration in seconds |
| `mw_app_fav_item` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'mobile_web' THEN fav_item_count END)` | Mobile web favorite items |
| `soe_app_fav_item` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'soe' THEN fav_item_count END)` | SOE app favorite items |
| `boe_app_fav_item` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'boe' THEN fav_item_count END)` | BOE app favorite items |
| `undefined_app_fav_item` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'undefined' THEN fav_item_count END)` | Undefined app favorite items |
| `mw_app_fav_shop` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'mobile_web' THEN fav_shop_count END)` | Mobile web favorite shops |
| `soe_app_fav_shop` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'soe' THEN fav_shop_count END)` | SOE app favorite shops |
| `boe_app_fav_shop` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'boe' THEN fav_shop_count END)` | BOE app favorite shops |
| `undefined_app_fav_shop` | INT64 | `analytics.visits` | `SUM(CASE WHEN browser_platform = 'undefined' THEN fav_shop_count END)` | Undefined app favorite shops |
| **Device Metrics** | | | | **Tablet, Handset, Undefined** |
| `tablet_device_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'tablet' THEN 1 END)` | Tablet visits |
| `handset_device_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'handset' THEN 1 END)` | Handset visits |
| `undefined_device_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'undefined' THEN 1 END)` | Undefined device visits |
| `tablet_device_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'tablet' AND converted = 1 THEN 1 END)` | Tablet converting visits |
| `handset_device_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'handset' AND converted = 1 THEN 1 END)` | Handset converting visits |
| `undefined_device_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'undefined' AND converted = 1 THEN 1 END)` | Undefined device converting visits |
| `tablet_device_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'tablet' THEN cart_adds END)` | Tablet cart adds |
| `handset_device_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'handset' THEN cart_adds END)` | Handset cart adds |
| `undefined_device_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'undefined' THEN cart_adds END)` | Undefined device cart adds |
| `tablet_device_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'tablet' THEN duration END)` | Tablet duration in seconds |
| `handset_device_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'handset' THEN duration END)` | Handset duration in seconds |
| `undefined_device_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN device_type = 'undefined' THEN duration END)` | Undefined device duration in seconds |
| **Site Metrics** | | | | **Etsy, Pattern, Leo** |
| `etsy_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'etsy' THEN 1 END)` | Etsy site visits |
| `pattern_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'customshops' THEN 1 END)` | Pattern site visits |
| `leo_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'craft_web' THEN 1 END)` | Leo site visits |
| `etsy_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'etsy' AND converted = 1 THEN 1 END)` | Etsy converting visits |
| `pattern_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'customshops' AND converted = 1 THEN 1 END)` | Pattern converting visits |
| `leo_conv_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'craft_web' AND converted = 1 THEN 1 END)` | Leo converting visits |
| `etsy_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'etsy' THEN cart_adds END)` | Etsy cart adds |
| `pattern_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'customshops' THEN cart_adds END)` | Pattern cart adds |
| `leo_cart_adds` | INT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'craft_web' THEN cart_adds END)` | Leo cart adds |
| `etsy_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'etsy' THEN duration END)` | Etsy duration in seconds |
| `pattern_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'customshops' THEN duration END)` | Pattern duration in seconds |
| `leo_duration` | FLOAT64 | `analytics.visits` | `SUM(CASE WHEN event_source = 'craft_web' THEN duration END)` | Leo duration in seconds |
| **Other Metrics** | | | | |
| `visit_duration_seconds` | FLOAT64 | `analytics.visits` | `SUM(duration)` | Total visit duration in seconds |
| `avg_visit_duration_seconds` | NUMERIC | Calculated | `visit_duration_seconds / visits` | Average visit duration |
| `pages_seen` | INT64 | `analytics.visits` | `SUM(page_views)` | Total pages viewed |
| `blog_visits` | INT64 | `analytics.visits` | `SUM(CASE WHEN is_blog_visit = 1 THEN 1 END)` | Blog visits |

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
