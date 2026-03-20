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
