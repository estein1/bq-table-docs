---
id: mapped_user_purch_daily_analytic
title: etsy-data-warehouse-prod.user_mart.mapped_user_purch_daily_analytic
sidebar_label: mapped_user_purch_daily_analytic
---

# mapped_user_purch_daily_analytic

Analytics-optimized daily purchase metrics by mapped_user_id

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.mapped_user_purch_daily_analytic`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_purch.sql`

**Primary Key**: `mapped_user_id, _date`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Purpose

Provides purchase metrics at the mapped_user_id and date level, sourced from transaction_mart tables. Includes order counts, GMS, categories, and product details.

## Key Metrics

- Orders and GMS (gross/net)
- Product counts and unique products
- Categories purchased
- Purchase recency and frequency
- First/last purchase dates

## Query Guidance

### When to Use This Table

- Daily purchase tracking with cross-device user tracking
- Purchase behavior analysis
- Customer segmentation
- Recency, frequency, monetary (RFM) analysis

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.mapped_user_purch_daily_analytic`
WHERE mapped_user_id = 123456789 AND _date >= '2026-01-01';
```

## Column Reference

**Data aggregated at the mapped_user_id level (guest + registered users with same email)**

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `mapped_user_id` | INT64 | `user_mart.mapped_user_purch_daily` | Primary Key | **Mapped user ID**. Primary Key. Links guest and registered users with same email |
| `purch_date` | INT64 | `user_mart.mapped_user_purch_daily` | Direct | Purchase date (unix format) |
| `_date` | DATE | `user_mart.mapped_user_purch_daily` | Direct | Purchase date (date format) |
| `purch_day_number` | INT64 | Calculated | `ROW_NUMBER() OVER (PARTITION BY mapped_user_id ORDER BY _date)` | Purchase day sequence number for this mapped_user_id |
| `days_since_last_purch` | INT64 | Calculated | `DATE_DIFF(_date, LAG(_date, 1) OVER (...), DAY)` | Days since last purchase for this mapped_user_id |
| `mapped_user_gms_rank` | INT64 | Calculated | `RANK() OVER (PARTITION BY mapped_user_id ORDER BY gms_gross DESC)` | Rank of daily GMS for mapped_user (highest GMS day = 1) |
| `mapped_user_gms_percentile` | INT64 | Calculated | `NTILE(100) OVER (PARTITION BY mapped_user_id ORDER BY gms_gross)` | Percentile of daily GMS for mapped_user (highest = 100) |
| `date_gms_rank` | INT64 | Calculated | `RANK() OVER (PARTITION BY purch_date ORDER BY gms_gross DESC)` | Rank of mapped_user GMS for this date (highest = 1) |
| `date_gms_percentile` | INT64 | Calculated | `NTILE(100) OVER (PARTITION BY purch_date ORDER BY gms_gross)` | Percentile of mapped_user GMS for this date (highest = 100) |

## Related Tables

- Other user_purch_* tables for different aggregation levels
- [user_visit_daily](user_visit_daily.md) - Visit metrics
- [user_profile](user_profile.md) - User demographics
