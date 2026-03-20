---
id: user_purch_stat_12m
title: etsy-data-warehouse-prod.user_mart.user_purch_stat_12m
sidebar_label: user_purch_stat_12m
---

# user_purch_stat_12m

12-month rolling window user purchase statistics

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_purch_stat_12m`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_purch_stats.sql`

**Primary Key**: `user_id`

**Time Window**: Past 12 months

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Purpose

Aggregated purchase statistics for users including:
- Total purchases and GMS
- Top seller countries purchased from
- Top sellers purchased from
- Category breakdown of purchases
- Purchase day counts

## Key Metrics

- **Total orders** and **total GMS**
- **Top seller countries** 
- **Top sellers** (by order count and GMS)
- **Purchase day counts**
- **Category statistics**

## Query Guidance

### When to Use This Table

- Pre-aggregated purchase statistics for past 12 months
- Faster than aggregating from transaction/visit tables
- User segmentation by purchase behavior
- Top seller/country analysis by user

### Common Query Pattern

```sql
SELECT
    user_id,
    total_orders,
    total_gms_gross,
    top_seller_country,
    top_seller_user_id,
    purchase_days
FROM `etsy-data-warehouse-prod.user_mart.user_purch_stat_12m`
WHERE user_id = 123456789;
```

## Related Tables

- [user_purch_daily](user_purch_daily.md) - Daily purchase metrics
- Other user_purch_stat tables with different time windows
- [user_visit_stats](user_visit_stats.md) - Visit statistics
