---
id: listing_gms
title: etsy-data-warehouse-prod.listing_mart.listing_gms
sidebar_label: listing_gms
---

# listing_gms

Listing revenue (GMS) and sales metrics including total, past day, past year, and normalized daily averages.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_gms`

**Source Script:** `Rollups/auto/p2/daily/listing_mart_bucket3.sql`

**Primary Key:** `listing_id`

**Clustering:** `listing_id`

**Update Frequency:** Daily (full refresh)

**Owner:** estein@etsy.com

**Owner Team:** cdata-alerts@etsy.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `transaction_mart.all_transactions` | Primary Key | Listing id. Primary Key |
| `is_active` | INT64 | `listing_mart.listings` | Direct | 1 if listing is active, 0 if inactive |
| `gift_card_purchases` | INT64 | `transaction_mart.all_transactions` | Sum where is_gift_card = 1 | Number of gift card purchases for this listing |
| `fraud_purchases` | INT64 | `transaction_mart.all_transactions` | Sum where is_fraud = 1 | Number of fraudulent purchases |
| `high_value_purchases` | INT64 | `transaction_mart.all_transactions` | Sum where is_highval = 1 | Number of high-value purchases |
| `cash_purchases` | INT64 | `transaction_mart.all_transactions` | Sum where is_cash = 1 | Number of cash purchases |
| `digital_purchases` | INT64 | `transaction_mart.all_transactions` | Sum where is_digital = 1 | Number of digital item purchases |
| `total_gms` | NUMERIC | `transaction_mart.transactions_gms_by_trans` | Sum where live = 1 | **Total GMS (USD dollars)** for active orders. All-time |
| `total_orders` | INT64 | `transaction_mart.all_transactions` | Count distinct receipts where live = 1 | Total number of active receipts. All-time |
| `total_quantity_sold` | INT64 | `transaction_mart.all_transactions` | Sum quantity where live = 1 | Total quantity sold for active orders. All-time |
| `total_cancelled_gms` | NUMERIC | `transaction_mart.transactions_gms_by_trans` | Sum where live = 0 | **Total GMS (USD dollars)** for cancelled orders |
| `total_cancelled_orders` | INT64 | `transaction_mart.all_transactions` | Count distinct receipts where live = 0 | Total number of cancelled receipts |
| `total_quantity_cancelled` | INT64 | `transaction_mart.all_transactions` | Sum quantity where live = 0 | Total quantity for cancelled orders |
| `past_day_gms` | NUMERIC | `transaction_mart.transactions_gms_by_trans` | Sum where date = yesterday and live = 1 | GMS for yesterday (USD dollars) |
| `past_day_orders` | INT64 | `transaction_mart.all_transactions` | Count receipts from yesterday where live = 1 | Number of orders from yesterday |
| `past_day_quantity_sold` | INT64 | `transaction_mart.all_transactions` | Sum quantity from yesterday where live = 1 | Quantity sold yesterday |
| `past_year_gms` | NUMERIC | `transaction_mart.transactions_gms_by_trans` | Sum where date >= 365 days ago and live = 1 | **GMS for past 365 days (USD dollars)** |
| `past_year_orders` | INT64 | `transaction_mart.all_transactions` | Count receipts from past 365 days where live = 1 | Number of orders in past 365 days |
| `past_year_quantity_sold` | INT64 | `transaction_mart.all_transactions` | Sum quantity from past 365 days where live = 1 | Quantity sold in past 365 days |
| `purchase_count` | INT64 | `transaction_mart.all_transactions` | Count transactions where live = 1 | Number of active transactions (can be multiple per receipt) |
| `cancelled_purchase_count` | INT64 | `transaction_mart.all_transactions` | Count transactions where live = 0 | Number of cancelled transactions |
| `normalized_daily_gms` | NUMERIC | Calculated | total_gms / days_since_original_create | Average daily GMS since listing creation |
| `normalized_daily_orders` | NUMERIC | Calculated | total_orders / days_since_original_create | Average daily orders since listing creation |
| `normalized_daily_quantity` | NUMERIC | Calculated | total_quantity_sold / days_since_original_create | Average daily quantity sold since listing creation |

## Query Guidance

### When to Use This Table

- Analyzing listing revenue and sales performance
- Identifying top-selling listings
- Comparing active vs cancelled orders
- Calculating listing-level conversion metrics
- Understanding sales velocity (normalized daily metrics)

### Avoid These Joins

- ❌ Don't join to `transaction_mart.all_transactions` - sales data already aggregated here
- ❌ Don't join to `transaction_mart.transactions_gms_by_trans` - GMS already aggregated here

### Performance Tips

- **Always filter on `listing_id`** (clustered column) for best performance
- Filter on `is_active = 1` for active listings only
- Use `past_year_*` fields for recent performance analysis
- Use `normalized_daily_*` fields to compare listings of different ages

### Important Notes

**Money in USD Dollars:**
- All GMS fields are in **USD dollars** (NUMERIC type)
- No conversion needed (unlike listing_mart prices which are in cents)

**Orders vs Transactions:**
- `*_orders` = receipt count (one receipt can have multiple items)
- `purchase_count` = transaction count (line items)
- Transactions are always >= orders

**Live vs Cancelled:**
- `live = 1` means active/completed order
- `live = 0` means cancelled order
- Total metrics include all-time data

### Common Query Patterns

**Top revenue-generating listings:**
```sql
SELECT
    listing_id,
    total_gms,
    total_orders,
    total_quantity_sold,
    SAFE_DIVIDE(total_gms, total_orders) AS avg_order_value
FROM `etsy-data-warehouse-prod.listing_mart.listing_gms`
WHERE is_active = 1
ORDER BY total_gms DESC
LIMIT 100;
```

**Recent performance (past year):**
```sql
SELECT
    listing_id,
    past_year_gms,
    past_year_orders,
    past_year_quantity_sold,
    SAFE_DIVIDE(past_year_gms, past_year_orders) AS avg_order_value
FROM `etsy-data-warehouse-prod.listing_mart.listing_gms`
WHERE is_active = 1
  AND past_year_orders > 0
ORDER BY past_year_gms DESC
LIMIT 100;
```

**High velocity listings (normalized metrics):**
```sql
SELECT
    listing_id,
    normalized_daily_gms,
    normalized_daily_orders,
    normalized_daily_quantity,
    total_gms,
    total_orders
FROM `etsy-data-warehouse-prod.listing_mart.listing_gms`
WHERE is_active = 1
  AND normalized_daily_orders > 1  -- Averages > 1 order per day
ORDER BY normalized_daily_gms DESC
LIMIT 100;
```

**Cancellation analysis:**
```sql
SELECT
    listing_id,
    total_orders,
    total_cancelled_orders,
    SAFE_DIVIDE(total_cancelled_orders, total_orders + total_cancelled_orders) AS cancellation_rate,
    total_gms,
    total_cancelled_gms
FROM `etsy-data-warehouse-prod.listing_mart.listing_gms`
WHERE (total_orders + total_cancelled_orders) >= 10  -- Min 10 total orders
ORDER BY cancellation_rate DESC
LIMIT 100;
```

**Yesterday's best sellers:**
```sql
SELECT
    listing_id,
    past_day_gms,
    past_day_orders,
    past_day_quantity_sold
FROM `etsy-data-warehouse-prod.listing_mart.listing_gms`
WHERE past_day_orders > 0
ORDER BY past_day_gms DESC
LIMIT 50;
```

**Listings with special purchase types:**
```sql
SELECT
    listing_id,
    digital_purchases,
    gift_card_purchases,
    fraud_purchases,
    high_value_purchases,
    total_orders
FROM `etsy-data-warehouse-prod.listing_mart.listing_gms`
WHERE digital_purchases > 0
   OR gift_card_purchases > 0
   OR high_value_purchases > 0
LIMIT 100;
```

## Related Tables

### listing_mart Tables (Join on listing_id)

- [listings](listings.md) - Main listing data with pricing
- [listing_post_purch](listing_post_purch.md) - Post-purchase metrics (returns, reviews)
- [listing_attributes](listing_attributes.md) - Listing attributes
- All other listing_mart tables

### transaction_mart Tables (Join on listing_id)

- `transaction_mart.all_transactions` - Individual transactions
- `transaction_mart.transactions_gms_by_trans` - Transaction-level GMS
- `transaction_mart.transaction_obt` - Complete transaction data

### Source Tables (Avoid - Data Already in Mart)

- `transaction_mart.all_transactions` - Transaction data already aggregated
- `transaction_mart.transactions_gms_by_trans` - GMS already aggregated
