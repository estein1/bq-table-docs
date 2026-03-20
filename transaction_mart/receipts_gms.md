---
id: receipts_gms
title: etsy-data-warehouse-prod.transaction_mart.receipts_gms
sidebar_label: receipts_gms
---

# receipts_gms

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.receipts_gms`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: Receipt-level GMS (Gross Merchandise Sales) metrics. One row per receipt with aggregated transaction-level and receipt-level GMS calculations. **Key table for revenue analysis**.

**Primary Key**: `receipt_id`

**Partitioning**: Partitioned by DATE(creation_tsz)

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `receipt_id` | INT64 | `transaction_mart.all_receipts` | Direct from all_receipts | Unique receipt identifier. Primary key |
| `buyer_user_id` | INT64 | `transaction_mart.all_receipts` | Direct from all_receipts | Buyer's user ID |
| `seller_user_id` | INT64 | `transaction_mart.all_receipts` | Direct from all_receipts | Seller's user ID |

### Market & Location

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `market` | STRING | `transaction_mart.all_receipts` | Direct from all_receipts | Market where purchase occurred (etsy, ipp, wholesale, pattern, leo, offsite) |
| `buyer_country_name` | STRING | Derived from country_id | Country name from etsy_v2.countries | Buyer's country name |
| `seller_country_name` | STRING | Derived from seller country_id | Country name from etsy_v2.countries | Seller's country name |

### Dates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `creation_tsz` | TIMESTAMP | `transaction_mart.all_receipts` | Direct from all_receipts | Receipt creation timestamp. **Used for partitioning** |

### Transaction-Level GMS (Aggregated from Transactions)

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `trans_gms_gross` | NUMERIC | Aggregated from transactions | SUM of transaction-level GMS (listing + gift wrap) where gms_eligible=1 | Total transaction GMS gross (includes cancelled transactions) |
| `trans_gms_net` | NUMERIC | Aggregated from transactions | SUM of transaction-level GMS where gms_eligible=1 AND live=1 | Total transaction GMS net (excludes cancelled) |
| `trans_listing_gms_gross` | NUMERIC | Aggregated from transactions | SUM of transaction-level listing GMS (excludes gift wrap) where gms_eligible=1 | Listing GMS gross (no gift wrap) |
| `trans_listing_gms_net` | NUMERIC | Aggregated from transactions | SUM of transaction-level listing GMS where gms_eligible=1 AND live=1 | Listing GMS net (no gift wrap, no cancelled) |
| `trans_giftwrap_gms_gross` | NUMERIC | Aggregated from transactions | SUM of transaction-level gift wrap GMS where gms_eligible=1 | Gift wrap GMS gross |
| `trans_giftwrap_gms_net` | NUMERIC | Aggregated from transactions | SUM of transaction-level gift wrap GMS where gms_eligible=1 AND live=1 | Gift wrap GMS net (excludes cancelled) |

### Receipt-Level GMS (From Accounting)

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | Aggregated accounting GMS gross | Receipt-level GMS gross from accounting perspective |
| `gms_net` | NUMERIC | `transaction_mart.accounting_gms` | Aggregated accounting GMS net | Receipt-level GMS net from accounting perspective |
| `listing_gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | Aggregated listing GMS gross from accounting | Receipt-level listing GMS gross |
| `listing_gms_net` | NUMERIC | `transaction_mart.accounting_gms` | Aggregated listing GMS net from accounting | Receipt-level listing GMS net |
| `giftwrap_gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | Aggregated gift wrap GMS gross from accounting | Receipt-level gift wrap GMS gross |
| `giftwrap_gms_net` | NUMERIC | `transaction_mart.accounting_gms` | Aggregated gift wrap GMS net from accounting | Receipt-level gift wrap GMS net |

### Value Metrics

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gmv` | NUMERIC | Calculated from transactions | SUM of GMS + shipping for gms_eligible transactions | Gross Merchandise Value (GMS + shipping) |
| `purchase_value` | NUMERIC | Calculated from transactions | SUM of usd_subtotal_price + gift wrap for all transactions | Total purchase value (includes non-GMS-eligible) |

### Transaction Counts

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_count` | INT64 | Calculated | COUNT of transactions in receipt | Total number of transactions (line items) in receipt |
| `live_transaction_count` | INT64 | Calculated | COUNT of live transactions in receipt | Number of non-cancelled transactions |

### Type Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `has_custom` | INT64 | Aggregated from transactions | MAX(is_custom_order) across transactions | Receipt contains at least one custom order |
| `has_digital` | INT64 | Aggregated from transactions | MAX(is_digital) across transactions | Receipt contains at least one digital item |
| `is_gift` | INT64 | Aggregated from transactions | MAX(is_gift) across transactions | Receipt contains at least one gift |
| `is_post_purchase_gift` | INT64 | Aggregated from transactions | MAX(is_post_purchase_gift) across transactions | Receipt contains post-purchase gift |
| `post_purchase_gift_timestamp` | TIMESTAMP | Aggregated from transactions | MAX(post_purchase_gift_timestamp) | When post-purchase gift was added |

### Exclusion Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_test_seller` | INT64 | Aggregated from transactions | MAX(is_test_seller) across transactions | Receipt has test seller |
| `is_test_buyer` | INT64 | Aggregated from transactions | MAX(is_test_buyer) across transactions | Receipt has test buyer |
| `is_cash` | INT64 | Aggregated from transactions | MAX(is_cash) across transactions | Receipt paid with cash |
| `is_gift_card` | INT64 | Aggregated from transactions | MAX(is_gift_card) across transactions | Receipt contains gift card purchase |

### Category Analysis

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `top_category_items` | STRING | Calculated from transaction categories | Category with most items in receipt | Top category by item count |
| `top_category_gms` | STRING | Calculated from transaction categories | Category with highest GMS in receipt | Top category by GMS value |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.receipts_gms` for:**
- **Receipt-level GMS analysis** (preferred for revenue metrics)
- Aggregating orders/receipts (not individual line items)
- Revenue reporting by buyer/seller/market
- GMS vs GMV comparison
- Category mix analysis per receipt
- **Revenue dashboards and reporting**

**Don't use this table for:**
- Complete receipt analysis with visit data → Use `transaction_mart.receipt_obt`
- Transaction/line-item level analysis → Use `transaction_mart.transactions_gms`
- Post-purchase metrics → Use `transaction_mart.receipt_post_purch`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete receipt analysis, use `receipt_obt` instead (has GMS + visits + post-purchase)
- `transaction_mart.all_receipts` - Basic receipt data already here (buyer, seller, market)

**✅ Do join to these mart tables**:
- `transaction_mart.receipt_post_purch` - For returns, refunds, cases
- `transaction_mart.receipts_visits` - For visit/channel attribution
- `transaction_mart.all_receipts` - For additional receipt details (location, payment method)

### Performance Tips

1. **Partitioned by DATE(creation_tsz)** - Always filter on creation_tsz date for best performance
2. **Use trans_gms_net for revenue** - This is the standard metric (excludes cancelled)
3. **Filter test users** - `is_test_buyer = 0 AND is_test_seller = 0`
4. **GMS eligible** - trans_gms fields already exclude non-GMS-eligible (gift cards, test users, fraud, cash, high-val cancelled)
5. **trans vs receipt GMS** - trans_* fields are aggregated from transactions; gms_* fields are from accounting

### Common Query Patterns

**Daily GMS by market:**
```sql
SELECT
    DATE(creation_tsz) AS date,
    market,
    COUNT(*) AS receipt_count,
    SUM(trans_gms_net) AS total_gms_net
FROM `etsy-data-warehouse-prod.transaction_mart.receipts_gms`
WHERE DATE(creation_tsz) BETWEEN '2026-03-01' AND '2026-03-19'
  AND is_test_buyer = 0
  AND is_test_seller = 0
GROUP BY date, market
ORDER BY date, market
```

**Buyer cohort analysis:**
```sql
SELECT
    buyer_country_name,
    COUNT(DISTINCT buyer_user_id) AS unique_buyers,
    COUNT(*) AS receipt_count,
    SUM(trans_gms_net) AS total_gms,
    AVG(trans_gms_net) AS avg_order_value
FROM `etsy-data-warehouse-prod.transaction_mart.receipts_gms`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND is_test_buyer = 0
GROUP BY buyer_country_name
ORDER BY total_gms DESC
```

**Category performance:**
```sql
SELECT
    top_category_gms,
    COUNT(*) AS receipt_count,
    SUM(trans_gms_net) AS category_gms,
    AVG(transaction_count) AS avg_items_per_order
FROM `etsy-data-warehouse-prod.transaction_mart.receipts_gms`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND is_test_buyer = 0
  AND is_test_seller = 0
GROUP BY top_category_gms
ORDER BY category_gms DESC
```

**GMV vs GMS:**
```sql
SELECT
    market,
    SUM(trans_gms_net) AS gms,
    SUM(gmv) AS gmv,
    SUM(gmv) - SUM(trans_gms_net) AS shipping_net
FROM `etsy-data-warehouse-prod.transaction_mart.receipts_gms`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND is_test_buyer = 0
GROUP BY market
```

---

## Important Notes

### GMS Eligibility

Transactions are **excluded from GMS** if any of these are true:
- `is_gift_card = 1` (gift card purchases)
- `is_test_seller = 1` (test sellers)
- `is_test_buyer = 1` (test buyers)
- `is_cash = 1` (cash payments)
- `is_fraud = 1` (fraudulent transactions)
- `is_highval = 1 AND live = 0` (high-value cancelled transactions >$10K)

### trans_* vs gms_* Fields

- **trans_*** fields: Aggregated from transaction-level calculations
- **gms_*** fields: From accounting_gms table (accounting perspective)
- **Prefer trans_gms_net** for standard revenue reporting

### GMS vs GMV

- **GMS** (trans_gms_net): Merchandise value only (listing + gift wrap)
- **GMV** (gmv): GMS + shipping
- **purchase_value**: Everything including non-GMS-eligible transactions

---

## Related Tables

### Transaction Mart Tables (Join on receipt_id)
- `transaction_mart.receipt_obt` - **PREFER THIS** - Complete receipt data with GMS, visits, and post-purchase
- `transaction_mart.all_receipts` - Basic receipt data
- `transaction_mart.receipts_visits` - Visit/channel attribution
- `transaction_mart.receipt_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_gms` - Transaction-level GMS (join for line-item details)

### Other Tables
- `transaction_mart.transaction_obt` - Transaction-level OBT
- `transaction_mart.transactions_gms_by_trans` - Alternative transaction GMS view
- `transaction_mart.accounting_gms` - Accounting perspective GMS
- `transaction_mart.accounting_gms_by_trans` - Transaction accounting GMS

### Source Tables (Avoid - Data Already in Mart)
- `transaction_mart.all_receipts` - Basic fields already included
- `transaction_mart.all_transactions` - Aggregated here
- `transaction_mart.accounting_gms` - GMS fields already here
