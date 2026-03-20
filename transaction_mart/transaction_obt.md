---
id: transaction_obt
title: etsy-data-warehouse-prod.transaction_mart.transaction_obt
sidebar_label: transaction_obt
---

# transaction_obt

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.transaction_obt`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: **One Big Table (OBT) for transactions**. Complete transaction-level data with all related information pre-joined: buyer/seller details, GMS, categories, visit attribution, post-purchase metrics. **PREFERRED TABLE for transaction-level queries**.

**Primary Key**: `transaction_id`

**Partitioning**: Partitioned by DATE(creation_tsz) (receipt creation date)

**Clustering**: Clustered by `mapped_user_id`

**Update Frequency**: Daily (p1 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

This table joins data from multiple sources. See individual source tables for detailed business logic.

### Transaction Identifiers

| Column | Source | Description |
|--------|--------|-------------|
| `transaction_id` | all_transactions | Unique transaction identifier. Primary key |
| `receipt_id` | all_receipts | Parent receipt ID |
| `transaction_seq` | all_transactions | Sequential number within receipt (1, 2, 3...) |
| `listing_id` | all_transactions | Listing that was purchased |

### Receipt Info & Dates

| Column | Source | Description |
|--------|--------|-------------|
| `receipt_type` | all_receipts | Receipt type code (0=etsy, 1=pattern, 3=leo, 5=offsite) |
| `market` | all_receipts | Market (etsy, ipp, wholesale, pattern, leo, offsite) |
| `receipt_live` | all_receipts | Receipt is live (not cancelled) |
| `receipt_group_id` | all_receipts | Groups receipts from multishop checkout |
| `offsite_application_id` | all_receipts | Offsite application ID (Etsy Open API) |
| `offsite_application_name` | all_receipts | Offsite application name |
| `creation_tsz` | all_receipts | **Receipt** creation timestamp. **Used for partitioning** |
| `prior_receipt_tsz` | all_receipts | Previous receipt timestamp for this buyer (any market) |
| `prior_receipt_tsz_same_mkt` | all_receipts | Previous receipt timestamp for this buyer in same market |
| `first_receipt_tsz` | all_receipts | First ever receipt timestamp for this buyer |
| `is_paid` | all_receipts | Receipt is paid |

### Transaction Status & Dates

| Column | Source | Description |
|--------|--------|-------------|
| `live` | all_transactions | Transaction is live (not cancelled). 0 = cancelled, 1 = live |
| `paid_tsz` | all_transactions | Transaction paid timestamp |
| `shipped_tsz` | all_transactions | Transaction shipped timestamp |
| `product_offering_id` | all_transactions | Product offering ID |

### User IDs & Guest Data

| Column | Source | Description |
|--------|--------|-------------|
| `mapped_user_id` | all_receipts | Canonical buyer ID (handles merged accounts). **Used for clustering** |
| `buyer_user_id` | all_receipts | Buyer's user ID |
| `seller_user_id` | all_receipts | Seller's user ID |
| `is_guest_checkout` | all_receipts | Buyer used guest checkout (guest or claimed) |
| `is_guest` | all_receipts | Checkout by guest, not later claimed |
| `is_claimed` | all_receipts | Guest checkout later claimed by registered user |
| `is_test_buyer` | transactions_gms_by_trans | Buyer is test user |
| `is_test_seller` | transactions_gms_by_trans | Seller is test user |

### Buyer Location

| Column | Source | Description |
|--------|--------|-------------|
| `buyer_country_id` | transactions_gms_by_trans | Buyer's country ID. **PII** |
| `buyer_country` | transactions_gms_by_trans | Buyer's country code. **PII** |
| `buyer_country_name` | transactions_gms_by_trans | Buyer's country name. **PII** |
| `buyer_state` | transactions_gms_by_trans | Buyer's state/province. **PII** |
| `buyer_city` | transactions_gms_by_trans | Buyer's city. **PII** |
| `buyer_zip` | transactions_gms_by_trans | Buyer's postal code. **PII** |
| `buyer_currency` | transactions_gms_by_trans | Currency buyer paid in |
| `buyer_exch_rt` | all_transactions | Exchange rate from buyer currency to USD |

### Seller Location

| Column | Source | Description |
|--------|--------|-------------|
| `seller_country_id` | transactions_gms_by_trans | Seller's country ID. **PII** |
| `seller_country` | transactions_gms_by_trans | Seller's country code. **PII** |
| `seller_country_name` | transactions_gms_by_trans | Seller's country name. **PII** |
| `seller_state` | transactions_gms_by_trans | Seller's state. **PII** |
| `seller_zip` | transactions_gms_by_trans | Seller's postal code. **PII** |
| `seller_currency` | all_transactions | Currency seller receives |
| `seller_exch_rt` | all_transactions | Exchange rate from seller currency to USD |

### Checkout Flags

| Column | Source | Description |
|--------|--------|-------------|
| `is_express_checkout` | all_receipts | Used express checkout (BUY IT NOW button) |
| `is_multishop_checkout` | all_receipts | Part of multishop checkout |
| `requested_gift_wrap` | all_receipts | Gift wrap was requested |

### Payment

| Column | Source | Description |
|--------|--------|-------------|
| `payment_method` | all_receipts | Payment method used |
| `dc_payment_method` | all_receipts | Direct checkout payment method |
| `card_type` | all_receipts | Credit card type. **PII** |
| `shop_payment_id` | all_receipts | Shop payment ID |
| `in_person_payment_type` | all_receipts | In-person payment type ('NA' if not IPP) |
| `is_dc` | transactions_gms_by_trans | Direct checkout payment (Etsy Payments) |
| `is_square` | all_transactions | Square payment |
| `default_rate` | all_receipts | Default transaction fee rate |

### Gift Fields

| Column | Source | Description |
|--------|--------|-------------|
| `gift_recipient_user_id` | all_receipts | Gift recipient user ID |
| `gift_recipient_name` | all_receipts | Gift recipient name |
| `gift_recipient_email` | all_receipts | Gift recipient email |
| `gift_claim_status` | all_receipts | Gift claim status code |
| `gift_buyer_first_name` | all_receipts | Gift buyer first name |
| `gift_message` | all_receipts | Gift message text |
| `is_gift_message` | all_receipts | Gift message is populated |
| `is_gift` | all_transactions | Transaction is a gift |
| `is_post_purchase_gift` | all_transactions | Post-purchase gift |
| `post_purchase_gift_timestamp` | all_transactions | When post-purchase gift was added |

### Shipping & Discounts

| Column | Source | Description |
|--------|--------|-------------|
| `shipping_upgrade` | all_transactions | Shipping upgrade selected |
| `trans_shipping_price` | all_transactions | Shipping price allocated to this transaction. **In USD dollars** |
| `trans_gift_wrap_price` | all_transactions | Gift wrap price allocated to this transaction. **In USD dollars** |
| `is_shipping_sale` | all_receipts | Shipping discount via sale |
| `is_shipping_coupon` | all_receipts | Shipping discount via coupon |

### Transaction Prices

| Column | Source | Description |
|--------|--------|-------------|
| `usd_subtotal_price` | all_transactions | Subtotal price. **In USD dollars** |
| `usd_price` | all_transactions | Unit price. **In USD dollars** |
| `price` | all_transactions | Price in original currency |
| `quantity` | all_transactions | Quantity purchased |

### Transaction Details

| Column | Source | Description |
|--------|--------|-------------|
| `is_digital` | all_transactions | Transaction is digital/download |
| `is_download` | all_transactions | Transaction has downloadable files |
| `is_custom_order` | all_transactions | Transaction is custom order |
| `personalization_request` | all_transactions | Personalization text requested |
| `is_fraud` | all_transactions | Transaction flagged as fraud |
| `is_highval` | all_transactions | High-value transaction (>$10K) |
| `is_cash` | all_transactions | Payment was cash |
| `is_gift_card` | all_transactions | Transaction is gift card purchase |

### Category & Taxonomy

| Column | Source | Description |
|--------|--------|-------------|
| `taxonomy_id` | all_transactions_categories | Taxonomy ID |
| `new_category` | all_transactions_categories | Top-level category name |
| `second_level_cat_new` | all_transactions_categories | Second-level category |
| `third_level_cat_new` | all_transactions_categories | Third-level category |
| `is_handmade` | all_transactions_categories | Item is handmade |
| `is_vintage` | all_transactions_categories | Item is vintage (20+ years old) |
| `is_supplies` | all_transactions_categories | Item is craft supplies |

### GMS Eligibility

| Column | Source | Description |
|--------|--------|-------------|
| `gms_eligible` | transactions_gms_by_trans | Transaction is eligible for GMS (excludes gift cards, test users, cash, fraud, high-value cancelled) |

### GMS - Transaction-Based (Operational)

| Column | Source | Description |
|--------|--------|-------------|
| `trans_gms_gross` | transactions_gms_by_trans | Transaction GMS gross. **In USD dollars** |
| `trans_gms_net` | transactions_gms_by_trans | Transaction GMS net (excludes cancelled). **In USD dollars** |
| `trans_listing_gms_gross` | transactions_gms_by_trans | Listing GMS gross (no gift wrap). **In USD dollars** |
| `trans_listing_gms_net` | transactions_gms_by_trans | Listing GMS net (no gift wrap, excludes cancelled). **In USD dollars** |
| `trans_giftwrap_gms_gross` | transactions_gms_by_trans | Gift wrap GMS gross. **In USD dollars** |
| `trans_giftwrap_gms_net` | transactions_gms_by_trans | Gift wrap GMS net (excludes cancelled). **In USD dollars** |

### GMS - Accounting-Based

| Column | Source | Description |
|--------|--------|-------------|
| `gms_gross` | transactions_gms_by_trans | Accounting GMS gross. **In USD dollars** |
| `gms_net` | transactions_gms_by_trans | Accounting GMS net. **In USD dollars** |
| `listing_gms_gross` | transactions_gms_by_trans | Accounting listing GMS gross. **In USD dollars** |
| `listing_gms_net` | transactions_gms_by_trans | Accounting listing GMS net. **In USD dollars** |
| `giftwrap_gms_gross` | transactions_gms_by_trans | Accounting gift wrap GMS gross. **In USD dollars** |
| `giftwrap_gms_net` | transactions_gms_by_trans | Accounting gift wrap GMS net. **In USD dollars** |

### Receipt Metrics

| Column | Source | Description |
|--------|--------|-------------|
| `purchase_value` | receipts_gms | Total purchase value for entire receipt. **In USD dollars** |
| `top_category_gms` | receipts_gms | Category with highest GMS in receipt |
| `top_category_items` | receipts_gms | Category with most items in receipt |
| `transaction_count` | receipts_gms | Total number of transactions in receipt |
| `live_transaction_count` | receipts_gms | Number of non-cancelled transactions in receipt |

### Visit Attribution

| Column | Source | Description |
|--------|--------|-------------|
| `visit_id` | receipts_visits | Visit ID. NULL if no visit match |
| `_date` | receipts_visits | Visit date. NULL if no visit match |
| `user_id` | receipts_visits | User ID from visit |
| `start_datetime` | receipts_visits | Visit start datetime |
| `visit_gms` | receipts_visits | Total buyer GMS in visit (not just this transaction) |
| `visit_market` | receipts_visits | Visit market (etsy/pattern/leo) |
| `platform_os` | receipts_visits | Platform OS (iOS, Android, Windows, Mac, etc.) |
| `platform_device` | receipts_visits | Platform device (mobile, desktop, tablet) |
| `platform_app` | receipts_visits | Platform app (BOE, SOE, Desktop) |
| `mapped_platform_type` | receipts_visits | Mapped platform type |
| `top_channel` | receipts_visits | Top-level marketing channel |
| `second_channel` | receipts_visits | Second-level marketing channel |
| `third_channel` | receipts_visits | Third-level marketing channel |
| `canonical_region` | receipts_visits | Visit region |
| `referrer_type` | receipts_visits | Referrer type |
| `referring_domain` | receipts_visits | Referring domain |

### Post-Purchase Metrics

| Column | Source | Description |
|--------|--------|-------------|
| `review` | transaction_post_purch | Review text left by buyer |
| `rating` | transaction_post_purch | Review rating (1-5 stars) |

---

## Query Guidance

### When to Use This Table

**✅ USE `transaction_mart.transaction_obt` for:**
- **ANY transaction-level analysis** - This is the preferred table
- Line-item level analysis
- Product/listing analysis
- Category performance
- Visit attribution by transaction
- Review/rating analysis
- **One-stop shop** - All transaction data in one table

**❌ DON'T use this table for:**
- Receipt/order level aggregations → Use `transaction_mart.receipt_obt` (more efficient for receipt queries)

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `transaction_mart.all_transactions` - Core transaction data already here
- `transaction_mart.transactions_gms_by_trans` - GMS data already here
- `transaction_mart.all_transactions_categories` - Category data already here
- `transaction_mart.receipts_visits` - Visit data already here (at receipt level)
- `transaction_mart.transaction_post_purch` - Post-purchase data already here

**✅ Do join if needed:**
- `transaction_mart.receipt_obt` - Alternative: query at receipt level instead

### Performance Tips

1. **Partitioned by DATE(creation_tsz)** - Always filter on creation_tsz date for best performance (note: this is receipt creation date, not transaction date)
2. **Clustered by mapped_user_id** - Efficient for buyer-centric queries
3. **All data pre-joined** - No need to join component tables
4. **Filter test users** - `is_test_buyer = 0 AND is_test_seller = 0`
5. **Filter by live** - `live = 1` to exclude cancelled transactions
6. **Use trans_gms_net for revenue** - Standard operational GMS metric

### Common Query Patterns

**Daily transactions and GMS:**
```sql
SELECT
    DATE(creation_tsz) AS order_date,
    market,
    COUNT(*) AS transaction_count,
    SUM(quantity) AS items_sold,
    SUM(trans_gms_net) AS total_gms,
    AVG(trans_gms_net) AS avg_transaction_value
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_obt`
WHERE DATE(creation_tsz) BETWEEN '2026-03-01' AND '2026-03-19'
  AND live = 1
  AND is_test_buyer = 0
  AND is_test_seller = 0
  AND gms_eligible = 1
GROUP BY order_date, market
ORDER BY order_date, market
```

**Category performance:**
```sql
SELECT
    new_category,
    second_level_cat_new,
    COUNT(*) AS transaction_count,
    SUM(quantity) AS items_sold,
    SUM(trans_gms_net) AS category_gms,
    AVG(trans_gms_net) AS avg_price
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND live = 1
  AND is_test_buyer = 0
  AND gms_eligible = 1
GROUP BY new_category, second_level_cat_new
ORDER BY category_gms DESC
LIMIT 20
```

**Visit attribution analysis:**
```sql
SELECT
    top_channel,
    platform_device,
    COUNT(*) AS transaction_count,
    SUM(trans_gms_net) AS total_gms,
    AVG(trans_gms_net) AS avg_gms
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND live = 1
  AND is_test_buyer = 0
  AND visit_id IS NOT NULL
  AND gms_eligible = 1
GROUP BY top_channel, platform_device
ORDER BY total_gms DESC
```

**Review analysis:**
```sql
SELECT
    rating,
    COUNT(*) AS review_count,
    AVG(trans_gms_net) AS avg_transaction_gms,
    SUM(CASE WHEN is_digital = 1 THEN 1 ELSE 0 END) AS digital_count
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND rating IS NOT NULL
  AND live = 1
  AND gms_eligible = 1
GROUP BY rating
ORDER BY rating
```

**Listing performance:**
```sql
SELECT
    listing_id,
    new_category,
    COUNT(*) AS transaction_count,
    SUM(quantity) AS total_quantity_sold,
    SUM(trans_gms_net) AS total_gms,
    AVG(usd_price) AS avg_unit_price,
    COUNT(DISTINCT buyer_user_id) AS unique_buyers
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND live = 1
  AND is_test_buyer = 0
  AND gms_eligible = 1
GROUP BY listing_id, new_category
ORDER BY total_gms DESC
LIMIT 100
```

---

## Important Notes

### Preferred Table for Transaction Analysis

**This is the PREFERRED table for transaction-level queries**. It joins:
- `all_transactions` - Core transaction data
- `all_receipts` - Receipt-level details
- `transactions_gms_by_trans` - GMS metrics
- `all_transactions_categories` - Category data
- `receipts_visits` - Visit attribution (at receipt level, applies to all transactions in receipt)
- `transaction_post_purch` - Post-purchase metrics
- `receipts_gms` - Receipt aggregates (for context)

Use this table instead of joining component tables yourself.

### Transaction vs Accounting GMS

Two sets of GMS fields:
- **trans_*** fields - Operational GMS calculated from transaction prices
- **gms_*** fields (no prefix) - Accounting GMS from billing systems

For most operational reporting, use `trans_gms_net`.

### Visit Attribution is Receipt-Level

Visit fields come from `receipts_visits`, so **all transactions in a receipt have the same visit attribution**. This is expected - the visit led to the entire receipt/order, not individual transactions.

### Post-Purchase Fields

Post-purchase fields (review, rating) are NULL if the transaction has no post-purchase activity. Use COALESCE or IS NULL checks as needed.

### Receipt Aggregates

Fields like `transaction_count`, `top_category_gms`, and `purchase_value` are **receipt-level aggregates** from receipts_gms. They're the same for all transactions in a receipt.

### Partition by Receipt Date

The table is partitioned by **receipt** creation_tsz, not transaction-specific dates. All transactions in a receipt are in the same partition.

### Already in Dollars

All price and GMS fields are in USD dollars (not cents). No conversion needed for display.

---

## Related Tables

### Receipt Tables
- `transaction_mart.receipt_obt` - **USE FOR RECEIPT-LEVEL** - One Big Table at receipt level
- `transaction_mart.all_receipts` - Receipt details (already joined here)

### Component Tables (Avoid - Already Joined Here)
- `transaction_mart.all_transactions` - Core transaction data (already here)
- `transaction_mart.transactions_gms_by_trans` - GMS data (already here)
- `transaction_mart.all_transactions_categories` - Category data (already here)
- `transaction_mart.receipts_visits` - Visit attribution (already here)
- `transaction_mart.transaction_post_purch` - Post-purchase data (already here)
- `transaction_mart.receipts_gms` - Receipt aggregates (already here)
