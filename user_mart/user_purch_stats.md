---
id: user_purch_stats
title: etsy-data-warehouse-prod.user_mart.user_purch_stats
sidebar_label: user_purch_stats
---

# user_purch_stats

Lifetime (LTD) user purchase statistics aggregated across all time

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_purch_stats`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_purch_stats.sql`

**Primary Key**: `user_id`

**Time Window**: Lifetime (since 2005-01-01)

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

- Pre-aggregated purchase statistics for lifetime (since 2005-01-01)
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
FROM `etsy-data-warehouse-prod.user_mart.user_purch_stats`
WHERE user_id = 123456789;
```

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `visit_mart.visits_transactions` | Primary Key | User ID. Primary Key |
| `from_date` | INT64 | Hardcoded | Unix timestamp for 2005-01-01 | From date (unix format) for LTD window |
| `to_date` | INT64 | Calculated | `UNIX_DATE(current_date) * 86400` | To date (unix format) |
| `from_dt` | DATE | Hardcoded | '2005-01-01' | From date (date format) |
| `to_dt` | DATE | Calculated | `current_date` | To date (date format) |
| `seller_countries` | INT64 | `visit_mart.visits_transactions` | `COUNT(DISTINCT seller_country)` | Number of seller countries purchased from |
| `sellers` | INT64 | `visit_mart.visits_transactions` | `COUNT(DISTINCT seller_user_id)` | Number of sellers purchased from |
| `purch_taxonomies` | INT64 | `visit_mart.visits_transactions` | `COUNT(DISTINCT taxonomy_id)` | Number of taxonomies for items purchased |
| `purch_categories` | INT64 | `visit_mart.visits_transactions` | `COUNT(DISTINCT new_category)` | Number of categories for items purchased |
| `purch_channels` | INT64 | `visit_mart.visits_transactions` | `COUNT(DISTINCT top_channel)` | Number of channels for purchase visits |
| `purch_days` | INT64 | `visit_mart.visits_transactions` | `COUNT(DISTINCT purch_date)` | Number of purchase days |
| `top_seller_country` | STRING | Temp: `top_seller_countries` | Seller country with most purchases (ROW_NUMBER = 1 by count, then GMS) | Top seller country purchased from |
| `top_seller` | INT64 | Temp: `top_sellers` | Seller with most purchases (ROW_NUMBER = 1 by count, then GMS) | Top seller purchased from (seller_user_id) |
| `top_purch_taxonomy` | INT64 | Temp: `top_taxonomies` | Taxonomy with most purchases (ROW_NUMBER = 1 by count, then GMS) | Top taxonomy for purchases |
| `top_purch_category` | STRING | Temp: `top_categories` | Category with most purchases (ROW_NUMBER = 1 by count, then GMS) | Top category for purchases |
| `top_purch_subcategory` | STRING | Temp: `top_categories` | Subcategory with most purchases (ROW_NUMBER = 1 by count, then GMS) | Top subcategory for purchases |
| `top_purch_channel` | STRING | Temp: `top_channels` | Channel with most purchases (ROW_NUMBER = 1 by count, then GMS) | Top visit channel for purchase visits |
| `top_purch_device` | STRING | Temp: `top_devices` | Device with most purchases (ROW_NUMBER = 1 by count, then GMS) | Top device for purchase visits |
| `last_purch_taxonomy` | INT64 | Temp: `last_all` | Latest taxonomy purchased (most recent _date, highest GMS) | Last purchase taxonomy |
| `last_purch_category` | STRING | Temp: `last_all` | Latest category purchased (most recent _date, highest GMS) | Last purchase category |
| `last_purch_subcategory` | STRING | Temp: `last_all` | Latest subcategory purchased (most recent _date, highest GMS) | Last purchase subcategory |
| `last_purch_channel` | STRING | Temp: `last_all` | Latest channel purchased (most recent _date, highest GMS) | Last purchase visit channel |
| `last_purch_device` | STRING | Temp: `last_all` | Latest device purchased (most recent _date, highest GMS) | Last purchase visit device |

## Related Tables

- [user_purch_daily](user_purch_daily.md) - Daily purchase metrics
- Other user_purch_stat tables with different time windows
- [user_visit_stats](user_visit_stats.md) - Visit statistics
