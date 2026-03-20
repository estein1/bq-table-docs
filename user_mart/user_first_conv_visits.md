---
id: user_first_conv_visits
title: etsy-data-warehouse-prod.user_mart.user_first_conv_visits
sidebar_label: user_first_conv_visits
---

# user_first_conv_visits

First converting visit attribution for each user

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_first_conv_visits`

**Source Script:** `Rollups/auto/p2/daily/user_mart_1.sql`

**Primary Key**: `user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `user_mart.user_visit_daily` | Primary Key | User ID. Primary Key |
| `visit_id` | INT64 | `visit_mart.visits` | From first converting visit (ROW_NUMBER = 1 by start_datetime where converted = 1) | Visit ID of user's first converting visit |
| `run_date` | INT64 | `user_mart.user_visit_daily` | Unix date of first converting visit (MIN where conv_visits >= 1) | Unix date of user's first converting visit |
| `start_datetime` | TIMESTAMP | `visit_mart.visits` | From first converting visit | Start time of user's first converting visit |
| `registered` | INT64 | `visit_mart.visits` | From first converting visit | Registered indicator of user's first converting visit |
| `top_channel` | STRING | `visit_mart.visits` | From first converting visit | Top channel of user's first converting visit (e.g., 'direct', 'organic', 'paid') |

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_first_conv_visits`
WHERE user_id = 123456789;
```

## Related Tables

- [user_visit_daily](user_visit_daily.md) - Daily visit metrics
- [user_profile](user_profile.md) - User demographics
- All other user_mart tables
