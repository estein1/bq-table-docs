---
id: all_receipts
title: transaction_mart.all_receipts
sidebar_label: all_receipts
---

# transaction_mart.all_receipts

## Table Overview

**Purpose**: Foundation table containing core receipt-level data for all Etsy orders. One row per receipt with buyer information, payment details, and order metadata.

**Primary Key**: `receipt_id`

**Clustering**: Clustered by `receipt_id`

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `receipt_id` | INT64 | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Unique receipt identifier. Primary key |
| `order_id` | INT64 | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Order ID (different from receipt_id) |
| `receipt_group_id` | INT64 | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Groups receipts from multishop checkout |
| `buyer_user_id` | INT64 | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Buyer's user ID |
| `mapped_user_id` | INT64 | `user_mart.user_mapping` | Mapped user ID from user_mapping table, defaults to buyer_user_id if no mapping | Canonical user ID (handles merged accounts) |
| `seller_user_id` | INT64 | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Seller's user ID |

### Market & Receipt Type

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `market` | STRING | Derived from multiple sources | Determined by: in_person_payment_type → wholesale (purchase_order_id) → receipt_type (pattern=1, leo=3, offsite=5) → default 'etsy' | Market where purchase occurred (etsy, ipp, wholesale, pattern, leo, offsite) |
| `receipt_type` | INT64 | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Receipt type code (0=etsy, 1=pattern, 3=leo, 5=offsite) |
| `receipt_live` | INT64 | `etsy_shard.shop_receipts2` | Returns 1 if status=1, else 0 | Receipt is live (not deleted/cancelled) |

### Dates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `creation_tsz` | TIMESTAMP | `etsy_shard.shop_receipts2` | TIMESTAMP_SECONDS(create_date) | Receipt creation timestamp |
| `prior_receipt_tsz` | TIMESTAMP | Calculated from buyer purchase history | LAG of create_date by mapped_user_id | Previous receipt timestamp for this buyer (any market) |
| `prior_receipt_tsz_same_mkt` | TIMESTAMP | Calculated from buyer purchase history | LAG of create_date by mapped_user_id and market | Previous receipt timestamp for this buyer in same market |
| `first_receipt_tsz` | TIMESTAMP | Calculated from buyer purchase history | FIRST_VALUE of create_date by mapped_user_id | First ever receipt timestamp for this buyer |

### Location

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `city` | STRING | `etsy_shard.shop_receipts2` | Trimmed and truncated to 80 chars | Buyer city. **PII** |
| `state` | STRING | `etsy_shard.shop_receipts2` | Trimmed and truncated to 80 chars | Buyer state/province. **PII** |
| `zip` | STRING | `etsy_shard.shop_receipts2` | Cleaned via clean_zip_code UDF | Buyer postal code. **PII** |
| `country_id` | INT64 | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Buyer country ID. **PII** |

### Checkout Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_guest` | INT64 | `etsy_shard.shop_receipts2` | Cast is_guest to INT64 | Buyer used guest checkout |
| `is_claimed` | INT64 | `etsy_shard.shop_receipts2` | Cast is_claimed to INT64 | Guest checkout was later claimed by user |
| `is_guest_checkout` | INT64 | Derived from is_guest and is_claimed | LEAST(is_guest + is_claimed, 1) | Either guest or claimed guest checkout |
| `is_express_checkout` | INT64 | `etsy_index.checkout_audit`, `etsy_index.carts_index` | Returns 1 if cart type_id=5 (express checkout) | Used express checkout |
| `is_multishop_checkout` | INT64 | Derived from receipt_group_id | Returns 1 if receipt_group_id has >1 receipts | Part of multishop cart |
| `requested_gift_wrap` | INT64 | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Gift wrap was requested |
| `is_gift_message` | INT64 | `etsy_shard.shop_receipts2` | Returns 1 if gift_message is not empty | Gift message included |

### Payment

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `payment_method` | STRING | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Payment method used |
| `dc_payment_method` | STRING | `etsy_payments.shop_payments` | Direct from shop_payments | Data center payment method |
| `card_type` | STRING | `etsy_payments.cc_txns` | First 2 chars of card_type from cc_txns where instrument_type='CREDITCARD' | Card type for credit card payments. **PII** |
| `shop_payment_id` | INT64 | `etsy_payments.shop_payments` | Direct from shop_payments | Shop payment ID |
| `in_person_payment_type` | STRING | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | In-person payment type ('NA' if not IPP) |

### Prices

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `receipt_usd_total_price` | NUMERIC | `etsy_shard.shop_receipts_totals` | usd_total_price / 100 | Total price in USD dollars. **Original in cents, converted to dollars** |
| `receipt_usd_subtotal_price` | NUMERIC | `etsy_shard.shop_receipts_totals` | usd_subtotal_price / 100 | Subtotal price in USD dollars. **Original in cents, converted to dollars** |
| `receipt_usd_shipping_price` | NUMERIC | `etsy_shard.shop_receipts_totals` | usd_shipping_price / 100 | Shipping price in USD dollars. **Original in cents, converted to dollars** |
| `usd_gift_wrap_price` | NUMERIC | `etsy_shard.shop_receipts2` | usd_gift_wrap_price / 100 | Gift wrap price in USD dollars. **Original in cents, converted to dollars** |
| `receipt_usd_shipping_discount` | NUMERIC | `etsy_shard.shop_receipts_totals` | usd_shipping_discount / 100 | Shipping discount in USD dollars. **Original in cents, converted to dollars** |

### Shipping Discounts

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_shipping_coupon` | INT64 | `etsy_shard.seller_marketing_promotion`, `etsy_shard.seller_marketing_promotion_revenue_attribution` | Returns 1 if shipping promotion with discoverability_type=1 exists | Shipping discount via coupon |
| `is_shipping_sale` | INT64 | `etsy_shard.seller_marketing_promotion`, `etsy_shard.seller_marketing_promotion_revenue_attribution` | Returns 1 if shipping promotion with discoverability_type=2 exists | Shipping discount via sale |

### Fraud & Risk

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_fraud` | INT64 | `etsy_payments.fraud_responses` | Returns 1 if recommendation_code='REJECT' or review_response='REJECT' for latest fraud response | Receipt flagged as fraud |

### Transaction Fees

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `default_rate` | NUMERIC | `etsy_aux.channel_transaction_fee` | Transaction fee rate for market and effective date. Defaults to 0.035 if not found | Default transaction fee rate |
| `override_rate` | NUMERIC | `etsy_aux.channel_transaction_fee_overridden_receipt` | Override rate if receipt has fee override | Overridden transaction fee rate |

### Gift Options

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gift_recipient_user_id` | INT64 | `etsy_shard.gift_receipt_options` | Direct from gift_receipt_options | Gift recipient user ID |
| `gift_recipient_email` | STRING | `etsy_shard.gift_receipt_options` | Direct from gift_receipt_options | Gift recipient email |
| `gift_recipient_name` | STRING | `etsy_shard.gift_receipt_options` | Direct from gift_receipt_options | Gift recipient name |
| `gift_claim_status` | INT64 | `etsy_shard.gift_receipt_options` | Direct from gift_receipt_options | Gift claim status code |
| `gift_message` | STRING | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Gift message text |
| `gift_buyer_first_name` | STRING | `etsy_shard.shop_receipts2` | Direct from shop_receipts2 | Gift buyer first name |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.all_receipts` for:**
- Receipt-level analysis (one row per order)
- Buyer demographics and location
- Payment method analysis
- Guest checkout analysis
- Multishop checkout analysis
- **Foundation table** - Use when you need basic receipt data

**Don't use this table for:**
- Complete order analysis → Use `transaction_mart.receipt_obt` (has everything pre-joined)
- Transaction-level data → Use `transaction_mart.all_transactions`
- GMS calculations → Use `transaction_mart.receipts_gms`
- Visit attribution → Use `transaction_mart.receipts_visits`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete receipt analysis, use `receipt_obt` instead (pre-joined)
- `etsy_shard.shop_receipts2` - This IS that table with enhancements
- `etsy_shard.shop_receipts_totals` - Price totals already joined

**✅ Do join to these mart tables** if not using OBT:
- `transaction_mart.receipts_gms` - For GMS metrics
- `transaction_mart.receipts_visits` - For visit/channel attribution
- `transaction_mart.receipt_post_purch` - For post-purchase metrics
- `transaction_mart.all_transactions` - For transaction-level details

### Performance Tips

1. **Clustered by `receipt_id`** - Always filter on receipt_id when possible
2. **Prices are in dollars** - Unlike listings (which are in cents), receipt prices are already converted
3. **PII fields** - city, state, zip, country_id, card_type contain PII
4. **Use receipt_obt for most queries** - This table is for specific use cases; receipt_obt has everything
5. **Filter Klarna cancelled receipts** - Already excluded from this table

### Common Query Patterns

**Get receipt basics:**
```sql
SELECT
    receipt_id,
    creation_tsz,
    market,
    mapped_user_id,
    receipt_usd_total_price
FROM `etsy-data-warehouse-prod.transaction_mart.all_receipts`
WHERE receipt_id IN (1234, 5678)
```

**Guest checkout analysis:**
```sql
SELECT
    market,
    COUNT(*) AS receipt_count,
    SUM(CASE WHEN is_guest_checkout = 1 THEN 1 ELSE 0 END) AS guest_receipts,
    AVG(receipt_usd_total_price) AS avg_total
FROM `etsy-data-warehouse-prod.transaction_mart.all_receipts`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND receipt_live = 1
GROUP BY market
```

**Repeat buyer analysis:**
```sql
SELECT
    receipt_id,
    creation_tsz,
    first_receipt_tsz,
    DATETIME_DIFF(creation_tsz, first_receipt_tsz, DAY) AS days_since_first_purchase
FROM `etsy-data-warehouse-prod.transaction_mart.all_receipts`
WHERE receipt_live = 1
  AND first_receipt_tsz IS NOT NULL
  AND first_receipt_tsz < creation_tsz
```

---

## Related Tables

### Transaction Mart Tables (Join on receipt_id)
- `transaction_mart.receipt_obt` - **USE THIS** - One Big Table with all receipt data pre-joined
- `transaction_mart.receipts_gms` - GMS metrics per receipt
- `transaction_mart.receipts_visits` - Visit and channel attribution
- `transaction_mart.receipt_post_purch` - Post-purchase metrics (returns, cases, etc.)
- `transaction_mart.all_transactions` - Transaction-level details (join via receipt_id)
- `transaction_mart.transactions_gms` - Transaction-level GMS
- `transaction_mart.accounting_gms` - Accounting-view GMS

### Other Tables
- `transaction_mart.transaction_obt` - Transaction-level OBT
- `transaction_mart.transactions_buyer` - Buyer-level transaction metrics
- `transaction_mart.transactions_seller` - Seller-level transaction metrics
- `transaction_mart.accounting_seller` - Accounting seller metrics
- `transaction_mart.all_transactions_categories` - Transaction categories

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.shop_receipts2` - Raw receipts (this table IS that data cleaned)
- `etsy_shard.shop_receipts_totals` - Price totals (already joined here)
- `user_mart.user_mapping` - User mappings (mapped_user_id already here)
- `etsy_payments.shop_payments` - Payment data (shop_payment_id already here)
