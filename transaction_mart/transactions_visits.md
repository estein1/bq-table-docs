---
id: transactions_visits
title: etsy-data-warehouse-prod.transaction_mart.transactions_visits
sidebar_label: transactions_visits
---

# transactions_visits

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.transactions_visits`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: Visit attribution data for each transaction. Associates transactions with the visit that led to the purchase, including platform, channel, and referrer information.

**Primary Key**: `transaction_id`

**Partitioning**: Partitioned by `_date` (visit date)

**Update Frequency**: Daily (after weblog.visits is loaded)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Unique transaction identifier. Primary key |
| `receipt_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Parent receipt ID |
| `order_id` | INT64 | `transaction_mart.all_receipts` | Direct from all_receipts | Order ID (different from receipt_id) |
| `user_id` | INT64 | `transaction_mart.all_transactions` | buyer_user_id from all_transactions | Buyer's user ID |

### Dates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `date` | DATE | `transaction_mart.all_transactions` | DATE(creation_tsz) from transaction | Transaction date |
| `_date` | DATE | `visit_mart.visits` | Visit date from visits table | Visit date. **Used for partitioning**. NULL if no visit match |
| `run_date` | INT64 | `visit_mart.visits` | Visit end date in integer format (unix seconds). Transaction date if no visit | Date visit ended (integer format) |
| `start_datetime` | TIMESTAMP | `visit_mart.visits` | Visit start timestamp from visits table | Visit start datetime. NULL if no visit match |

### Visit Attribution

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `visit_id` | STRING | `weblog.order_attribution`, `visit_mart.visits` | From order_attribution table, matched to visits | Visit ID that resulted in this transaction. NULL if no visit match |
| `visit_gms` | NUMERIC | `visit_mart.visits` | total_gms from visits table | Total buyer GMS in this visit (not just this transaction). NULL if no visit match |
| `visit_market` | STRING | `visit_mart.visits` | From visits table | Visit market (etsy/pattern/leo). NULL if no visit match |

### Platform

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `platform_os` | STRING | `visit_mart.visits` | From visit mart mapping | Platform OS (iOS, Android, Windows, Mac, etc.). NULL if no visit match |
| `platform_app` | STRING | `visit_mart.visits` | From visit mart mapping | Platform app (BOE, SOE, Desktop, etc.). NULL if no visit match |
| `platform_device` | STRING | `visit_mart.visits` | From visit mart mapping | Platform device type (mobile, desktop, tablet). NULL if no visit match |
| `mapped_platform_type` | STRING | `visit_mart.visits` | From visit mart mapping | Mapped platform type. NULL if no visit match |

### Channel & Referrer

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `top_channel` | STRING | `visit_mart.visits` | From visits table | Top-level marketing channel. NULL if no visit match |
| `second_channel` | STRING | `visit_mart.visits` | From visits table | Second-level marketing channel. NULL if no visit match |
| `third_channel` | STRING | `visit_mart.visits` | From visits table | Third-level marketing channel. NULL if no visit match |
| `referrer_type` | STRING | `visit_mart.visits` | From visits table | Referrer type (internal, external, direct, etc.). NULL if no visit match |
| `referring_domain` | STRING | `visit_mart.visits` | From visits table | Referring domain. NULL if no visit match |

### Geography & Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `canonical_region` | STRING | `visit_mart.visits` | From visits table | Visit region. NULL if no visit match |
| `is_preliminary` | INT64 | `visit_mart.visits` | From visits table. 0 if no visit match | Is preliminary flag (deprecated). Returns 0 if no visit |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.transactions_visits` for:**
- **Transaction-level visit attribution** - Link transactions to visits
- Marketing channel analysis per transaction
- Platform/device analysis
- **Customer journey analysis** - Understand path to purchase

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt` (has visit data pre-joined)
- Receipt-level visit attribution → Use `transaction_mart.receipts_visits`
- Visit-level analysis → Use `visit_mart.visits`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `transaction_obt` instead (visit data already joined)
- `weblog.order_attribution` - Already incorporated here
- `visit_mart.visits` - Visit fields already here

**✅ Do join to these mart tables**:
- `transaction_mart.all_transactions` - Transaction details (join via transaction_id)
- `transaction_mart.transactions_gms_by_trans` - GMS data
- `transaction_mart.transactions_buyer` - Buyer demographics
- `transaction_mart.transaction_post_purch` - Post-purchase metrics

### Performance Tips

1. **Partitioned by `_date`** - Always filter on _date (visit date) for best performance when possible
2. **NULL visit fields** - Transactions without matching visits have NULL for all visit-related fields
3. **Use transaction_obt for most queries** - This table is for specific attribution analysis; transaction_obt has visit data pre-joined
4. **Deduplication** - Visit deduped by visit_id + transaction_id, ordered by start_datetime (earliest visit wins)
5. **Converted visits only** - Only visits with converted=1 are matched to transactions

### Common Query Patterns

**Get visit attribution for transactions:**
```sql
SELECT
    transaction_id,
    date AS transaction_date,
    visit_id,
    _date AS visit_date,
    top_channel,
    second_channel,
    platform_device,
    referring_domain
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_visits`
WHERE transaction_id IN (1234, 5678)
```

**Channel performance analysis:**
```sql
SELECT
    v.top_channel,
    v.second_channel,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT v.user_id) AS unique_buyers,
    SUM(tg.trans_gms_net) AS total_gms,
    AVG(tg.trans_gms_net) AS avg_gms
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_visits` v
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans` tg
    ON v.transaction_id = tg.transaction_id
WHERE v.date = '2026-03-19'
  AND v.top_channel IS NOT NULL
  AND tg.gms_eligible = 1
GROUP BY v.top_channel, v.second_channel
ORDER BY total_gms DESC
LIMIT 20
```

**Platform breakdown:**
```sql
SELECT
    v.platform_device,
    v.platform_app,
    COUNT(*) AS transaction_count,
    SUM(tg.trans_gms_net) AS total_gms,
    AVG(tg.trans_gms_net) AS avg_gms
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_visits` v
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans` tg
    ON v.transaction_id = tg.transaction_id
WHERE v.date = '2026-03-19'
  AND v.platform_device IS NOT NULL
  AND tg.gms_eligible = 1
GROUP BY v.platform_device, v.platform_app
ORDER BY total_gms DESC
```

**Transactions without visit attribution:**
```sql
SELECT
    COUNT(*) AS transactions_without_visits,
    SUM(tg.trans_gms_net) AS unattributed_gms,
    AVG(tg.trans_gms_net) AS avg_gms
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_visits` v
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans` tg
    ON v.transaction_id = tg.transaction_id
WHERE v.date = '2026-03-19'
  AND v.visit_id IS NULL
  AND tg.gms_eligible = 1
```

---

## Important Notes

### Visit Matching Logic

Transactions are matched to visits via `weblog.order_attribution`:
1. Order ID from receipt matches order_attribution
2. Visit ID from order_attribution matches visit_mart.visits
3. Only converted visits (converted=1) are included
4. If multiple visits match, earliest start_datetime wins

### NULL Visit Fields

**Transactions without matching visits have NULL for all visit-related fields**:
- This is expected for some transaction types (e.g., guest checkout, API orders, in-person)
- When visit_id is NULL, all other visit fields are also NULL
- run_date defaults to transaction date (unix seconds) when no visit

To analyze only transactions with visit attribution, filter WHERE visit_id IS NOT NULL.

### Visit GMS vs Transaction GMS

`visit_gms` is the **total buyer GMS in the visit**, not just this transaction:
- A buyer may purchase multiple items in a single visit
- visit_gms is the sum of all transactions in that visit for that buyer
- To get transaction-level GMS, join to transactions_gms_by_trans

### Channel Hierarchy

Channels are hierarchical:
- `top_channel`: Highest level (e.g., 'Paid', 'Organic', 'Direct')
- `second_channel`: Second level (e.g., 'SEM', 'SEO', 'Email')
- `third_channel`: Most specific level (e.g., 'Google Shopping', 'Brand Search')

Not all transactions have all three levels populated.

### Platform Mapping

Platform fields come from visit_mart's standardized mapping:
- `platform_device`: mobile, desktop, tablet
- `platform_app`: BOE (Buyer on Etsy app), SOE (Seller on Etsy app), Desktop
- `platform_os`: iOS, Android, Windows, Mac, etc.
- `mapped_platform_type`: Combination of device and app

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transaction_obt` - **PREFER THIS** - Complete transaction data with visits pre-joined
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transactions_gms_by_trans` - GMS metrics
- `transaction_mart.transactions_buyer` - Buyer demographics
- `transaction_mart.transaction_post_purch` - Post-purchase metrics

### Receipt Tables (Join via receipt_id)
- `transaction_mart.receipts_visits` - **Receipt-level visit attribution** (aggregated from this table)
- `transaction_mart.all_receipts` - Receipt details
- `transaction_mart.receipts_gms` - Receipt GMS
- `transaction_mart.receipt_obt` - Receipt-level OBT

### Visit Tables
- `visit_mart.visits` - **Visit-level data** - For visit-centric analysis
- `weblog.order_attribution` - Attribution mapping (already incorporated)

### Source Tables (Avoid - Data Already in Mart)
- `weblog.order_attribution` - Order to visit mapping (already incorporated)
- `visit_mart.visits` - Visit details (key fields already here)
