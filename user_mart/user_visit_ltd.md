---
id: user_visit_ltd
title: etsy-data-warehouse-prod.user_mart.user_visit_ltd
sidebar_label: user_visit_ltd
---

# user_visit_ltd

Lifetime (LTD) visit metrics aggregated per user across all time

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_visit_ltd`

**Source Script:** `Rollups/auto/p2/daily/user_mart_1.sql`

**Primary Key**: `user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `user_mart.user_visit_daily` | Primary Key | User ID. Primary Key |
| `run_date` | INT64 | Calculated | `MAX(run_date)` from user_visit_daily | As-of date (unix format) |
| **Platform OS Visit Counts (Lifetime)** | | | | |
| `desktop_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(desktop_visits)` | Total desktop visits (LTD) |
| `ios_os_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(ios_os_visits)` | Total iOS visits (LTD) |
| `android_os_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(android_os_visits)` | Total Android visits (LTD) |
| `other_os_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(other_os_visits)` | Total other OS visits (LTD) |
| `undefined_os_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_os_visits)` | Total undefined OS visits (LTD) |
| **App Visit Counts (Lifetime)** | | | | |
| `mw_app_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(mw_app_visits)` | Total mobile web visits (LTD) |
| `soe_app_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(soe_app_visits)` | Total SOE app visits (LTD) |
| `boe_app_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(boe_app_visits)` | Total BOE app visits (LTD) |
| `undefined_app_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_app_visits)` | Total undefined app visits (LTD) |
| **Device Visit Counts (Lifetime)** | | | | |
| `tablet_device_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(tablet_device_visits)` | Total tablet visits (LTD) |
| `handset_device_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(handset_device_visits)` | Total handset visits (LTD) |
| `undefined_device_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_device_visits)` | Total undefined device visits (LTD) |
| **Site Visit Counts (Lifetime)** | | | | |
| `etsy_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(etsy_visits)` | Total Etsy site visits (LTD) |
| `pattern_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(pattern_visits)` | Total Pattern site visits (LTD) |
| `leo_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(leo_visits)` | Total Leo site visits (LTD) |
| `engaged_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(engaged_visits)` | Total engaged visits (LTD) |
| **Converting Visits by Platform (Lifetime)** | | | | |
| `desktop_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(desktop_conv_visits)` | Total desktop converting visits (LTD) |
| `ios_os_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(ios_os_conv_visits)` | Total iOS converting visits (LTD) |
| `android_os_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(android_os_conv_visits)` | Total Android converting visits (LTD) |
| `other_os_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(other_os_conv_visits)` | Total other OS converting visits (LTD) |
| `undefined_os_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_os_conv_visits)` | Total undefined OS converting visits (LTD) |
| `mw_app_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(mw_app_conv_visits)` | Total mobile web converting visits (LTD) |
| `soe_app_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(soe_app_conv_visits)` | Total SOE app converting visits (LTD) |
| `boe_app_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(boe_app_conv_visits)` | Total BOE app converting visits (LTD) |
| `undefined_app_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_app_conv_visits)` | Total undefined app converting visits (LTD) |
| `tablet_device_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(tablet_device_conv_visits)` | Total tablet converting visits (LTD) |
| `handset_device_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(handset_device_conv_visits)` | Total handset converting visits (LTD) |
| `undefined_device_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_device_conv_visits)` | Total undefined device converting visits (LTD) |
| `etsy_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(etsy_conv_visits)` | Total Etsy converting visits (LTD) |
| `pattern_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(pattern_conv_visits)` | Total Pattern converting visits (LTD) |
| `leo_conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(leo_conv_visits)` | Total Leo converting visits (LTD) |
| **Cart Adds by Platform (Lifetime)** | | | | |
| `desktop_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(desktop_cart_adds)` | Total desktop cart adds (LTD) |
| `ios_os_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(ios_os_cart_adds)` | Total iOS cart adds (LTD) |
| `android_os_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(android_os_cart_adds)` | Total Android cart adds (LTD) |
| `other_os_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(other_os_cart_adds)` | Total other OS cart adds (LTD) |
| `undefined_os_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_os_cart_adds)` | Total undefined OS cart adds (LTD) |
| `mw_app_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(mw_app_cart_adds)` | Total mobile web cart adds (LTD) |
| `soe_app_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(soe_app_cart_adds)` | Total SOE app cart adds (LTD) |
| `boe_app_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(boe_app_cart_adds)` | Total BOE app cart adds (LTD) |
| `undefined_app_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_app_cart_adds)` | Total undefined app cart adds (LTD) |
| `tablet_device_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(tablet_device_cart_adds)` | Total tablet cart adds (LTD) |
| `handset_device_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(handset_device_cart_adds)` | Total handset cart adds (LTD) |
| `undefined_device_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_device_cart_adds)` | Total undefined device cart adds (LTD) |
| `etsy_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(etsy_cart_adds)` | Total Etsy cart adds (LTD) |
| `pattern_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(pattern_cart_adds)` | Total Pattern cart adds (LTD) |
| `leo_cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(leo_cart_adds)` | Total Leo cart adds (LTD) |
| **Core Visit Metrics** | | | | |
| `visits` | INT64 | `user_mart.user_visit_daily` | `SUM(visits)` | Total visits (LTD) |
| `conv_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(conv_visits)` | Total converting visits (LTD) |
| `cart_adds` | INT64 | `user_mart.user_visit_daily` | `SUM(cart_adds)` | Total cart adds (LTD) |
| `abandoned_carts` | INT64 | `user_mart.user_visit_daily` | `SUM(abandoned_carts)` | Total abandoned carts (LTD) |
| `blog_visits` | INT64 | `user_mart.user_visit_daily` | `SUM(blog_visits)` | Total blog visits (LTD) |
| `visit_duration_seconds` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(visit_duration_seconds)` | Total visit duration in seconds (LTD) |
| `pages_seen` | INT64 | `user_mart.user_visit_daily` | `SUM(pages_seen)` | Total pages viewed (LTD) |
| **Duration by Platform (Lifetime)** | | | | |
| `desktop_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(desktop_duration)` | Total desktop duration in seconds (LTD) |
| `ios_os_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(ios_os_duration)` | Total iOS duration in seconds (LTD) |
| `android_os_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(android_os_duration)` | Total Android duration in seconds (LTD) |
| `other_os_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(other_os_duration)` | Total other OS duration in seconds (LTD) |
| `undefined_os_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(undefined_os_duration)` | Total undefined OS duration in seconds (LTD) |
| `mw_app_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(mw_app_duration)` | Total mobile web duration in seconds (LTD) |
| `soe_app_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(soe_app_duration)` | Total SOE app duration in seconds (LTD) |
| `boe_app_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(boe_app_duration)` | Total BOE app duration in seconds (LTD) |
| `undefined_app_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(undefined_app_duration)` | Total undefined app duration in seconds (LTD) |
| `tablet_device_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(tablet_device_duration)` | Total tablet duration in seconds (LTD) |
| `handset_device_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(handset_device_duration)` | Total handset duration in seconds (LTD) |
| `undefined_device_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(undefined_device_duration)` | Total undefined device duration in seconds (LTD) |
| `etsy_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(etsy_duration)` | Total Etsy duration in seconds (LTD) |
| `pattern_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(pattern_duration)` | Total Pattern duration in seconds (LTD) |
| `leo_duration` | FLOAT64 | `user_mart.user_visit_daily` | `SUM(leo_duration)` | Total Leo duration in seconds (LTD) |
| **Favorite Actions (Lifetime)** | | | | |
| `desktop_fav_item` | INT64 | `user_mart.user_visit_daily` | `SUM(desktop_fav_item)` | Total desktop favorite items (LTD) |
| `mw_app_fav_item` | INT64 | `user_mart.user_visit_daily` | `SUM(mw_app_fav_item)` | Total mobile web favorite items (LTD) |
| `soe_app_fav_item` | INT64 | `user_mart.user_visit_daily` | `SUM(soe_app_fav_item)` | Total SOE app favorite items (LTD) |
| `boe_app_fav_item` | INT64 | `user_mart.user_visit_daily` | `SUM(boe_app_fav_item)` | Total BOE app favorite items (LTD) |
| `undefined_app_fav_item` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_app_fav_item)` | Total undefined app favorite items (LTD) |
| `desktop_fav_shop` | INT64 | `user_mart.user_visit_daily` | `SUM(desktop_fav_shop)` | Total desktop favorite shops (LTD) |
| `mw_app_fav_shop` | INT64 | `user_mart.user_visit_daily` | `SUM(mw_app_fav_shop)` | Total mobile web favorite shops (LTD) |
| `soe_app_fav_shop` | INT64 | `user_mart.user_visit_daily` | `SUM(soe_app_fav_shop)` | Total SOE app favorite shops (LTD) |
| `boe_app_fav_shop` | INT64 | `user_mart.user_visit_daily` | `SUM(boe_app_fav_shop)` | Total BOE app favorite shops (LTD) |
| `undefined_app_fav_shop` | INT64 | `user_mart.user_visit_daily` | `SUM(undefined_app_fav_shop)` | Total undefined app favorite shops (LTD) |
| **Date Metrics** | | | | |
| `first_visit_date` | INT64 | Calculated | `MIN(run_date)` | First visit date (unix format) |
| `last_visit_date` | INT64 | Calculated | `MAX(run_date)` | Last visit date (unix format) |
| `last_boe_visit_date` | INT64 | Calculated | `MAX(CASE WHEN boe_app_visits > 0 THEN run_date END)` | Last BOE visit date (unix format) |
| `last_cart_add_date` | INT64 | Calculated | `MAX(CASE WHEN cart_adds > 0 THEN run_date END)` | Last cart add date (unix format) |
| `days_visited` | INT64 | Calculated | `COUNT(DISTINCT _date)` | Number of days visited (LTD) |
| `avg_visit_duration_seconds` | NUMERIC | Calculated | `visit_duration_seconds / visits` | Average visit duration in seconds |
| **12-Month Metrics** | | | | |
| `visits_decay_12m` | FLOAT64 | Calculated | `SUM(exp(-0.00770164 * days_ago) * visits)` for past 365 days | Visits with decay factor applied (past 12 months) |
| `visits_12m` | INT64 | Calculated | `SUM(visits)` for past 365 days | Total visits in past 12 months |
| `days_visited_12m` | INT64 | Calculated | `COUNT(DISTINCT _date)` for past 365 days | Days visited in past 12 months |

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_visit_ltd`
WHERE user_id = 123456789;
```

## Related Tables

- [user_visit_daily](user_visit_daily.md) - Daily visit metrics
- [user_profile](user_profile.md) - User demographics
- All other user_mart tables
