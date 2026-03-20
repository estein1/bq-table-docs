---
id: mapped_user_purch_ltd_analytic
title: etsy-data-warehouse-prod.user_mart.mapped_user_purch_ltd_analytic
sidebar_label: mapped_user_purch_ltd_analytic
---

# mapped_user_purch_ltd_analytic

Analytics-optimized lifetime purchase metrics by mapped_user_id

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.mapped_user_purch_ltd_analytic`

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
FROM `etsy-data-warehouse-prod.user_mart.mapped_user_purch_ltd_analytic`
WHERE mapped_user_id = 123456789;
```

## Column Reference

**Data aggregated at the mapped_user_id level (guest + registered users with same email)**

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `mapped_user_id` | INT64 | `user_mart.mapped_user_purch_ltd` | Primary Key | **Mapped user ID**. Primary Key. Links guest and registered users with same email |
| `purch_date` | INT64 | `user_mart.mapped_user_purch_ltd` | Direct | Last purchase date (unix format) |
| `_date` | DATE | `user_mart.mapped_user_purch_ltd` | Direct | Last purchase date (date format) |
| `avg_days_bet_purch` | INT64 | Calculated | `DATE_DIFF(last_purch_date_dt, first_purch_date_dt, DAY) / (days_purchased - 1)` | Average days between purchases (LTD) for this mapped_user_id |
| `gms_gross_rank` | INT64 | Calculated | `RANK() OVER (ORDER BY gms_gross DESC)` | Rank of mapped_user LTD GMS gross (highest = 1) |
| `gms_gross_percentile` | INT64 | Calculated | `NTILE(100) OVER (ORDER BY gms_gross)` | Percentile of mapped_user LTD GMS gross (highest = 100) |
| `gms_gross_12m_rank` | INT64 | Calculated | `RANK() OVER (ORDER BY gms_gross_12m DESC)` | Rank of mapped_user 12 month GMS gross (highest = 1) |
| `gms_gross_12m_percentile` | INT64 | Calculated | `NTILE(100) OVER (ORDER BY gms_gross_12m)` | Percentile of mapped_user 12 month GMS gross (highest = 100) |

## Related Tables

- Other user_purch_* tables for different aggregation levels
- [user_visit_daily](user_visit_daily.md) - Visit metrics
- [user_profile](user_profile.md) - User demographics
