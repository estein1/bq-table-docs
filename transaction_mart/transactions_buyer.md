---
id: transactions_buyer
title: transaction_mart.transactions_buyer
sidebar_label: transactions_buyer
---

# transaction_mart.transactions_buyer

## Table Overview

**Purpose**: Buyer information for each transaction. One row per transaction with buyer location, payment method, and receipt status details.

**Primary Key**: `transaction_id`

**Clustering**: Clustered by `transaction_id`

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Unique transaction identifier. Primary key |
| `receipt_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Parent receipt ID |
| `buyer_user_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Buyer's user ID |

### Payment

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `payment_method` | STRING | `etsy_payments.shop_payments`, `all_receipts` | CASE: if shop_payments exists then 'dc', else if payment_method='pp' then 'pp', else 'other' | Payment method used. 'dc' = direct checkout, 'pp' = PayPal, 'other' = other methods |

### Buyer Location

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `buyer_country_id` | INT64 | `all_receipts.country_id` via etsy_v2.countries | coalesce(country_id, 0) | Buyer's country ID. Defaults to 0 if missing. **PII** |
| `buyer_country` | STRING | `all_receipts.country_id` via etsy_v2.countries | coalesce(country_code, 'UNKNOWN') | Buyer's country code. Defaults to 'UNKNOWN' if missing. **PII** |
| `buyer_country_name` | STRING | `all_receipts.country_id` via etsy_v2.countries | coalesce(country_name, 'UNKNOWN') | Buyer's country name. Defaults to 'UNKNOWN' if missing. **PII** |
| `buyer_city` | STRING | `all_receipts` | Direct from all_receipts.city | Buyer's city. **PII** |
| `buyer_state` | STRING | `all_receipts` | Direct from all_receipts.state | Buyer's state/province. **PII** |
| `buyer_zip` | STRING | `all_receipts` | coalesce(zip, '') | Buyer's postal code. Defaults to empty string if missing. **PII** |

### Receipt Status

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `receipt_live` | INT64 | `all_receipts` | Cast receipt_live to INT64 | Receipt is live (not deleted/cancelled) |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.transactions_buyer` for:**
- Transaction-level buyer demographics
- Payment method analysis per transaction
- Buyer location analysis at transaction level
- **Enriching transaction queries** with buyer data

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt` (has buyer data pre-joined)
- Receipt-level buyer data → Use `transaction_mart.all_receipts`
- GMS calculations → Use `transaction_mart.transactions_gms`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete transaction analysis, use `transaction_obt` instead (buyer data already joined)
- `transaction_mart.all_receipts` - Receipt data, but prefer OBT for transaction queries

**✅ Do join to these mart tables** if not using OBT:
- `transaction_mart.all_transactions` - Core transaction data (join via transaction_id)
- `transaction_mart.transactions_gms` - GMS metrics
- `transaction_mart.transactions_seller` - Seller location data
- `transaction_mart.all_transactions_categories` - Category data

### Performance Tips

1. **Clustered by `transaction_id`** - Always filter on transaction_id when possible
2. **PII fields** - buyer_country_id, buyer_country, buyer_country_name, buyer_city, buyer_state, buyer_zip contain PII
3. **Use transaction_obt for most queries** - This table is for specific use cases; transaction_obt has buyer data already joined
4. **Payment method logic** - 'dc' means direct checkout (Etsy Payments), 'pp' means PayPal, 'other' covers all other methods

### Common Query Patterns

**Get buyer location for transactions:**
```sql
SELECT
    transaction_id,
    buyer_user_id,
    buyer_country_name,
    buyer_city,
    buyer_state,
    payment_method
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_buyer`
WHERE transaction_id IN (1234, 5678)
```

**Payment method analysis:**
```sql
SELECT
    tb.payment_method,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT tb.buyer_user_id) AS unique_buyers
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_buyer` tb
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON tb.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
GROUP BY tb.payment_method
ORDER BY transaction_count DESC
```

**Buyer country breakdown:**
```sql
SELECT
    tb.buyer_country_name,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT tb.buyer_user_id) AS unique_buyers
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_buyer` tb
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON tb.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
  AND tb.receipt_live = 1
GROUP BY tb.buyer_country_name
ORDER BY transaction_count DESC
LIMIT 20
```

---

## Important Notes

### Payment Method Logic

- **'dc' (Direct Checkout)**: Transaction paid via Etsy Payments (has record in etsy_payments.shop_payments)
- **'pp' (PayPal)**: Transaction paid via PayPal
- **'other'**: All other payment methods (credit card, gift card, etc.)

### PII Data

All buyer location fields contain Personally Identifiable Information (PII):
- `buyer_country_id`
- `buyer_country`
- `buyer_country_name`
- `buyer_city`
- `buyer_state`
- `buyer_zip`

Use appropriate data governance and access controls when querying these fields.

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transaction_obt` - **PREFER THIS** - Complete transaction data with buyer info pre-joined
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transactions_gms` - GMS metrics
- `transaction_mart.transactions_seller` - Seller location data
- `transaction_mart.all_transactions_categories` - Category taxonomy
- `transaction_mart.transaction_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - Visit attribution

### Receipt Tables (Join via receipt_id)
- `transaction_mart.all_receipts` - Receipt-level details
- `transaction_mart.receipts_gms` - Receipt-level GMS
- `transaction_mart.receipt_obt` - Receipt-level OBT
- `transaction_mart.receipt_post_purch` - Receipt post-purchase data
- `transaction_mart.receipts_visits` - Receipt visit attribution

### Other Transaction Tables
- `transaction_mart.accounting_gms` - Accounting perspective GMS
- `transaction_mart.accounting_gms_by_trans` - Alternative accounting GMS
- `transaction_mart.accounting_seller` - Accounting seller data

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.shop_receipts2` - Receipt data (use all_receipts instead)
- `etsy_payments.shop_payments` - Payment data (payment_method already derived here)
- `etsy_v2.countries` - Country lookup (already joined via all_receipts)
