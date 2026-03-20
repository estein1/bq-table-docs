---
id: transactions_gms
title: etsy-data-warehouse-prod.transaction_mart.transactions_gms
sidebar_label: transactions_gms
---

# transactions_gms

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.transactions_gms`

**Source Script:** `Rollups/auto/p1/daily/transactions-mart/trans_mart_by_date.sql`

**Purpose**: Transaction-level GMS (Gross Merchandise Sales) with buyer/seller demographics and category data. Combines operational GMS calculations with accounting GMS for comparison. **Key table for transaction-level revenue analysis**.

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
| `trans_date` | DATE | `transaction_mart.all_transactions` | DATE(creation_tsz) from all_transactions | Transaction date. **Use for date filtering** |
| `market` | STRING | `transaction_mart.all_transactions` | Direct from all_transactions | Market where purchase occurred (etsy, ipp, wholesale, pattern, leo, offsite) |

### Buyer Information

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `buyer_user_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Buyer's user ID |
| `buyer_currency` | STRING | `transaction_mart.all_transactions` | Direct from all_transactions | Currency buyer paid in |
| `buyer_exch_rt` | NUMERIC | `transaction_mart.all_transactions` | Calculated exchange rate from buyer currency to USD | Exchange rate from buyer currency to USD |
| `buyer_country_id` | INT64 | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's country ID. **PII** |
| `buyer_country` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's country code. **PII** |
| `buyer_country_name` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's country name. **PII** |
| `buyer_city` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's city. **PII** |
| `buyer_state` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's state/province. **PII** |
| `buyer_zip` | STRING | `transaction_mart.transactions_buyer` | From transactions_buyer | Buyer's postal code. **PII** |

### Seller Information

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `seller_user_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Seller's user ID |
| `seller_currency` | STRING | `transaction_mart.all_transactions` | Direct from all_transactions | Currency seller receives |
| `seller_exch_rt` | NUMERIC | `transaction_mart.all_transactions` | Calculated exchange rate from seller currency to USD | Exchange rate from seller currency to USD |
| `trans_seller_country_id` | INT64 | `transaction_mart.transactions_seller` | From transactions_seller (transaction time) | Seller's country ID at transaction time. **PII** |
| `trans_seller_country` | STRING | `transaction_mart.transactions_seller` | From transactions_seller (transaction time) | Seller's country code at transaction time. **PII** |
| `trans_seller_country_name` | STRING | `transaction_mart.transactions_seller` | From transactions_seller (transaction time) | Seller's country name at transaction time. **PII** |
| `trans_seller_state` | STRING | `transaction_mart.transactions_seller` | From transactions_seller (transaction time) | Seller's state at transaction time. **PII** |
| `trans_seller_zip` | STRING | `transaction_mart.transactions_seller` | From transactions_seller (transaction time) | Seller's postal code at transaction time. **PII** |
| `acctg_seller_country_id` | INT64 | `transaction_mart.accounting_seller` (primary), fallback to transactions_seller | coalesce(accounting seller location, transaction seller location) | Seller's country ID from accounting perspective. **PII** |
| `acctg_seller_country` | STRING | `transaction_mart.accounting_seller` (primary), fallback to transactions_seller | coalesce(accounting seller location, transaction seller location) | Seller's country code from accounting perspective. **PII** |
| `acctg_seller_country_name` | STRING | `transaction_mart.accounting_seller` (primary), fallback to transactions_seller | coalesce(accounting seller location, transaction seller location) | Seller's country name from accounting perspective. **PII** |
| `acctg_seller_state` | STRING | `transaction_mart.accounting_seller` (primary), fallback to transactions_seller | coalesce(accounting seller location, transaction seller location) | Seller's state from accounting perspective. **PII** |
| `acctg_seller_zip` | STRING | `transaction_mart.accounting_seller` (primary), fallback to transactions_seller | coalesce(accounting seller location, transaction seller location) | Seller's postal code from accounting perspective. **PII** |

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
| `receipt_id` | INT64 | `transaction_mart.transactions_buyer` | From transactions_buyer | Parent receipt ID |
| `transaction_live` | INT64 | `transaction_mart.all_transactions` | live field from all_transactions | Transaction is live (not cancelled) |

### GMS Eligibility & Dates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gms_eligible` | INT64 | Calculated | 1 if all flags are 0: is_gift_card, is_test_seller, is_test_buyer, is_cash, is_fraud. Also excludes high-value cancelled (>$10K) | Transaction is eligible for GMS reporting |
| `acctg_date` | DATE | `transaction_mart.accounting_gms` | coalesce(accounting date, trans_date) | Accounting date. May differ from trans_date |

### Operational GMS (from Accounting System)

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | Accounting GMS gross. Returns 0 if test seller | Total accounting GMS gross. **In USD dollars** |
| `gms_net` | NUMERIC | `transaction_mart.accounting_gms` | Accounting GMS net. Returns 0 if test seller | Total accounting GMS net. **In USD dollars** |
| `listing_gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | Accounting listing GMS gross. Returns 0 if test seller | Listing GMS gross from accounting. **In USD dollars** |
| `listing_gms_net` | NUMERIC | `transaction_mart.accounting_gms` | Accounting listing GMS net. Returns 0 if test seller | Listing GMS net from accounting. **In USD dollars** |
| `giftwrap_gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | Accounting gift wrap GMS gross. Returns 0 if test seller | Gift wrap GMS gross from accounting. **In USD dollars** |
| `giftwrap_gms_net` | NUMERIC | `transaction_mart.accounting_gms` | Accounting gift wrap GMS net. Returns 0 if test seller | Gift wrap GMS net from accounting. **In USD dollars** |
| `trans_correction_gms` | NUMERIC | `transaction_mart.accounting_gms` | Accounting corrections. Returns 0 if test seller | Transaction-level GMS corrections. **In USD dollars** |
| `rcpt_correction_gms` | NUMERIC | `transaction_mart.accounting_gms` | Accounting corrections. Returns 0 if test seller | Receipt-level GMS corrections. **In USD dollars** |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.transactions_gms` for:**
- **Transaction-level revenue analysis** with complete buyer/seller demographics
- GMS reporting by market, category, country
- Comparing operational vs accounting GMS
- **Preferred for operational reporting** (vs accounting_gms for financial reporting)

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt` (has all fields pre-joined)
- Receipt-level GMS → Use `transaction_mart.receipts_gms`
- Simple accounting GMS → Use `transaction_mart.accounting_gms` or `accounting_gms_by_trans`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `transaction_obt` instead (all data pre-joined)
- `transaction_mart.transactions_buyer` - Already joined here
- `transaction_mart.transactions_seller` - Already joined here
- `transaction_mart.accounting_seller` - Already joined here
- `transaction_mart.all_transactions_categories` - Already joined here

**✅ Do join to these mart tables** if not using OBT:
- `transaction_mart.all_transactions` - Additional transaction fields not included here
- `transaction_mart.transaction_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - Visit attribution

### Performance Tips

1. **Filter on `trans_date`** - Use for date-based queries (not partitioned but indexed)
2. **GMS eligibility already applied** - gms_eligible field pre-calculated
3. **Test users** - Filter with `is_test_buyer = 0 AND is_test_seller = 0`
4. **Already in dollars** - All GMS fields are NUMERIC in USD dollars (not cents)
5. **Use transaction_obt for most queries** - This table is for specific use cases

### Common Query Patterns

**Daily GMS by market:**
```sql
SELECT
    trans_date,
    market,
    COUNT(*) AS transaction_count,
    SUM(gms_net) AS total_gms_net
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms`
WHERE trans_date BETWEEN '2026-03-01' AND '2026-03-19'
  AND gms_eligible = 1
  AND is_test_buyer = 0
  AND is_test_seller = 0
GROUP BY trans_date, market
ORDER BY trans_date, market
```

**GMS by buyer country:**
```sql
SELECT
    buyer_country_name,
    COUNT(*) AS transaction_count,
    SUM(gms_net) AS total_gms,
    AVG(gms_net) AS avg_gms
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms`
WHERE trans_date = '2026-03-19'
  AND gms_eligible = 1
  AND is_test_buyer = 0
GROUP BY buyer_country_name
ORDER BY total_gms DESC
LIMIT 20
```

**Cross-border transactions:**
```sql
SELECT
    CASE
        WHEN buyer_country = trans_seller_country THEN 'Domestic'
        ELSE 'International'
    END AS transaction_type,
    COUNT(*) AS transaction_count,
    SUM(gms_net) AS total_gms,
    AVG(gms_net) AS avg_gms
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms`
WHERE trans_date = '2026-03-19'
  AND gms_eligible = 1
  AND is_test_buyer = 0
GROUP BY transaction_type
```

**Category performance:**
```sql
SELECT
    new_category,
    COUNT(*) AS transaction_count,
    SUM(gms_net) AS category_gms,
    AVG(gms_net) AS avg_gms
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms`
WHERE trans_date = '2026-03-19'
  AND gms_eligible = 1
  AND is_test_buyer = 0
  AND is_test_seller = 0
GROUP BY new_category
ORDER BY category_gms DESC
LIMIT 20
```

---

## Important Notes

### GMS Eligibility

Transactions are **excluded from GMS** (`gms_eligible = 0`) if any of these are true:
- `is_gift_card = 1` - Gift card purchases
- `is_test_seller = 1` - Test sellers
- `is_test_buyer = 1` - Test buyers
- `is_cash = 1` - Cash payments
- `is_fraud = 1` - Fraudulent transactions
- `is_highval = 1 AND live = 0` - High-value cancelled transactions (>$10K)

### Accounting vs Transaction GMS

This table contains GMS from the **accounting system** (not the operational calculation):
- `gms_gross`, `gms_net`, etc. are from `accounting_gms` table
- These are the **accounting perspective** of GMS
- May differ slightly from operational calculations
- For operational GMS, see `transactions_gms_by_trans` table

### Transaction vs Accounting Seller Location

Two sets of seller location fields:
- **trans_seller_*** - Seller location at transaction time
- **acctg_seller_*** - Seller location from accounting perspective (may use different timestamp)

For most analyses, use `trans_seller_*` fields.

### Already in Dollars

All GMS fields are NUMERIC in USD dollars (not cents). No conversion needed for display.

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transaction_obt` - **PREFER THIS** - Complete transaction data with all fields pre-joined
- `transaction_mart.transactions_gms_by_trans` - Alternative GMS view with operational calculations
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transaction_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - Visit attribution
- `transaction_mart.accounting_gms` - Detailed accounting GMS with dates/sources
- `transaction_mart.accounting_gms_by_trans` - Simplified accounting GMS

### Receipt Tables (Join via receipt_id)
- `transaction_mart.receipts_gms` - Receipt-level GMS aggregations
- `transaction_mart.all_receipts` - Receipt details
- `transaction_mart.receipt_obt` - Receipt-level OBT
- `transaction_mart.receipt_post_purch` - Receipt post-purchase data
- `transaction_mart.receipts_visits` - Receipt visit attribution

### Component Tables (Already Joined - Avoid Direct Joins)
- `transaction_mart.transactions_buyer` - Buyer data (already here)
- `transaction_mart.transactions_seller` - Seller data (already here)
- `transaction_mart.accounting_seller` - Accounting seller data (already here)
- `transaction_mart.all_transactions_categories` - Category data (already here)

### Source Tables (Avoid - Data Already in Mart)
- `bill_mart_ledger.gms_billing` - Billing ledger (use accounting_gms instead)
- `etsy_payments.fre_gms` - Financial Reporting Engine (use accounting_gms instead)
