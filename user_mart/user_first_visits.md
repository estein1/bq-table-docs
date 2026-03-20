---
id: user_first_visits
title: etsy-data-warehouse-prod.user_mart.user_first_visits
sidebar_label: user_first_visits
---

# user_first_visits

First visit attribution and metadata for each user

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_first_visits`

**Source Script:** `Rollups/auto/p2/daily/user_mart_1.sql`

**Primary Key**: `user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `user_mart.user_visit_daily_analytic` | Primary Key | User ID. Primary Key |
| `visit_id` | INT64 | `visit_mart.visits` | From first visit (ROW_NUMBER = 1 by start_datetime) | Visit ID of user's first visit |
| `run_date` | INT64 | `user_mart.user_visit_daily_analytic` | Unix date of first visit (visit_day_number = 1) | Unix date of user's first visit |
| `start_datetime` | TIMESTAMP | `visit_mart.visits` | From first visit | Start time of user's first visit |
| `registered` | INT64 | `visit_mart.visits` | From first visit | Registered indicator of user's first visit (1 = registered, 0 = guest) |
| `top_channel` | STRING | `visit_mart.visits` | From first visit | Top channel of user's first visit (e.g., 'direct', 'organic', 'paid') |

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_first_visits`
WHERE user_id = 123456789;
```

## Related Tables

- [user_visit_daily](user_visit_daily.md) - Daily visit metrics
- [user_profile](user_profile.md) - User demographics
- All other user_mart tables
