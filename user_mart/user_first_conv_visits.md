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
