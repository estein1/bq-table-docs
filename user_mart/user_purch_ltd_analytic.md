---
id: user_purch_ltd_analytic
title: etsy-data-warehouse-prod.user_mart.user_purch_ltd_analytic
sidebar_label: user_purch_ltd_analytic
---

# user_purch_ltd_analytic

Analytics-optimized lifetime purchase metrics

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_purch_ltd_analytic`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_purch.sql`

**Primary Key**: `user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Purpose

Provides purchase metrics at the user_id level, sourced from transaction_mart tables. Includes order counts, GMS, categories, and product details.

## Key Metrics

- Orders and GMS (gross/net)
- Product counts and unique products
- Categories purchased
- Purchase recency and frequency
- First/last purchase dates

## Query Guidance

### When to Use This Table

- Lifetime purchase analysis
- Purchase behavior analysis
- Customer segmentation
- Recency, frequency, monetary (RFM) analysis

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_purch_ltd_analytic`
WHERE user_id = 123456789;
```

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `user_mart.user_purch_ltd` | Primary Key | User ID. Primary Key |
| `purch_date` | INT64 | `user_mart.user_purch_ltd` | Direct | Last purchase date (unix format) |
| `_date` | DATE | `user_mart.user_purch_ltd` | Direct | Last purchase date (date format) |
| `avg_days_bet_purch` | INT64 | Calculated | `DATE_DIFF(last_purch_date_dt, first_purch_date_dt, DAY) / (days_purchased - 1)` | Average days between purchases (LTD) |
| `gms_gross_rank` | INT64 | Calculated | `RANK() OVER (ORDER BY gms_gross DESC)` | Rank of user LTD GMS gross (highest user = 1) |
| `gms_gross_percentile` | INT64 | Calculated | `NTILE(100) OVER (ORDER BY gms_gross)` | Percentile of user LTD GMS gross (highest user = 100) |
| `gms_net_rank` | INT64 | Calculated | `RANK() OVER (ORDER BY gms_net DESC)` | Rank of user LTD GMS net (highest user = 1) |
| `gms_net_percentile` | INT64 | Calculated | `NTILE(100) OVER (ORDER BY gms_net)` | Percentile of user LTD GMS net (highest user = 100) |
| `gms_gross_12m_rank` | INT64 | Calculated | `RANK() OVER (ORDER BY gms_gross_12m DESC)` | Rank of user 12 month GMS gross (highest user = 1) |
| `gms_gross_12m_percentile` | INT64 | Calculated | `NTILE(100) OVER (ORDER BY gms_gross_12m)` | Percentile of user 12 month GMS gross (highest user = 100) |
| `gms_net_12m_rank` | INT64 | Calculated | `RANK() OVER (ORDER BY gms_net_12m DESC)` | Rank of user 12 month GMS net (highest user = 1) |
| `gms_net_12m_percentile` | INT64 | Calculated | `NTILE(100) OVER (ORDER BY gms_net_12m)` | Percentile of user 12 month GMS net (highest user = 100) |

## Related Tables

- Other user_purch_* tables for different aggregation levels
- [user_visit_daily](user_visit_daily.md) - Visit metrics
- [user_profile](user_profile.md) - User demographics
