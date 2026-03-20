---
id: user_visit_ltd_analytic
title: etsy-data-warehouse-prod.user_mart.user_visit_ltd_analytic
sidebar_label: user_visit_ltd_analytic
---

# user_visit_ltd_analytic

Analytics-optimized lifetime visit metrics

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_visit_ltd_analytic`

**Source Script:** `Rollups/auto/p2/daily/user_mart_1.sql`

**Primary Key**: `user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `user_mart.user_visit_ltd` | Primary Key | User ID. Primary Key |
| `run_date` | INT64 | `user_mart.user_visit_ltd` | Direct | As-of date (unix format), same for entire table |
| `avg_days_bet_visits` | INT64 | Calculated | `(last_visit_date - first_visit_date) / (days_visited - 1) / 86400` | Average number of days between visits (LTD) |
| `duration_rank` | INT64 | Calculated | `RANK() OVER (ORDER BY visit_duration_seconds DESC)` | Descending rank of LTD user visit duration (highest total = 1) |
| `duration_percentile` | INT64 | Calculated | `NTILE(100) OVER (ORDER BY visit_duration_seconds)` | Percentile of LTD visit duration (top 1% = 100) |
| `visits_decay_12m` | FLOAT64 | `user_mart.user_visit_ltd` | Direct | Number of visits in past 365 days with decay factor applied |
| `visits_decay_p90` | FLOAT64 | Calculated | `PERCENTILE_CONT(visits_decay_12m, 0.9)` across all users | P90 of 12 month visits with decay (same value for all users) |
| `visits_decay_p70` | FLOAT64 | Calculated | `PERCENTILE_CONT(visits_decay_12m, 0.7)` across all users | P70 of 12 month visits with decay (same value for all users) |
| `visits_decay_p50` | FLOAT64 | Calculated | `PERCENTILE_CONT(visits_decay_12m, 0.5)` across all users | P50 of 12 month visits with decay (same value for all users) |

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_visit_ltd_analytic`
WHERE user_id = 123456789;
```

## Related Tables

- [user_visit_daily](user_visit_daily.md) - Daily visit metrics
- [user_profile](user_profile.md) - User demographics
- All other user_mart tables
