---
id: transactions_gms_by_trans
title: etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans
sidebar_label: transactions_gms_by_trans
---

# transactions_gms_by_trans

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: Transaction-level **operational GMS** (calculated from transaction prices) with buyer/seller demographics and category data. **Alternative to transactions_gms with operational GMS instead of accounting GMS**.

**Primary Key**: `transaction_id`

**Clustering**: Not specified

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers & Dates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Unique transaction identifier. Primary key |
| `date` | DATE | `transaction_mart.all_transactions` | DATE(creation_tsz) from all_transactions | Transaction date. **Use for date filtering** |
| `market` | STRING | `transaction_mart.all_transactions` | Direct from all_transactions | Market where purchase occurred (etsy, ipp, wholesale, pattern, leo, offsite) |
| `receipt_id` | INT64 | `transaction_mart.transactions_buyer` | From transactions_buyer | Parent receipt ID |

### Buyer Information

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `buyer_user_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Buyer's user ID |
| `mapped_user_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Canonical buyer ID (handles merged accounts) |
| `buyer_country_id` | INT64 | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's country ID. **PII** |
| `buyer_country` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's country code. **PII** |
| `buyer_country_name` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's country name. **PII** |
| `buyer_state` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's state/province. **PII** |
| `buyer_city` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's city. **PII** |
| `buyer_zip` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's postal code. **PII** |
| `buyer_currency` | STRING | `transaction_mart.all_transactions` | Direct from all_transactions | Currency buyer paid in |

### Seller Information

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `seller_user_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Seller's user ID |
| `seller_currency` | STRING | `transaction_mart.all_transactions` | Direct from all_transactions | Currency seller receives |
| `seller_exch_rt` | NUMERIC | `transaction_mart.all_transactions` | Calculated exchange rate from seller currency to USD | Exchange rate from seller currency to USD |
| `seller_country_id` | INT64 | `transaction_mart.transactions_seller` | From transactions_seller | Seller's country ID at transaction time. **PII** |
| `seller_country` | STRING | `transaction_mart.transactions_seller` | From transactions_seller | Seller's country code at transaction time. **PII** |
| `seller_country_name` | STRING | `transaction_mart.transactions_seller` | From transactions_seller | Seller's country name at transaction time. **PII** |
| `seller_state` | STRING | `transaction_mart.transactions_seller` | From transactions_seller | Seller's state at transaction time. **PII** |
| `seller_zip` | STRING | `transaction_mart.transactions_seller` | From transactions_seller | Seller's postal code at transaction time. **PII** |

### Category & Payment

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `new_category` | STRING | `transaction_mart.all_transactions_categories` | From all_transactions_categories | Top-level category name |
| `payment_method` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Payment method ('dc', 'pp', 'other') |
| `is_dc` | INT64 | Derived from payment_method | 1 if payment_method='dc', else 0 | Direct checkout (Etsy Payments) payment |

### Transaction Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_gift_card` | INT64 | `transaction_mart.all_transactions` | From all_transactions | Transaction is gift card purchase |
| `is_test_seller` | INT64 | `transaction_mart.all_transactions` | From all_transactions | Seller is test user |
| `is_test_buyer` | INT64 | `transaction_mart.all_transactions` | From all_transactions | Buyer is test user |
| `is_cash` | INT64 | `transaction_mart.all_transactions` | From all_transactions | Payment was cash |
| `is_guest_checkout` | INT64 | `transaction_mart.all_transactions` | From all_transactions | Buyer used guest checkout |
| `transaction_live` | INT64 | `transaction_mart.all_transactions` | live field from all_transactions | Transaction is live (not cancelled) |
| `quantity` | INT64 | `transaction_mart.all_transactions` | From all_transactions | Quantity purchased |

### GMS Eligibility

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gms_eligible` | INT64 | Calculated | 1 if all flags are 0: is_gift_card, is_test_seller, is_test_buyer, is_cash, is_fraud. Also excludes high-value cancelled (>$10K) | Transaction is eligible for GMS reporting |

### Operational GMS (Calculated from Transaction Prices)

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `trans_gms_gross` | NUMERIC | Calculated | If gms_eligible=1: usd_subtotal_price + gift_wrap_price. Else 0 | Transaction GMS gross (listing + gift wrap). **In USD dollars** |
| `trans_gms_net` | NUMERIC | Calculated | If gms_eligible=1 AND live=1: usd_subtotal_price + gift_wrap_price. Else 0 | Transaction GMS net (excludes cancelled). **In USD dollars** |
| `trans_listing_gms_gross` | NUMERIC | Calculated | If gms_eligible=1: usd_subtotal_price. Else 0 | Listing GMS gross (no gift wrap). **In USD dollars** |
| `trans_listing_gms_net` | NUMERIC | Calculated | If gms_eligible=1 AND live=1: usd_subtotal_price. Else 0 | Listing GMS net (no gift wrap, excludes cancelled). **In USD dollars** |
| `trans_giftwrap_gms_gross` | NUMERIC | Calculated | If gms_eligible=1: gift_wrap_price. Else 0 | Gift wrap GMS gross. **In USD dollars** |
| `trans_giftwrap_gms_net` | NUMERIC | Calculated | If gms_eligible=1 AND live=1: gift_wrap_price. Else 0 | Gift wrap GMS net (excludes cancelled). **In USD dollars** |
| `gmv` | NUMERIC | Calculated | If gms_eligible=1: usd_subtotal_price + gift_wrap_price + shipping_price. Else 0 | Gross Merchandise Value (GMS + shipping). **In USD dollars** |
| `purchase_value` | NUMERIC | Calculated | usd_subtotal_price + gift_wrap_price | Total purchase value (includes non-GMS-eligible). **In USD dollars** |

### Accounting GMS (from Accounting System)

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gms_gross` | NUMERIC | `transaction_mart.accounting_gms_by_trans` | Accounting GMS gross. Returns 0 if test seller | Total accounting GMS gross. **In USD dollars** |
| `gms_net` | NUMERIC | `transaction_mart.accounting_gms_by_trans` | Accounting GMS net. Returns 0 if test seller | Total accounting GMS net. **In USD dollars** |
| `listing_gms_gross` | NUMERIC | `transaction_mart.accounting_gms_by_trans` | Accounting listing GMS gross. Returns 0 if test seller | Listing GMS gross from accounting. **In USD dollars** |
| `listing_gms_net` | NUMERIC | `transaction_mart.accounting_gms_by_trans` | Accounting listing GMS net. Returns 0 if test seller | Listing GMS net from accounting. **In USD dollars** |
| `giftwrap_gms_gross` | NUMERIC | `transaction_mart.accounting_gms_by_trans` | Accounting gift wrap GMS gross. Returns 0 if test seller | Gift wrap GMS gross from accounting. **In USD dollars** |
| `giftwrap_gms_net` | NUMERIC | `transaction_mart.accounting_gms_by_trans` | Accounting gift wrap GMS net. Returns 0 if test seller | Gift wrap GMS net from accounting. **In USD dollars** |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.transactions_gms_by_trans` for:**
- **Operational GMS analysis** - Calculated directly from transaction prices
- Comparing operational vs accounting GMS (has both trans_* and acctg *)
- Transaction-level revenue analysis with demographics
- **Preferred for operational reporting** (vs transactions_gms which has accounting GMS in main columns)

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt` (has all fields pre-joined)
- Receipt-level GMS → Use `transaction_mart.receipts_gms`
- Financial/accounting reporting → Use `transaction_mart.transactions_gms` or `accounting_gms`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `transaction_obt` instead (all data pre-joined)
- `transaction_mart.transactions_buyer` - Already joined here
- `transaction_mart.transactions_seller` - Already joined here
- `transaction_mart.all_transactions_categories` - Already joined here
- `transaction_mart.accounting_gms_by_trans` - Already joined here

**✅ Do join to these mart tables** if not using OBT:
- `transaction_mart.all_transactions` - Additional transaction fields not included here
- `transaction_mart.transaction_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - Visit attribution

### Performance Tips

1. **Filter on `date`** - Use for date-based queries (transaction date)
2. **GMS eligibility already applied** - gms_eligible field pre-calculated
3. **Test users** - Filter with `is_test_buyer = 0 AND is_test_seller = 0`
4. **Already in dollars** - All GMS fields are NUMERIC in USD dollars (not cents)
5. **Use trans_* for operational** - trans_gms_net is operational GMS; gms_net is accounting GMS
6. **Use transaction_obt for most queries** - This table is for specific use cases

### Common Query Patterns

**Daily operational GMS:**
```sql
SELECT
    date,
    market,
    COUNT(*) AS transaction_count,
    SUM(trans_gms_net) AS operational_gms,
    SUM(gms_net) AS accounting_gms,
    SUM(trans_gms_net - gms_net) AS variance
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans`
WHERE date BETWEEN '2026-03-01' AND '2026-03-19'
  AND gms_eligible = 1
  AND is_test_buyer = 0
  AND is_test_seller = 0
GROUP BY date, market
ORDER BY date, market
```

**Operational GMS by category:**
```sql
SELECT
    new_category,
    COUNT(*) AS transaction_count,
    SUM(trans_gms_net) AS category_gms,
    AVG(trans_gms_net) AS avg_gms,
    SUM(gmv) AS category_gmv
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans`
WHERE date = '2026-03-19'
  AND gms_eligible = 1
  AND is_test_buyer = 0
GROUP BY new_category
ORDER BY category_gms DESC
LIMIT 20
```

**GMV vs GMS:**
```sql
SELECT
    market,
    SUM(trans_gms_net) AS gms,
    SUM(gmv) AS gmv,
    SUM(gmv) - SUM(trans_gms_net) AS shipping_net,
    SUM(purchase_value) AS total_purchase_value
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans`
WHERE date = '2026-03-19'
  AND is_test_buyer = 0
GROUP BY market
ORDER BY gms DESC
```

**Compare operational vs accounting GMS:**
```sql
SELECT
    date,
    COUNT(*) AS transaction_count,
    SUM(trans_gms_net) AS operational_gms,
    SUM(gms_net) AS accounting_gms,
    SUM(ABS(trans_gms_net - gms_net)) AS total_variance,
    SUM(CASE WHEN ABS(trans_gms_net - gms_net) > 0.01 THEN 1 ELSE 0 END) AS mismatched_count
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans`
WHERE date = '2026-03-19'
  AND gms_eligible = 1
GROUP BY date
```

---

## Important Notes

### Operational vs Accounting GMS

This table has **both** GMS perspectives:

**Operational GMS (trans_* fields)**:
- Calculated directly from transaction prices
- `trans_gms_net` = usd_subtotal_price + gift_wrap_price (for live, eligible transactions)
- Real-time, matches transaction data exactly
- **Use for operational reporting**

**Accounting GMS (gms_* fields)**:
- From billing/ledger systems via accounting_gms_by_trans
- May differ slightly due to timing, rounding, corrections
- **Use for financial/accounting reconciliation**

### GMS vs GMV vs Purchase Value

- **GMS** (`trans_gms_net`): Merchandise only (listing + gift wrap), eligible only
- **GMV** (`gmv`): GMS + shipping, eligible only
- **Purchase Value** (`purchase_value`): Everything including non-eligible transactions

### GMS Eligibility

Transactions are **excluded from GMS** (`gms_eligible = 0`) if any of these are true:
- `is_gift_card = 1` - Gift card purchases
- `is_test_seller = 1` - Test sellers
- `is_test_buyer = 1` - Test buyers
- `is_cash = 1` - Cash payments
- `is_fraud = 1` - Fraudulent transactions
- `is_highval = 1 AND live = 0` - High-value cancelled transactions (>$10K)

### Gift Wrap Allocation

Gift wrap price is allocated to the **first transaction** (transaction_seq = 1) in each receipt. Other transactions in the same receipt have 0 gift wrap.

### Already in Dollars

All GMS fields are NUMERIC in USD dollars (not cents). No conversion needed for display.

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transaction_obt` - **PREFER THIS** - Complete transaction data with all fields pre-joined
- `transaction_mart.transactions_gms` - Alternative view with accounting GMS in main columns
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transaction_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - Visit attribution
- `transaction_mart.accounting_gms` - Detailed accounting GMS with dates/sources
- `transaction_mart.accounting_gms_by_trans` - Simplified accounting GMS (already joined here)

### Receipt Tables (Join via receipt_id)
- `transaction_mart.receipts_gms` - Receipt-level GMS aggregations
- `transaction_mart.all_receipts` - Receipt details
- `transaction_mart.receipt_obt` - Receipt-level OBT
- `transaction_mart.receipt_post_purch` - Receipt post-purchase data
- `transaction_mart.receipts_visits` - Receipt visit attribution

### Component Tables (Already Joined - Avoid Direct Joins)
- `transaction_mart.transactions_buyer` - Buyer data (already here)
- `transaction_mart.transactions_seller` - Seller data (already here)
- `transaction_mart.all_transactions_categories` - Category data (already here)

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.shop_transactions` - Raw transactions (use all_transactions instead)
