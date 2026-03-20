---
id: user_purch_ltd
title: etsy-data-warehouse-prod.user_mart.user_purch_ltd
sidebar_label: user_purch_ltd
---

# user_purch_ltd

Lifetime (LTD) purchase metrics aggregated across all time per user

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_purch_ltd`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_purch.sql`

**Primary Key**: `user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Purpose

Provides purchase metrics at the user_id level, sourced from transaction_mart tables. Includes order counts, GMS, categories, and product details.

## Key Metrics

- Orders and GMS (gross/net)
- Product counts and unique products
- Categories purchased
- Purchase recency and frequency
- First/last purchase dates

## Query Guidance

### When to Use This Table

- Lifetime purchase analysis
- Purchase behavior analysis
- Customer segmentation
- Recency, frequency, monetary (RFM) analysis

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_purch_ltd`
WHERE user_id = 123456789;
```

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `user_mart.user_purch_daily` | Primary Key | User ID. Primary Key |
| `is_guest` | INT64 | `user_mart.user_purch_daily` | Direct | 1 if guest user |
| `mapped_user_id` | INT64 | `user_mart.user_purch_daily` | Direct | Mapped user ID |
| `purch_date` | INT64 | Calculated | `MAX(purch_date)` | Last purchase date (unix format) |
| `_date` | DATE | Calculated | `MAX(_date)` | Last purchase date (date format) |
| **Lifetime Order Counts by Platform** | | | | **All-time totals** |
| `desktop_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(desktop_orders)` | Total desktop orders (LTD) |
| `ios_os_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(ios_os_orders)` | Total iOS orders (LTD) |
| `android_os_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(android_os_orders)` | Total Android orders (LTD) |
| `other_os_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(other_os_orders)` | Total other OS orders (LTD) |
| `undefined_os_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(undefined_os_orders)` | Total undefined OS orders (LTD) |
| `mw_app_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(mw_app_orders)` | Total mobile web orders (LTD) |
| `soe_app_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(soe_app_orders)` | Total SOE app orders (LTD) |
| `boe_app_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(boe_app_orders)` | Total BOE app orders (LTD) |
| `undefined_app_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(undefined_app_orders)` | Total undefined app orders (LTD) |
| `tablet_device_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(tablet_device_orders)` | Total tablet orders (LTD) |
| `handset_device_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(handset_device_orders)` | Total handset orders (LTD) |
| `undefined_device_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(undefined_device_orders)` | Total undefined device orders (LTD) |
| **Lifetime GMS by Platform** | | | | **USD dollars, all-time** |
| `desktop_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(desktop_gms)` | Total desktop GMS (LTD, **USD dollars**) |
| `ios_os_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(ios_os_gms)` | Total iOS GMS (LTD, **USD dollars**) |
| `android_os_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(android_os_gms)` | Total Android GMS (LTD, **USD dollars**) |
| `other_os_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(other_os_gms)` | Total other OS GMS (LTD, **USD dollars**) |
| `undefined_os_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(undefined_os_gms)` | Total undefined OS GMS (LTD, **USD dollars**) |
| `mw_app_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(mw_app_gms)` | Total mobile web GMS (LTD, **USD dollars**) |
| `soe_app_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(soe_app_gms)` | Total SOE app GMS (LTD, **USD dollars**) |
| `boe_app_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(boe_app_gms)` | Total BOE app GMS (LTD, **USD dollars**) |
| `undefined_app_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(undefined_app_gms)` | Total undefined app GMS (LTD, **USD dollars**) |
| `tablet_device_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(tablet_device_gms)` | Total tablet GMS (LTD, **USD dollars**) |
| `handset_device_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(handset_device_gms)` | Total handset GMS (LTD, **USD dollars**) |
| `undefined_device_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(undefined_device_gms)` | Total undefined device GMS (LTD, **USD dollars**) |
| **Lifetime Item Counts** | | | | |
| `digital_items` | INT64 | `user_mart.user_purch_daily` | `SUM(digital_items)` | Total digital items purchased (LTD) |
| `custom_items` | INT64 | `user_mart.user_purch_daily` | `SUM(custom_items)` | Total custom items purchased (LTD) |
| `gift_items` | INT64 | `user_mart.user_purch_daily` | `SUM(gift_items)` | Total gift items purchased (LTD) |
| `vintage_items` | INT64 | `user_mart.user_purch_daily` | `SUM(vintage_items)` | Total vintage items purchased (LTD) |
| `handmade_items` | INT64 | `user_mart.user_purch_daily` | `SUM(handmade_items)` | Total handmade items purchased (LTD) |
| `supplies_items` | INT64 | `user_mart.user_purch_daily` | `SUM(supplies_items)` | Total supplies items purchased (LTD) |
| `leo_items` | INT64 | `user_mart.user_purch_daily` | `SUM(leo_items)` | Total Leo items purchased (LTD) |
| **Core Metrics** | | | | |
| `orders` | INT64 | `user_mart.user_purch_daily` | `SUM(orders)` | Total orders (LTD) |
| `guest_checkout_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(guest_checkout_orders)` | Total guest checkout orders (LTD) |
| `cancelled_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(cancelled_orders)` | Total cancelled orders (LTD) |
| `non_domestic_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(non_domestic_orders)` | Total non-domestic orders (LTD) |
| `dc_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(dc_orders)` | Total direct checkout orders (LTD) |
| `pattern_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(pattern_orders)` | Total Pattern orders (LTD) |
| `leo_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(leo_orders)` | Total Leo orders (LTD) |
| `gift_card_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(gift_card_orders)` | Total gift card orders (LTD) |
| `ipp_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(ipp_orders)` | Total in-person orders (LTD) |
| `wholesale_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(wholesale_orders)` | Total wholesale orders (LTD) |
| **Market GMS (Lifetime)** | | | | **USD dollars** |
| `non_domestic_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(non_domestic_gms_gross)` | Total non-domestic GMS gross (LTD, **USD dollars**) |
| `dc_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(dc_gms_gross)` | Total direct checkout GMS gross (LTD, **USD dollars**) |
| `pattern_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(pattern_gms_gross)` | Total Pattern GMS gross (LTD, **USD dollars**) |
| `leo_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(leo_gms_gross)` | Total Leo GMS gross (LTD, **USD dollars**) |
| `leo_gms_net` | NUMERIC | `user_mart.user_purch_daily` | `SUM(leo_gms_net)` | Total Leo GMS net (LTD, **USD dollars**) |
| `gift_card_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gift_card_gms_gross)` | Total gift card GMS gross (LTD, **USD dollars**) |
| `ipp_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(ipp_gms_gross)` | Total in-person GMS gross (LTD, **USD dollars**) |
| `wholesale_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(wholesale_gms_gross)` | Total wholesale GMS gross (LTD, **USD dollars**) |
| `guest_checkout_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(guest_checkout_gms_gross)` | Total guest checkout GMS gross (LTD, **USD dollars**) |
| **Quantity Metrics** | | | | |
| `quantity_items` | INT64 | `user_mart.user_purch_daily` | `SUM(quantity_items)` | Total quantity purchased (LTD) |
| `items` | INT64 | `user_mart.user_purch_daily` | `SUM(items)` | Total transaction count (LTD) |
| `cancelled_items` | INT64 | `user_mart.user_purch_daily` | `SUM(cancelled_items)` | Total cancelled items (LTD) |
| **Total GMS (Lifetime)** | | | | **USD dollars** |
| `trans_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(trans_gms_gross)` | Total transaction-based GMS gross (LTD, **USD dollars**) |
| `trans_gms_net` | NUMERIC | `user_mart.user_purch_daily` | `SUM(trans_gms_net)` | Total transaction-based GMS net (LTD, **USD dollars**) |
| `gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gms_gross)` | Total GMS gross (LTD, **USD dollars**) |
| `gms_net` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gms_net)` | Total GMS net (LTD, **USD dollars**) |
| **12-Month Rolling Metrics** | | | | **Past 365 days** |
| `gms_gross_12m` | NUMERIC | `user_mart.user_purch_daily` | `SUM(CASE WHEN date_diff <= 365 THEN gms_gross END)` | GMS gross in past 12 months (**USD dollars**) |
| `gms_net_12m` | NUMERIC | `user_mart.user_purch_daily` | `SUM(CASE WHEN date_diff <= 365 THEN gms_net END)` | GMS net in past 12 months (**USD dollars**) |
| `leo_gms_gross_12m` | NUMERIC | `user_mart.user_purch_daily` | `SUM(CASE WHEN date_diff <= 365 THEN leo_gms_gross END)` | Leo GMS gross in past 12 months (**USD dollars**) |
| `leo_gms_net_12m` | NUMERIC | `user_mart.user_purch_daily` | `SUM(CASE WHEN date_diff <= 365 THEN leo_gms_net END)` | Leo GMS net in past 12 months (**USD dollars**) |
| **Averages** | | | | |
| `avg_quantity_per_order` | NUMERIC | Calculated | `quantity_items / orders` | Average quantity per order (LTD) |
| `avg_transactions_per_order` | NUMERIC | Calculated | `items / orders` | Average line items per order (LTD) |
| `avg_order_value` | NUMERIC | Calculated | `gms_gross / orders` | Average order value (LTD, **USD dollars**) |
| **Date Metrics** | | | | |
| `first_purch_date` | INT64 | Calculated | `MIN(purch_date)` | First purchase date (unix format) |
| `last_purch_date` | INT64 | Calculated | `MAX(purch_date)` | Last purchase date (unix format) |
| `first_purch_date_dt` | DATE | Calculated | `MIN(_date)` | First purchase date (date format) |
| `last_purch_date_dt` | DATE | Calculated | `MAX(_date)` | Last purchase date (date format) |
| `days_purchased` | INT64 | Calculated | `COUNT(DISTINCT _date)` | Number of distinct purchase days (LTD) |
| `days_purchased_12m` | INT64 | Calculated | `SUM(CASE WHEN date_diff <= 365 THEN 1 END)` | Number of distinct purchase days in past 12 months |
| `leo_days_purchased` | INT64 | Calculated | `SUM(LEAST(leo_orders, 1))` | Number of days with Leo purchases (LTD) |
| `leo_days_purchased_12m` | INT64 | Calculated | `SUM(CASE WHEN date_diff <= 365 THEN LEAST(leo_orders, 1) END)` | Number of days with Leo purchases in past 12 months |
| `gift_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gift_gms_gross)` | Total gift GMS gross (LTD, **USD dollars**) |

## Related Tables

- Other user_purch_* tables for different aggregation levels
- [user_visit_daily](user_visit_daily.md) - Visit metrics
- [user_profile](user_profile.md) - User demographics
