---
id: user_purch_daily_analytic
title: etsy-data-warehouse-prod.user_mart.user_purch_daily_analytic
sidebar_label: user_purch_daily_analytic
---

# user_purch_daily_analytic

Analytics-optimized daily purchase metrics with additional calculated fields

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_purch_daily_analytic`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_purch.sql`

**Primary Key**: `user_id, _date`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Purpose

Provides purchase metrics at the user_id and date level, sourced from transaction_mart tables. Includes order counts, GMS, categories, and product details.

## Key Metrics

- Orders and GMS (gross/net)
- Product counts and unique products
- Categories purchased
- Purchase recency and frequency
- First/last purchase dates

## Query Guidance

### When to Use This Table

- Daily purchase tracking
- Purchase behavior analysis
- Customer segmentation
- Recency, frequency, monetary (RFM) analysis

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_purch_daily_analytic`
WHERE user_id = 123456789 AND _date >= '2026-01-01';
```

## Related Tables

- Other user_purch_* tables for different aggregation levels
- [user_visit_daily](user_visit_daily.md) - Visit metrics
- [user_profile](user_profile.md) - User demographics
