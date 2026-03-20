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
