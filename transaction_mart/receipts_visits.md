---
id: receipts_visits
title: etsy-data-warehouse-prod.transaction_mart.receipts_visits
sidebar_label: receipts_visits
---

# receipts_visits

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.receipts_visits`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: Visit attribution data for each receipt. Aggregated from transactions_visits using MIN() to select one visit per receipt. **Simplified receipt-level visit attribution**.

**Primary Key**: `receipt_id`

**Partitioning**: Partitioned by `_date` (visit date)

**Update Frequency**: Daily (after weblog.visits is loaded)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `receipt_id` | INT64 | `transaction_mart.transactions_visits` | Aggregated from transactions_visits | Unique receipt identifier. Primary key |
| `order_id` | INT64 | `transaction_mart.transactions_visits` | MIN(order_id) from transactions in receipt | Order ID (different from receipt_id) |
| `user_id` | INT64 | `transaction_mart.transactions_visits` | MIN(user_id) from transactions in receipt | Buyer's user ID |

### Dates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `date` | DATE | `transaction_mart.transactions_visits` | MIN(date) from transactions in receipt | Earliest transaction date in receipt |
| `_date` | DATE | `transaction_mart.transactions_visits` | MIN(_date) from transactions in receipt | Earliest visit date. **Used for partitioning**. NULL if no visit match |
| `run_date` | INT64 | `transaction_mart.transactions_visits` | MIN(run_date) from transactions in receipt | Earliest visit end date (integer format) |
| `start_datetime` | TIMESTAMP | `transaction_mart.transactions_visits` | MIN(start_datetime) from transactions in receipt | Earliest visit start datetime. NULL if no visit match |

### Visit Attribution

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `visit_id` | STRING | `transaction_mart.transactions_visits` | MIN(visit_id) from transactions in receipt | Visit ID that resulted in this receipt. NULL if no visit match |
| `visit_gms` | NUMERIC | `transaction_mart.transactions_visits` | MIN(visit_gms) from transactions in receipt | Total buyer GMS in the visit (not just this receipt). NULL if no visit match |
| `visit_market` | STRING | `transaction_mart.transactions_visits` | MIN(visit_market) from transactions in receipt | Visit market (etsy/pattern/leo). NULL if no visit match |

### Platform

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `platform_os` | STRING | `transaction_mart.transactions_visits` | MIN(platform_os) from transactions in receipt | Platform OS. NULL if no visit match |
| `platform_app` | STRING | `transaction_mart.transactions_visits` | MIN(platform_app) from transactions in receipt | Platform app. NULL if no visit match |
| `platform_device` | STRING | `transaction_mart.transactions_visits` | MIN(platform_device) from transactions in receipt | Platform device type. NULL if no visit match |
| `mapped_platform_type` | STRING | `transaction_mart.transactions_visits` | MIN(mapped_platform_type) from transactions in receipt | Mapped platform type. NULL if no visit match |

### Channel & Referrer

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `top_channel` | STRING | `transaction_mart.transactions_visits` | MIN(top_channel) from transactions in receipt | Top-level marketing channel. NULL if no visit match |
| `second_channel` | STRING | `transaction_mart.transactions_visits` | MIN(second_channel) from transactions in receipt | Second-level marketing channel. NULL if no visit match |
| `third_channel` | STRING | `transaction_mart.transactions_visits` | MIN(third_channel) from transactions in receipt | Third-level marketing channel. NULL if no visit match |
| `referrer_type` | STRING | `transaction_mart.transactions_visits` | MIN(referrer_type) from transactions in receipt | Referrer type. NULL if no visit match |
| `referring_domain` | STRING | `transaction_mart.transactions_visits` | MIN(referring_domain) from transactions in receipt | Referring domain. NULL if no visit match |

### Geography & Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `canonical_region` | STRING | `transaction_mart.transactions_visits` | MIN(canonical_region) from transactions in receipt | Visit region. NULL if no visit match |
| `is_preliminary` | INT64 | `transaction_mart.transactions_visits` | MIN(is_preliminary) from transactions in receipt | Is preliminary flag (deprecated) |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.receipts_visits` for:**
- **Receipt-level visit attribution** - One visit per receipt
- Marketing channel analysis per order/receipt
- Platform/device analysis at receipt level
- **Simpler than transactions_visits** - One row per receipt instead of per transaction

**Don't use this table for:**
- Complete receipt analysis → Use `transaction_mart.receipt_obt` (has visit data pre-joined)
- Transaction-level attribution → Use `transaction_mart.transactions_visits`
- Visit-level analysis → Use `visit_mart.visits`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `receipt_obt` instead (visit data already joined)
- `transaction_mart.transactions_visits` - This table IS that data aggregated
- `weblog.order_attribution` - Already incorporated in transactions_visits
- `visit_mart.visits` - Visit fields already here

**✅ Do join to these mart tables**:
- `transaction_mart.all_receipts` - Receipt details (join via receipt_id)
- `transaction_mart.receipts_gms` - GMS data
- `transaction_mart.receipt_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - If you need transaction-level visit data

### Performance Tips

1. **Partitioned by `_date`** - Always filter on _date (visit date) for best performance when possible
2. **NULL visit fields** - Receipts without matching visits have NULL for all visit-related fields
3. **Use receipt_obt for most queries** - This table is for specific attribution analysis; receipt_obt has visit data pre-joined
4. **Aggregated with MIN()** - All fields use MIN() to pick one value per receipt
5. **One row per receipt** - Much smaller than transactions_visits (which has one row per transaction)

### Common Query Patterns

**Get visit attribution for receipts:**
```sql
SELECT
    receipt_id,
    date AS receipt_date,
    visit_id,
    _date AS visit_date,
    top_channel,
    second_channel,
    platform_device,
    referring_domain
FROM `etsy-data-warehouse-prod.transaction_mart.receipts_visits`
WHERE receipt_id IN (1234, 5678)
```

**Channel performance by receipt:**
```sql
SELECT
    v.top_channel,
    v.second_channel,
    COUNT(*) AS receipt_count,
    COUNT(DISTINCT v.user_id) AS unique_buyers,
    SUM(rg.trans_gms_net) AS total_gms,
    AVG(rg.trans_gms_net) AS avg_order_value
FROM `etsy-data-warehouse-prod.transaction_mart.receipts_visits` v
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.receipts_gms` rg
    ON v.receipt_id = rg.receipt_id
WHERE v.date = '2026-03-19'
  AND v.top_channel IS NOT NULL
  AND rg.is_test_buyer = 0
GROUP BY v.top_channel, v.second_channel
ORDER BY total_gms DESC
LIMIT 20
```

**Platform analysis:**
```sql
SELECT
    v.platform_device,
    v.platform_app,
    COUNT(*) AS receipt_count,
    SUM(rg.trans_gms_net) AS total_gms,
    AVG(rg.trans_gms_net) AS avg_order_value,
    AVG(rg.transaction_count) AS avg_items_per_order
FROM `etsy-data-warehouse-prod.transaction_mart.receipts_visits` v
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.receipts_gms` rg
    ON v.receipt_id = rg.receipt_id
WHERE v.date = '2026-03-19'
  AND v.platform_device IS NOT NULL
  AND rg.is_test_buyer = 0
GROUP BY v.platform_device, v.platform_app
ORDER BY total_gms DESC
```

**Attribution rate:**
```sql
SELECT
    DATE(r.creation_tsz) AS order_date,
    COUNT(*) AS total_receipts,
    SUM(CASE WHEN v.visit_id IS NOT NULL THEN 1 ELSE 0 END) AS attributed_receipts,
    SUM(CASE WHEN v.visit_id IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) AS attribution_rate
FROM `etsy-data-warehouse-prod.transaction_mart.all_receipts` r
LEFT JOIN `etsy-data-warehouse-prod.transaction_mart.receipts_visits` v
    ON r.receipt_id = v.receipt_id
WHERE DATE(r.creation_tsz) = '2026-03-19'
  AND r.receipt_live = 1
GROUP BY order_date
```

---

## Important Notes

### Aggregation with MIN()

This table aggregates from `transactions_visits` using **MIN()** for all fields:
- When a receipt has multiple transactions, MIN() selects the "first" value
- For strings, MIN() is alphabetical
- For numbers/dates, MIN() is smallest/earliest
- This means you get the **earliest visit** associated with the receipt

### Why MIN() Instead of First Transaction?

The aggregation uses MIN() rather than selecting values from the first transaction because:
- Simpler logic (no need to define "first")
- Consistent behavior across all fields
- For most fields (visit_id, channels, platform), all transactions in a receipt should have the same values

If transactions in a receipt have different visits, MIN() picks the alphabetically/numerically first one.

### NULL Visit Fields

**Receipts without matching visits have NULL for all visit-related fields**:
- This is expected for some receipt types (e.g., guest checkout, API orders, in-person)
- When visit_id is NULL, all other visit fields are also NULL
- run_date defaults to transaction date (unix seconds) when no visit

To analyze only receipts with visit attribution, filter WHERE visit_id IS NOT NULL.

### Visit GMS vs Receipt GMS

`visit_gms` is the **total buyer GMS in the visit**, not just this receipt:
- A buyer may purchase from multiple shops in a single visit (multishop checkout)
- visit_gms is the sum of all receipts in that visit for that buyer
- To get receipt-level GMS, join to receipts_gms

### One Visit Per Receipt

Unlike transactions (which can theoretically have different visits per transaction in a receipt), this table forces **one visit per receipt**:
- This is the expected behavior - all transactions in a receipt should come from the same visit
- If there are discrepancies, MIN() resolves them

---

## Related Tables

### Transaction Mart Tables (Join on receipt_id)
- `transaction_mart.receipt_obt` - **PREFER THIS** - Complete receipt data with visits pre-joined
- `transaction_mart.all_receipts` - Core receipt data
- `transaction_mart.receipts_gms` - GMS metrics
- `transaction_mart.receipt_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - **Transaction-level** visit attribution (more granular)

### Transaction Tables (Join via receipt_id)
- `transaction_mart.all_transactions` - Transaction details
- `transaction_mart.transactions_gms_by_trans` - Transaction GMS

### Visit Tables
- `visit_mart.visits` - **Visit-level data** - For visit-centric analysis
- `weblog.order_attribution` - Attribution mapping (incorporated in transactions_visits)

### Source Tables (Avoid - Data Already in Mart)
- `transaction_mart.transactions_visits` - This table IS that data aggregated by receipt_id
