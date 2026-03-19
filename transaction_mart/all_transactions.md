---
id: all_transactions
title: transaction_mart.all_transactions
sidebar_label: all_transactions
---

# transaction_mart.all_transactions

## Table Overview

**Purpose**: Foundation table containing core transaction-level data for all Etsy purchases. One row per transaction (line item) with buyer, seller, listing, and price information.

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
| `transaction_id` | INT64 | `schlep_views.transactions_vw` | Direct from transactions_vw | Unique transaction identifier. Primary key |
| `receipt_id` | INT64 | `schlep_views.transactions_vw` | Direct from transactions_vw | Parent receipt ID |
| `transaction_seq` | INT64 | Calculated | ROW_NUMBER() partitioned by receipt_id, ordered by transaction_id | Sequential number of transaction within receipt (1, 2, 3...) |
| `listing_id` | INT64 | `schlep_views.transactions_vw` | Direct from transactions_vw | Listing that was purchased |
| `buyer_user_id` | INT64 | `schlep_views.transactions_vw` | Direct from transactions_vw | Buyer's user ID |
| `mapped_user_id` | INT64 | `user_mart.user_mapping` via all_receipts | Mapped user ID from user_mapping table | Canonical buyer ID (handles merged accounts) |
| `seller_user_id` | INT64 | `schlep_views.transactions_vw` | Direct from transactions_vw | Seller's user ID |

### Market & Type

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `market` | STRING | `transaction_mart.all_receipts` | From receipt's market field, defaults to 'etsy' | Market where purchase occurred (etsy, ipp, wholesale, pattern, leo, offsite) |
| `live` | INT64 | `schlep_views.transactions_vw` | Returns 1 if status=1, else 0 | Transaction is live (not deleted/cancelled) |

### Dates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `creation_tsz` | TIMESTAMP | `schlep_views.transactions_vw` | Direct from transactions_vw | Transaction creation timestamp |
| `date` | DATE | Derived from creation_tsz | DATE(creation_tsz) | Transaction date (for partitioning/grouping) |
| `shipped_tsz` | TIMESTAMP | `schlep_views.transactions_vw` | Direct from transactions_vw | Timestamp when item was marked shipped |
| `paid_tsz` | TIMESTAMP | `schlep_views.transactions_vw` | Direct from transactions_vw | Timestamp when transaction was paid |

### Prices

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `usd_subtotal_price` | NUMERIC | `schlep_views.transactions_vw` | Direct from transactions_vw | Subtotal price in USD dollars. **Already in dollars** |
| `usd_price` | NUMERIC | `schlep_views.transactions_vw` | Direct from transactions_vw | Unit price in USD dollars. **Already in dollars** |
| `price` | NUMERIC | `schlep_views.transactions_vw` | Direct from transactions_vw | Price in original currency |
| `quantity` | INT64 | `schlep_views.transactions_vw` | Direct from transactions_vw | Quantity purchased |
| `trans_gift_wrap_price` | NUMERIC | Calculated from all_receipts | Gift wrap price allocated to first transaction only (transaction_seq=1) | Gift wrap price for this transaction (0 for trans 2+) |
| `trans_shipping_price` | NUMERIC | Calculated from all_receipts | Net shipping (shipping - discount) allocated to first transaction only | Shipping price for this transaction (0 for trans 2+) |

### Currency & Exchange Rates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `buyer_currency` | STRING | `schlep_views.transactions_vw` | buyer_currency_code, defaults to 'USD' | Currency buyer paid in |
| `seller_currency` | STRING | `schlep_views.transactions_vw` | currency_code | Currency seller receives |
| `buyer_exch_rt` | NUMERIC | Calculated | ROUND(buyer_subtotal_price * 1000 / usd_subtotal_price, 4) / 1000 if buyer_currency != 'USD', else 1 | Exchange rate from buyer currency to USD |
| `seller_exch_rt` | NUMERIC | Calculated | ROUND(subtotal_price * 1000 / usd_subtotal_price, 4) / 1000 if seller_currency != 'USD', else 1 | Exchange rate from seller currency to USD |

### Transaction Type Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_gift_card` | INT64 | Derived | Returns 1 if seller_user_id in (15422986, 36489892, 36489881, 36489515, 61172445) | Transaction is for gift card purchase |
| `is_gift` | INT64 | `materialized.transaction_data` | From transaction_data where is_gift=1 | Transaction is a gift |
| `is_digital` | INT64 | `materialized.transaction_data` | From transaction_data where is_digital=1 | Transaction is digital/download |
| `is_download` | INT64 | `etsy_shard.transaction_file_attachments` | Returns 1 if transaction has file attachments | Transaction has downloadable files |
| `is_post_purchase_gift` | INT64 | `materialized.transaction_data` | From transaction_data where is_post_purchase_gift=1 | Gift purchased after original order |
| `post_purchase_gift_timestamp` | TIMESTAMP | `materialized.transaction_data` | Direct from transaction_data | When post-purchase gift was purchased |
| `is_custom_order` | INT64 | `etsy_shard.custom_order_requests` | Returns 1 if transaction in custom_order_requests | Transaction is custom order |

### Checkout & Guest Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_guest` | INT64 | `schlep_views.transactions_vw` | Cast is_guest to INT64 | Buyer used guest checkout |
| `is_guest_checkout` | INT64 | From all_receipts | From receipt's is_guest_checkout | Either guest or claimed guest checkout |
| `requested_gift_wrap` | INT64 | From all_receipts | From receipt's requested_gift_wrap | Gift wrap was requested |

### Payment Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_cash` | INT64 | From all_receipts | Returns 1 if in_person_payment_type in ('SQUARE_CASH', 'CASH') | Payment was cash |
| `is_square` | INT64 | From all_receipts | Returns 1 if in_person_payment_type starts with 'SQUARE' | Payment via Square |

### Risk & Testing

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_fraud` | INT64 | From all_receipts | From receipt's is_fraud flag | Transaction flagged as fraud |
| `is_test_buyer` | INT64 | `materialized.test_users` | Returns 1 if buyer_user_id in test_users | Buyer is test user |
| `is_test_seller` | INT64 | `materialized.test_users` | Returns 1 if seller_user_id in test_users | Seller is test user |
| `is_highval` | INT64 | Calculated | Returns 1 if usd_subtotal_price > 10000 | High-value transaction (>$10K) |

### Transaction Details

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `shipping_upgrade` | INT64 | `schlep_views.transactions_vw` | Regex check for identifier_type:[0-4],identifier:700 in optional_variation_details | Shipping upgrade selected |
| `personalization_request` | STRING | `schlep_views.transactions_vw` | Extracted from optional_variation_details where identifier:54, truncated to 256 chars | Personalization text requested |

### Transaction Fees

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `default_rate` | NUMERIC | From all_receipts | Transaction fee rate from receipt | Default transaction fee rate |
| `override_rate` | NUMERIC | From all_receipts | Override fee rate from receipt | Overridden transaction fee rate |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.all_transactions` for:**
- Transaction-level analysis (one row per line item)
- Item/listing-level metrics
- Quantity analysis
- Per-item pricing
- **Foundation table** - Use when you need basic transaction data

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt` (has everything pre-joined)
- GMS calculations → Use `transaction_mart.transactions_gms`
- Receipt-level data → Use `transaction_mart.all_receipts`
- Visit attribution → Use `transaction_mart.transactions_visits`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete transaction analysis, use `transaction_obt` instead (pre-joined)
- `schlep_views.transactions_vw` - This IS that table with enhancements
- `etsy_shard.shop_transactions` - Use transactions_vw, not raw table

**✅ Do join to these mart tables** if not using OBT:
- `transaction_mart.transactions_gms` - For GMS metrics
- `transaction_mart.transactions_visits` - For visit/channel attribution
- `transaction_mart.transaction_post_purch` - For post-purchase metrics
- `transaction_mart.all_receipts` - For receipt-level details

### Performance Tips

1. **Clustered by `transaction_id`** - Always filter on transaction_id when possible
2. **Prices are in dollars** - usd_subtotal_price, usd_price are already in dollars
3. **Gift wrap/shipping on first transaction only** - Only transaction_seq=1 has these values
4. **Use transaction_obt for most queries** - This table is for specific use cases; transaction_obt has everything
5. **Filter out test users** - Use `is_test_buyer = 0 AND is_test_seller = 0`

### Common Query Patterns

**Get transaction basics:**
```sql
SELECT
    transaction_id,
    receipt_id,
    creation_tsz,
    listing_id,
    seller_user_id,
    usd_subtotal_price,
    quantity
FROM `etsy-data-warehouse-prod.transaction_mart.all_transactions`
WHERE transaction_id IN (1234, 5678)
```

**Calculate total revenue by listing:**
```sql
SELECT
    listing_id,
    COUNT(*) AS transaction_count,
    SUM(quantity) AS items_sold,
    SUM(usd_subtotal_price) AS total_revenue
FROM `etsy-data-warehouse-prod.transaction_mart.all_transactions`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND live = 1
  AND is_test_buyer = 0
  AND is_test_seller = 0
GROUP BY listing_id
ORDER BY total_revenue DESC
```

**Digital vs physical transactions:**
```sql
SELECT
    CASE WHEN is_digital = 1 THEN 'Digital' ELSE 'Physical' END AS product_type,
    COUNT(*) AS transaction_count,
    AVG(usd_subtotal_price) AS avg_price
FROM `etsy-data-warehouse-prod.transaction_mart.all_transactions`
WHERE live = 1
  AND is_test_buyer = 0
GROUP BY product_type
```

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transaction_obt` - **USE THIS** - One Big Table with all transaction data pre-joined
- `transaction_mart.transactions_gms` - GMS metrics per transaction
- `transaction_mart.transactions_visits` - Visit and channel attribution
- `transaction_mart.transaction_post_purch` - Post-purchase metrics
- `transaction_mart.all_receipts` - Receipt-level details (join via receipt_id)
- `transaction_mart.receipts_gms` - Receipt-level GMS
- `transaction_mart.transactions_buyer` - Buyer-level transaction metrics
- `transaction_mart.transactions_seller` - Seller-level transaction metrics
- `transaction_mart.all_transactions_categories` - Transaction categories
- `transaction_mart.accounting_gms` - Accounting-view GMS

### Other Tables
- `transaction_mart.receipt_obt` - Receipt-level OBT
- `transaction_mart.accounting_seller` - Accounting seller metrics

### Source Tables (Avoid - Data Already in Mart)
- `schlep_views.transactions_vw` - Transactions view (this table IS that data cleaned)
- `etsy_shard.shop_transactions` - Raw transactions (use transactions_vw instead)
- `materialized.transaction_data` - Transaction metadata (gift, digital flags already here)
- `user_mart.user_mapping` - User mappings (mapped_user_id already here)
