---
id: mapped_user_purch_daily
title: etsy-data-warehouse-prod.user_mart.mapped_user_purch_daily
sidebar_label: mapped_user_purch_daily
---

# mapped_user_purch_daily

Daily purchase metrics by mapped_user_id for cross-device tracking

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.mapped_user_purch_daily`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_purch.sql`

**Primary Key**: `mapped_user_id, _date`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

## Purpose

Provides purchase metrics at the mapped_user_id and date level, sourced from transaction_mart tables. Includes order counts, GMS, categories, and product details.

## Key Metrics

- Orders and GMS (gross/net)
- Product counts and unique products
- Categories purchased
- Purchase recency and frequency
- First/last purchase dates

## Query Guidance

### When to Use This Table

- Daily purchase tracking with cross-device user tracking
- Purchase behavior analysis
- Customer segmentation
- Recency, frequency, monetary (RFM) analysis

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.mapped_user_purch_daily`
WHERE mapped_user_id = 123456789 AND _date >= '2026-01-01';
```

## Column Reference

**Data aggregated at the mapped_user_id level (guest + registered users with same email)**

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `mapped_user_id` | INT64 | `user_mart.user_purch_daily` | Primary Key | **Mapped user ID**. Primary Key. Links guest and registered users with same email |
| `purch_date` | INT64 | `user_mart.user_purch_daily` | Direct | Purchase date (unix format) |
| `_date` | DATE | `user_mart.user_purch_daily` | Direct | Purchase date (date format) |
| `no_of_etsy_users` | INT64 | `user_mart.user_purch_daily` | `COUNT(DISTINCT user_id WHERE is_guest = 0)` | Number of purchasing registered users for this mapped_user_id |
| `no_of_guest_users` | INT64 | `user_mart.user_purch_daily` | `COUNT(DISTINCT user_id WHERE is_guest = 1)` | Number of purchasing guest users for this mapped_user_id |
| **Order Counts by Platform (Aggregated across all user_ids)** | | | | |
| `desktop_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(desktop_orders)` | Total desktop orders for this mapped_user_id |
| `ios_os_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(ios_os_orders)` | Total iOS orders for this mapped_user_id |
| `android_os_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(android_os_orders)` | Total Android orders for this mapped_user_id |
| `other_os_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(other_os_orders)` | Total other OS orders for this mapped_user_id |
| `undefined_os_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(undefined_os_orders)` | Total undefined OS orders for this mapped_user_id |
| `mw_app_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(mw_app_orders)` | Total mobile web orders for this mapped_user_id |
| `soe_app_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(soe_app_orders)` | Total SOE app orders for this mapped_user_id |
| `boe_app_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(boe_app_orders)` | Total BOE app orders for this mapped_user_id |
| `undefined_app_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(undefined_app_orders)` | Total undefined app orders for this mapped_user_id |
| `tablet_device_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(tablet_device_orders)` | Total tablet orders for this mapped_user_id |
| `handset_device_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(handset_device_orders)` | Total handset orders for this mapped_user_id |
| `undefined_device_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(undefined_device_orders)` | Total undefined device orders for this mapped_user_id |
| **GMS by Platform (USD dollars)** | | | | **Aggregated across all user_ids** |
| `desktop_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(desktop_gms)` | Desktop GMS (**USD dollars**) for this mapped_user_id |
| `ios_os_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(ios_os_gms)` | iOS GMS (**USD dollars**) for this mapped_user_id |
| `android_os_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(android_os_gms)` | Android GMS (**USD dollars**) for this mapped_user_id |
| `other_os_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(other_os_gms)` | Other OS GMS (**USD dollars**) for this mapped_user_id |
| `undefined_os_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(undefined_os_gms)` | Undefined OS GMS (**USD dollars**) for this mapped_user_id |
| `mw_app_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(mw_app_gms)` | Mobile web GMS (**USD dollars**) for this mapped_user_id |
| `soe_app_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(soe_app_gms)` | SOE app GMS (**USD dollars**) for this mapped_user_id |
| `boe_app_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(boe_app_gms)` | BOE app GMS (**USD dollars**) for this mapped_user_id |
| `undefined_app_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(undefined_app_gms)` | Undefined app GMS (**USD dollars**) for this mapped_user_id |
| `tablet_device_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(tablet_device_gms)` | Tablet GMS (**USD dollars**) for this mapped_user_id |
| `handset_device_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(handset_device_gms)` | Handset GMS (**USD dollars**) for this mapped_user_id |
| `undefined_device_gms` | NUMERIC | `user_mart.user_purch_daily` | `SUM(undefined_device_gms)` | Undefined device GMS (**USD dollars**) for this mapped_user_id |
| **Item Counts** | | | | **Aggregated across all user_ids** |
| `digital_items` | INT64 | `user_mart.user_purch_daily` | `SUM(digital_items)` | Digital items purchased by this mapped_user_id |
| `custom_items` | INT64 | `user_mart.user_purch_daily` | `SUM(custom_items)` | Custom items purchased by this mapped_user_id |
| `gift_items` | INT64 | `user_mart.user_purch_daily` | `SUM(gift_items)` | Gift items purchased by this mapped_user_id |
| `vintage_items` | INT64 | `user_mart.user_purch_daily` | `SUM(vintage_items)` | Vintage items purchased by this mapped_user_id |
| `handmade_items` | INT64 | `user_mart.user_purch_daily` | `SUM(handmade_items)` | Handmade items purchased by this mapped_user_id |
| `supplies_items` | INT64 | `user_mart.user_purch_daily` | `SUM(supplies_items)` | Supplies items purchased by this mapped_user_id |
| `leo_items` | INT64 | `user_mart.user_purch_daily` | `SUM(leo_items)` | Leo items purchased by this mapped_user_id |
| **Core Metrics** | | | | |
| `orders` | INT64 | `user_mart.user_purch_daily` | `SUM(orders)` | Total orders for this mapped_user_id |
| `guest_checkout_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(guest_checkout_orders)` | Guest checkout orders for this mapped_user_id |
| `cancelled_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(cancelled_orders)` | Cancelled orders for this mapped_user_id |
| `non_domestic_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(non_domestic_orders)` | Non-domestic orders for this mapped_user_id |
| `dc_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(dc_orders)` | Direct checkout orders for this mapped_user_id |
| `pattern_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(pattern_orders)` | Pattern orders for this mapped_user_id |
| `leo_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(leo_orders)` | Leo orders for this mapped_user_id |
| `gift_card_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(gift_card_orders)` | Gift card orders for this mapped_user_id |
| `ipp_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(ipp_orders)` | In-person orders for this mapped_user_id |
| `wholesale_orders` | INT64 | `user_mart.user_purch_daily` | `SUM(wholesale_orders)` | Wholesale orders for this mapped_user_id |
| **Market GMS (USD dollars)** | | | | |
| `non_domestic_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(non_domestic_gms_gross)` | Non-domestic GMS gross (**USD dollars**) |
| `dc_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(dc_gms_gross)` | Direct checkout GMS gross (**USD dollars**) |
| `pattern_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(pattern_gms_gross)` | Pattern GMS gross (**USD dollars**) |
| `leo_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(leo_gms_gross)` | Leo GMS gross (**USD dollars**) |
| `leo_gms_net` | NUMERIC | `user_mart.user_purch_daily` | `SUM(leo_gms_net)` | Leo GMS net (**USD dollars**) |
| `gift_card_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gift_card_gms_gross)` | Gift card GMS gross (**USD dollars**) |
| `ipp_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(ipp_gms_gross)` | In-person GMS gross (**USD dollars**) |
| `wholesale_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(wholesale_gms_gross)` | Wholesale GMS gross (**USD dollars**) |
| `guest_checkout_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(guest_checkout_gms_gross)` | Guest checkout GMS gross (**USD dollars**) |
| **Quantity & Item Metrics** | | | | |
| `quantity_items` | INT64 | `user_mart.user_purch_daily` | `SUM(quantity_items)` | Total quantity of items for this mapped_user_id |
| `items` | INT64 | `user_mart.user_purch_daily` | `SUM(items)` | Total transaction count for this mapped_user_id |
| `cancelled_items` | INT64 | `user_mart.user_purch_daily` | `SUM(cancelled_items)` | Cancelled items for this mapped_user_id |
| **Total GMS (USD dollars)** | | | | |
| `trans_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(trans_gms_gross)` | Transaction-based GMS gross (**USD dollars**) |
| `trans_gms_net` | NUMERIC | `user_mart.user_purch_daily` | `SUM(trans_gms_net)` | Transaction-based GMS net (**USD dollars**) |
| `gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gms_gross)` | GMS gross (**USD dollars**) |
| `gms_net` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gms_net)` | GMS net (**USD dollars**) |
| **Calculated Metrics** | | | | |
| `avg_quantity_per_order` | NUMERIC | Calculated | `quantity_items / orders` | Average quantity per order |
| `avg_transactions_per_order` | NUMERIC | Calculated | `items / orders` | Average line items per order |
| `avg_order_value` | NUMERIC | Calculated | `gms_gross / orders` | Average order value (**USD dollars**) |
| `gift_gms_gross` | NUMERIC | `user_mart.user_purch_daily` | `SUM(gift_gms_gross)` | Gift GMS gross (**USD dollars**) |

## Related Tables

- Other user_purch_* tables for different aggregation levels
- [user_visit_daily](user_visit_daily.md) - Visit metrics
- [user_profile](user_profile.md) - User demographics
