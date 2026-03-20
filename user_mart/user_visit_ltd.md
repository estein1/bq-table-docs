---
id: user_visit_ltd
title: etsy-data-warehouse-prod.user_mart.user_visit_ltd
sidebar_label: user_visit_ltd
---

# user_visit_ltd

Lifetime (LTD) visit metrics aggregated per user across all time

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_visit_ltd`

**Source Script:** `Rollups/auto/p2/daily/user_mart_1.sql`

**Primary Key**: `user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_visit_ltd`
WHERE user_id = 123456789;
```

## Related Tables

- [user_visit_daily](user_visit_daily.md) - Daily visit metrics
- [user_profile](user_profile.md) - User demographics
- All other user_mart tables
