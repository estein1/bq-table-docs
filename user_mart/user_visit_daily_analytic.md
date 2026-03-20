---
id: user_visit_daily_analytic
title: etsy-data-warehouse-prod.user_mart.user_visit_daily_analytic
sidebar_label: user_visit_daily_analytic
---

# user_visit_daily_analytic

Analytics-optimized version of user_visit_daily with additional calculated metrics

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_visit_daily_analytic`

**Source Script:** `Rollups/auto/p2/daily/user_mart_1.sql`

**Primary Key**: `user_id, _date`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `user_mart.user_visit_daily` | Primary Key | User ID. Primary Key |
| `run_date` | INT64 | `user_mart.user_visit_daily` | Direct | Date in Unix format |
| `visit_day_number` | INT64 | Calculated | `ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY run_date)` | Sequence number of visit date for user (1st day visited = 1, 2nd = 2, etc.) |
| `days_since_last_visit` | INT64 | Calculated | `(run_date - LAG(run_date, 1)) / 86400` over user_id, ordered by run_date | Number of days since last visit for this user |
| `user_duration_rank` | INT64 | Calculated | `RANK() OVER (PARTITION BY user_id ORDER BY visit_duration_seconds DESC)` | Descending rank of visit duration for user (longest visit = 1) |
| `user_duration_percentile` | INT64 | Calculated | `NTILE(100) OVER (PARTITION BY user_id ORDER BY visit_duration_seconds)` | Percentile of visit duration for user (top 1% = 100) |
| `date_duration_rank` | INT64 | Calculated | `RANK() OVER (PARTITION BY run_date ORDER BY visit_duration_seconds DESC)` | Descending rank of visit duration for this date (longest visit = 1) |
| `date_duration_percentile` | INT64 | Calculated | `NTILE(100) OVER (PARTITION BY run_date ORDER BY visit_duration_seconds)` | Percentile of visit duration for this date (top 1% = 100) |

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_visit_daily_analytic`
WHERE user_id = 123456789;
```

## Related Tables

- [user_visit_daily](user_visit_daily.md) - Daily visit metrics
- [user_profile](user_profile.md) - User demographics
- All other user_mart tables
