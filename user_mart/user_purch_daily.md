---
id: user_purch_daily
title: etsy-data-warehouse-prod.user_mart.user_purch_daily
sidebar_label: user_purch_daily
---

# user_purch_daily

Daily user purchase metrics including orders, GMS, and product counts

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_purch_daily`

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
FROM `etsy-data-warehouse-prod.user_mart.user_purch_daily`
WHERE user_id = 123456789 AND _date >= '2026-01-01';
```

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `visit_mart.visits_transactions` | Primary Key | User ID |
| `mapped_user_id` | INT64 | `user_mart.user_mapping` | Joined via user_id | Mapped user ID for cross-device tracking |
| `is_guest` | INT64 | `user_mart.user_mapping` | Direct | 1 if guest user, 0 if registered |
| `purch_date` | INT64 | `visit_mart.visits_transactions` | Unix timestamp | Purchase date (unix format) |
| `_date` | DATE | `visit_mart.visits_transactions` | `DATE(TIMESTAMP_SECONDS(purch_date))` | Purchase date (date format) |
| **Order Counts by Platform/OS** | | | | |
| `desktop_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_os = 'desktop' THEN 1 END)` | Desktop orders |
| `ios_os_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_os = 'ios' THEN 1 END)` | iOS orders |
| `android_os_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_os = 'android' THEN 1 END)` | Android orders |
| `other_os_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_os = 'other' THEN 1 END)` | Other OS orders |
| `undefined_os_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_os = 'undefined' THEN 1 END)` | Undefined OS orders |
| **Order Counts by App** | | | | |
| `mw_app_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_app = 'mobile_web' THEN 1 END)` | Mobile web orders |
| `soe_app_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_app = 'soe' THEN 1 END)` | Sell on Etsy app orders |
| `boe_app_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_app = 'boe' THEN 1 END)` | Buy on Etsy app orders |
| `undefined_app_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_app = 'undefined' THEN 1 END)` | Undefined app orders |
| **Order Counts by Device** | | | | |
| `tablet_device_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_device = 'tablet' THEN 1 END)` | Tablet orders |
| `handset_device_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_device = 'handset' THEN 1 END)` | Handset orders |
| `undefined_device_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN platform_device = 'undefined' THEN 1 END)` | Undefined device orders |
| **GMS by Platform/OS** | | | | **USD dollars** |
| `desktop_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_os = 'desktop' THEN gms_gross END)` | Desktop GMS (**USD dollars**) |
| `ios_os_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_os = 'ios' THEN gms_gross END)` | iOS GMS (**USD dollars**) |
| `android_os_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_os = 'android' THEN gms_gross END)` | Android GMS (**USD dollars**) |
| `other_os_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_os = 'other' THEN gms_gross END)` | Other OS GMS (**USD dollars**) |
| `undefined_os_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_os = 'undefined' THEN gms_gross END)` | Undefined OS GMS (**USD dollars**) |
| **GMS by App** | | | | **USD dollars** |
| `mw_app_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_app = 'mobile_web' THEN gms_gross END)` | Mobile web GMS (**USD dollars**) |
| `soe_app_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_app = 'soe' THEN gms_gross END)` | SOE app GMS (**USD dollars**) |
| `boe_app_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_app = 'boe' THEN gms_gross END)` | BOE app GMS (**USD dollars**) |
| `undefined_app_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_app = 'undefined' THEN gms_gross END)` | Undefined app GMS (**USD dollars**) |
| **GMS by Device** | | | | **USD dollars** |
| `tablet_device_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_device = 'tablet' THEN gms_gross END)` | Tablet GMS (**USD dollars**) |
| `handset_device_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_device = 'handset' THEN gms_gross END)` | Handset GMS (**USD dollars**) |
| `undefined_device_gms` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN platform_device = 'undefined' THEN gms_gross END)` | Undefined device GMS (**USD dollars**) |
| **Item Counts** | | | | |
| `digital_items` | INT64 | `visit_mart.visits_transactions` | `SUM(is_digital)` | Digital items purchased |
| `custom_items` | INT64 | `visit_mart.visits_transactions` | `SUM(is_custom_order)` | Custom items purchased |
| `gift_items` | INT64 | `visit_mart.visits_transactions` | `SUM(is_gift)` | Gift items purchased |
| `vintage_items` | INT64 | `visit_mart.visits_transactions` | `SUM(is_vintage)` | Vintage items purchased |
| `handmade_items` | INT64 | `visit_mart.visits_transactions` | `SUM(is_handmade)` | Handmade items purchased |
| `supplies_items` | INT64 | `visit_mart.visits_transactions` | `SUM(is_supplies)` | Supplies items purchased |
| `leo_items` | INT64 | `visit_mart.visits_transactions` | `SUM(CASE WHEN market = 'leo' THEN 1 END)` | Leo items purchased |
| **Core Order Metrics** | | | | |
| `orders` | INT64 | Temp: `user_purch_receipts` | `COUNT(*)` | Total orders (receipts) |
| `guest_checkout_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(is_guest_checkout)` | Guest checkout orders |
| `cancelled_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(1 - receipt_live)` | Cancelled orders |
| `non_domestic_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN is_non_domestic = 1 THEN 1 END)` | Non-domestic orders |
| `dc_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN is_dc = 1 THEN 1 END)` | Direct checkout orders |
| `pattern_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN market = 'pattern' THEN 1 END)` | Pattern orders |
| `leo_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN market = 'leo' THEN 1 END)` | Leo orders |
| `gift_card_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN is_gift_card = 1 THEN 1 END)` | Gift card orders |
| `ipp_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN market = 'ipp' THEN 1 END)` | In-person orders |
| `wholesale_orders` | INT64 | Temp: `user_purch_receipts` | `SUM(CASE WHEN market = 'wholesale' THEN 1 END)` | Wholesale orders |
| **Market-Specific GMS** | | | | **USD dollars** |
| `non_domestic_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN is_non_domestic = 1 THEN gms_gross END)` | Non-domestic GMS gross (**USD dollars**) |
| `dc_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN is_dc = 1 THEN gms_gross END)` | Direct checkout GMS gross (**USD dollars**) |
| `pattern_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN market = 'pattern' THEN gms_gross END)` | Pattern GMS gross (**USD dollars**) |
| `leo_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN market = 'leo' THEN gms_gross END)` | Leo GMS gross (**USD dollars**) |
| `leo_gms_net` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN market = 'leo' THEN gms_net END)` | Leo GMS net (**USD dollars**) |
| `gift_card_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN is_gift_card = 1 THEN gms_gross END)` | Gift card GMS gross (**USD dollars**) |
| `ipp_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN market = 'ipp' THEN gms_gross END)` | In-person GMS gross (**USD dollars**) |
| `wholesale_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN market = 'wholesale' THEN gms_gross END)` | Wholesale GMS gross (**USD dollars**) |
| `guest_checkout_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN is_guest_checkout = 1 THEN gms_gross END)` | Guest checkout GMS gross (**USD dollars**) |
| **Item & Quantity Metrics** | | | | |
| `quantity_items` | INT64 | `visit_mart.visits_transactions` | `SUM(quantity)` | Total quantity of items |
| `items` | INT64 | `visit_mart.visits_transactions` | `COUNT(*)` | Total transaction count (line items) |
| `cancelled_items` | INT64 | `visit_mart.visits_transactions` | `SUM(1 - transaction_live)` | Cancelled items |
| **Total GMS** | | | | **USD dollars** |
| `trans_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(trans_gms_gross)` | Transaction-based GMS gross (**USD dollars**) |
| `trans_gms_net` | NUMERIC | `visit_mart.visits_transactions` | `SUM(trans_gms_net)` | Transaction-based GMS net (**USD dollars**) |
| `gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(gms_gross)` | GMS gross (**USD dollars**) |
| `gms_net` | NUMERIC | `visit_mart.visits_transactions` | `SUM(gms_net)` | GMS net (**USD dollars**) |
| **Calculated Metrics** | | | | |
| `avg_quantity_per_order` | NUMERIC | Calculated | `quantity_items / orders` | Average quantity per order |
| `avg_transactions_per_order` | NUMERIC | Calculated | `items / orders` | Average line items per order |
| `avg_order_value` | NUMERIC | Calculated | `gms_gross / orders` | Average order value (**USD dollars**) |
| `gift_gms_gross` | NUMERIC | `visit_mart.visits_transactions` | `SUM(CASE WHEN is_gift = 1 THEN gms_gross END)` | Gift GMS gross (**USD dollars**) |

## Related Tables

- Other user_purch_* tables for different aggregation levels
- [user_visit_daily](user_visit_daily.md) - Visit metrics
- [user_profile](user_profile.md) - User demographics
