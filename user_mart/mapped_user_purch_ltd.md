---
id: mapped_user_purch_ltd
title: etsy-data-warehouse-prod.user_mart.mapped_user_purch_ltd
sidebar_label: mapped_user_purch_ltd
---

# mapped_user_purch_ltd

Lifetime purchase metrics by mapped_user_id

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.mapped_user_purch_ltd`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_purch.sql`

**Primary Key**: `mapped_user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Purpose

Provides purchase metrics at the mapped_user_id level, sourced from transaction_mart tables. Includes order counts, GMS, categories, and product details.

## Key Metrics

- Orders and GMS (gross/net)
- Product counts and unique products
- Categories purchased
- Purchase recency and frequency
- First/last purchase dates

## Query Guidance

### When to Use This Table

- Lifetime purchase analysis with cross-device user tracking
- Purchase behavior analysis
- Customer segmentation
- Recency, frequency, monetary (RFM) analysis

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.mapped_user_purch_ltd`
WHERE mapped_user_id = 123456789;
```

## Column Reference

**Data aggregated at the mapped_user_id level (cross-device, all-time)**

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `mapped_user_id` | INT64 | `user_mart.user_purch_daily` | Primary Key | **Mapped user ID**. Primary Key. Cross-device identifier |
| `purch_date` | INT64 | Calculated | `MAX(purch_date)` | Last purchase date (unix format) |
| `_date` | DATE | Calculated | `MAX(_date)` | Last purchase date (date format) |
| `first_purch_guest_checkout` | INT64 | `rollups.buyer_first_purchase` | From buyer_first_purchase table | 1 if first purchase was a guest checkout |
| `no_of_etsy_users` | INT64 | `user_mart.user_purch_daily` | `COUNT(DISTINCT user_id WHERE is_guest = 0)` | Number of registered user_ids mapping to this mapped_user_id |
| `no_of_guest_users` | INT64 | `user_mart.user_purch_daily` | `COUNT(DISTINCT user_id WHERE is_guest = 1)` | Number of guest user_ids mapping to this mapped_user_id |
| `orders` | INT64 | `user_mart.user_purch_daily` | `SUM(orders)` across all user_ids | Total orders (LTD) for this mapped_user_id |
| `gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gms_gross)` across all user_ids | Total GMS gross (LTD, **USD dollars**) |
| `gms_net` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gms_net)` across all user_ids | Total GMS net (LTD, **USD dollars**) |
| `gms_gross_12m` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gms_gross)` for past 365 days | GMS gross in past 12 months (**USD dollars**) |
| `gms_net_12m` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gms_net)` for past 365 days | GMS net in past 12 months (**USD dollars**) |
| `leo_gms_gross_12m` | NUMERIC | `user_mart.user_purch_daily` | `SUM(leo_gms_gross)` for past 365 days | Leo GMS gross in past 12 months (**USD dollars**) |
| `leo_gms_net_12m` | NUMERIC | `user_mart.user_purch_daily` | `SUM(leo_gms_net)` for past 365 days | Leo GMS net in past 12 months (**USD dollars**) |
| `avg_quantity_per_order` | NUMERIC | Calculated | `SUM(quantity_items) / SUM(orders)` | Average quantity per order (LTD) |
| `avg_transactions_per_order` | NUMERIC | Calculated | `SUM(items) / SUM(orders)` | Average line items per order (LTD) |
| `avg_order_value` | NUMERIC | Calculated | `SUM(gms_gross) / SUM(orders)` | Average order value (LTD, **USD dollars**) |
| `first_purch_date` | INT64 | Calculated | `MIN(purch_date)` across all user_ids | First purchase date (unix format) |
| `last_purch_date` | INT64 | Calculated | `MAX(purch_date)` across all user_ids | Last purchase date (unix format) |
| `first_purch_date_dt` | DATE | Calculated | `MIN(_date)` across all user_ids | First purchase date (date format) |
| `last_purch_date_dt` | DATE | Calculated | `MAX(_date)` across all user_ids | Last purchase date (date format) |
| `days_purchased` | INT64 | Calculated | `COUNT(DISTINCT _date)` across all user_ids | Number of distinct purchase days (LTD) |
| `days_purchased_12m` | INT64 | Calculated | `COUNT(DISTINCT _date)` for past 365 days | Number of distinct purchase days in past 12 months |
| `leo_days_purchased` | INT64 | Calculated | `SUM(LEAST(leo_orders, 1))` across all user_ids | Number of days with Leo purchases (LTD) |
| `leo_days_purchased_12m` | INT64 | Calculated | `SUM(LEAST(leo_orders, 1))` for past 365 days | Number of days with Leo purchases in past 12 months |
| `gift_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gift_gms_gross)` across all user_ids | Gift GMS gross (LTD, **USD dollars**) |

**Note**: This table also includes all the detailed platform/app/device breakdowns and market-specific metrics found in user_purch_ltd, but aggregated across all user_ids for the mapped_user_id. See the SQL script for the complete list of 80+ columns.

## Related Tables

- Other user_purch_* tables for different aggregation levels
- [user_visit_daily](user_visit_daily.md) - Visit metrics
- [user_profile](user_profile.md) - User demographics
