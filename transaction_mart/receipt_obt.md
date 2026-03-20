---
id: receipt_obt
title: etsy-data-warehouse-prod.transaction_mart.receipt_obt
sidebar_label: receipt_obt
---

# receipt_obt

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.receipt_obt`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: **One Big Table (OBT) for receipts**. Complete receipt-level data with all related information pre-joined: buyer/seller details, GMS, visit attribution, post-purchase metrics. **PREFERRED TABLE for receipt-level queries**.

**Primary Key**: `receipt_id`

**Partitioning**: Partitioned by DATE(creation_tsz)

**Clustering**: Clustered by `mapped_user_id`

**Update Frequency**: Daily (p1 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

This table joins data from multiple sources. See individual source tables for detailed business logic.

### Receipt Identifiers & Dates

| Column | Source | Description |
|--------|--------|-------------|
| `receipt_id` | all_receipts | Unique receipt identifier. Primary key |
| `receipt_type` | all_receipts | Receipt type code (0=etsy, 1=pattern, 3=leo, 5=offsite) |
| `market` | all_receipts | Market (etsy, ipp, wholesale, pattern, leo, offsite) |
| `receipt_live` | all_receipts | Receipt is live (not cancelled). 0 = cancelled, 1 = live |
| `receipt_group_id` | all_receipts | Groups receipts from multishop checkout |
| `offsite_application_id` | all_receipts | Offsite application ID (Etsy Open API) |
| `offsite_application_name` | all_receipts | Offsite application name |
| `creation_tsz` | all_receipts | Receipt creation timestamp. **Used for partitioning** |
| `prior_receipt_tsz` | all_receipts | Previous receipt timestamp for this buyer (any market) |
| `prior_receipt_tsz_same_mkt` | all_receipts | Previous receipt timestamp for this buyer in same market |
| `first_receipt_tsz` | all_receipts | First ever receipt timestamp for this buyer |
| `is_paid` | all_receipts | Receipt is paid |

### User IDs & Guest Data

| Column | Source | Description |
|--------|--------|-------------|
| `mapped_user_id` | all_receipts | Canonical buyer ID (handles merged accounts). **Used for clustering** |
| `buyer_user_id` | all_receipts | Buyer's user ID |
| `seller_user_id` | all_receipts | Seller's user ID |
| `is_guest_checkout` | all_receipts | Buyer used guest checkout (guest or claimed) |
| `is_guest` | all_receipts | Checkout by guest, not later claimed |
| `is_claimed` | all_receipts | Guest checkout later claimed by registered user |
| `is_fraud` | all_receipts | Payment flagged as fraud |
| `is_test_buyer` | receipts_gms | Buyer is test user |
| `is_test_seller` | receipts_gms | Seller is test user |

### Buyer Location

| Column | Source | Description |
|--------|--------|-------------|
| `buyer_country_id` | all_receipts | Buyer's country ID. **PII** |
| `buyer_country` | from transactions_gms_by_trans | Buyer's country code. **PII** |
| `buyer_country_name` | from transactions_gms_by_trans | Buyer's country name. **PII** |
| `buyer_state` | all_receipts | Buyer's state/province. **PII** |
| `buyer_city` | all_receipts | Buyer's city. **PII** |
| `buyer_zip` | all_receipts | Buyer's postal code. **PII** |
| `buyer_currency` | from transactions_gms_by_trans | Currency buyer paid in |

### Seller Location

| Column | Source | Description |
|--------|--------|-------------|
| `seller_country_id` | from transactions_gms_by_trans | Seller's country ID. **PII** |
| `seller_country` | from transactions_gms_by_trans | Seller's country code. **PII** |
| `seller_country_name` | receipts_gms | Seller's country name. **PII** |
| `seller_state` | from transactions_gms_by_trans | Seller's state. **PII** |
| `seller_zip` | from transactions_gms_by_trans | Seller's postal code. **PII** |
| `seller_currency` | from transactions_gms_by_trans | Currency seller receives |

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
| `is_dc` | from transactions_gms_by_trans | Direct checkout payment (Etsy Payments) |
| `is_square` | from transactions_gms_by_trans | Square payment |
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
| `is_gift` | receipts_gms | Receipt contains at least one gift transaction |
| `is_post_purchase_gift` | receipts_gms | Contains post-purchase gift |
| `post_purchase_gift_timestamp` | receipts_gms | When post-purchase gift was added |

### Prices (Receipt Level)

| Column | Source | Description |
|--------|--------|-------------|
| `receipt_usd_total_price` | all_receipts | Total price. **In USD dollars** |
| `receipt_usd_subtotal_price` | all_receipts | Subtotal price. **In USD dollars** |
| `receipt_usd_shipping_price` | all_receipts | Shipping price. **In USD dollars** |
| `usd_gift_wrap_price` | all_receipts | Gift wrap price. **In USD dollars** |
| `receipt_usd_shipping_discount` | all_receipts | Shipping discount. **In USD dollars** |
| `is_shipping_sale` | all_receipts | Shipping discount via sale |
| `is_shipping_coupon` | all_receipts | Shipping discount via coupon |

### GMS - Transaction-Based (Operational)

| Column | Source | Description |
|--------|--------|-------------|
| `trans_gms_gross` | receipts_gms | Transaction GMS gross. **In USD dollars** |
| `trans_gms_net` | receipts_gms | Transaction GMS net (excludes cancelled). **In USD dollars** |
| `trans_listing_gms_gross` | receipts_gms | Listing GMS gross (no gift wrap). **In USD dollars** |
| `trans_listing_gms_net` | receipts_gms | Listing GMS net (no gift wrap, excludes cancelled). **In USD dollars** |
| `trans_giftwrap_gms_gross` | receipts_gms | Gift wrap GMS gross. **In USD dollars** |
| `trans_giftwrap_gms_net` | receipts_gms | Gift wrap GMS net (excludes cancelled). **In USD dollars** |

### GMS - Accounting-Based

| Column | Source | Description |
|--------|--------|-------------|
| `gms_gross` | receipts_gms | Accounting GMS gross. **In USD dollars** |
| `gms_net` | receipts_gms | Accounting GMS net. **In USD dollars** |
| `listing_gms_gross` | receipts_gms | Accounting listing GMS gross. **In USD dollars** |
| `listing_gms_net` | receipts_gms | Accounting listing GMS net. **In USD dollars** |
| `giftwrap_gms_gross` | receipts_gms | Accounting gift wrap GMS gross. **In USD dollars** |
| `giftwrap_gms_net` | receipts_gms | Accounting gift wrap GMS net. **In USD dollars** |

### Receipt Metrics

| Column | Source | Description |
|--------|--------|-------------|
| `purchase_value` | receipts_gms | Total purchase value (includes non-GMS-eligible). **In USD dollars** |
| `transaction_count` | receipts_gms | Number of transactions (line items) in receipt |
| `live_transaction_count` | receipts_gms | Number of non-cancelled transactions |
| `top_category_gms` | receipts_gms | Category with highest GMS |
| `top_category_items` | receipts_gms | Category with most items |

### Transaction Flags

| Column | Source | Description |
|--------|--------|-------------|
| `is_cash` | receipts_gms | Payment was cash |
| `is_gift_card` | receipts_gms | Receipt contains gift card purchase |
| `has_custom` | receipts_gms | Receipt contains custom order |
| `has_digital` | receipts_gms | Receipt contains digital item |

### Visit Attribution

| Column | Source | Description |
|--------|--------|-------------|
| `visit_id` | receipts_visits | Visit ID. NULL if no visit match |
| `_date` | receipts_visits | Visit date. NULL if no visit match |
| `user_id` | receipts_visits | User ID from visit |
| `start_datetime` | receipts_visits | Visit start datetime |
| `visit_gms` | receipts_visits | Total buyer GMS in visit (not just this receipt) |
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
| `transactions_in_receipt` | receipt_post_purch | Number of transactions in receipt |
| `receipt_refund` | receipt_post_purch | Total refund amount. **In USD dollars** |
| `receipt_return` | receipt_post_purch | Receipt had return performed (1 = yes, 0 = no) |
| `help_with_order` | receipt_post_purch | Buyer requested help with order (1 = yes, 0 = no) |
| `non_delivery_cases` | receipt_post_purch | Number of non-delivery cases |
| `not_as_described_cases` | receipt_post_purch | Number of not-as-described cases |
| `arrived_damaged_cases` | receipt_post_purch | Number of arrived-damaged cases |

---

## Query Guidance

### When to Use This Table

**✅ USE `transaction_mart.receipt_obt` for:**
- **ANY receipt-level analysis** - This is the preferred table
- Revenue reporting by receipt
- Buyer/seller demographics
- Visit attribution and marketing analysis
- Post-purchase metrics (returns, refunds, cases)
- **One-stop shop** - All receipt data in one table

**❌ DON'T use this table for:**
- Transaction/line-item level analysis → Use `transaction_mart.transaction_obt`

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `transaction_mart.all_receipts` - Core receipt data already here
- `transaction_mart.receipts_gms` - GMS data already here
- `transaction_mart.receipts_visits` - Visit data already here
- `transaction_mart.receipt_post_purch` - Post-purchase data already here

**✅ Do join if needed:**
- `transaction_mart.all_transactions` - For transaction-level details (join via receipt_id)
- `transaction_mart.transaction_obt` - Alternative: query at transaction level instead

### Performance Tips

1. **Partitioned by DATE(creation_tsz)** - Always filter on creation_tsz date for best performance
2. **Clustered by mapped_user_id** - Efficient for buyer-centric queries
3. **All data pre-joined** - No need to join component tables
4. **Filter test users** - `is_test_buyer = 0 AND is_test_seller = 0`
5. **Use trans_gms_net for revenue** - Standard operational GMS metric

### Common Query Patterns

**Daily receipts and GMS:**
```sql
SELECT
    DATE(creation_tsz) AS order_date,
    market,
    COUNT(*) AS receipt_count,
    SUM(trans_gms_net) AS total_gms,
    AVG(trans_gms_net) AS avg_order_value
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_obt`
WHERE DATE(creation_tsz) BETWEEN '2026-03-01' AND '2026-03-19'
  AND is_test_buyer = 0
  AND is_test_seller = 0
GROUP BY order_date, market
ORDER BY order_date, market
```

**Receipt analysis with visit attribution:**
```sql
SELECT
    top_channel,
    platform_device,
    COUNT(*) AS receipt_count,
    SUM(trans_gms_net) AS total_gms,
    AVG(transaction_count) AS avg_items_per_order
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND is_test_buyer = 0
  AND visit_id IS NOT NULL
GROUP BY top_channel, platform_device
ORDER BY total_gms DESC
```

**Cross-border analysis:**
```sql
SELECT
    buyer_country_name,
    seller_country_name,
    CASE
        WHEN buyer_country = seller_country THEN 'Domestic'
        ELSE 'International'
    END AS order_type,
    COUNT(*) AS receipt_count,
    SUM(trans_gms_net) AS total_gms
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND is_test_buyer = 0
GROUP BY buyer_country_name, seller_country_name, order_type
ORDER BY total_gms DESC
LIMIT 20
```

**Post-purchase issue rate:**
```sql
SELECT
    DATE(creation_tsz) AS order_date,
    COUNT(*) AS total_receipts,
    SUM(CASE WHEN receipt_refund > 0 THEN 1 ELSE 0 END) AS refunded_receipts,
    SUM(CASE WHEN receipt_return = 1 THEN 1 ELSE 0 END) AS returned_receipts,
    SUM(CASE WHEN help_with_order = 1 THEN 1 ELSE 0 END) AS help_requests,
    SUM(non_delivery_cases + not_as_described_cases + arrived_damaged_cases) AS total_cases
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND receipt_live = 1
GROUP BY order_date
```

---

## Important Notes

### Preferred Table for Receipt Analysis

**This is the PREFERRED table for receipt-level queries**. It joins:
- `all_receipts` - Core receipt data
- `receipts_gms` - GMS metrics
- `receipts_visits` - Visit attribution
- `receipt_post_purch` - Post-purchase metrics
- Selected fields from `transactions_gms_by_trans` (using MAX to get receipt-level values)

Use this table instead of joining component tables yourself.

### Transaction vs Accounting GMS

Two sets of GMS fields:
- **trans_*** fields - Operational GMS calculated from transaction prices
- **gms_*** fields (no prefix) - Accounting GMS from billing systems

For most operational reporting, use `trans_gms_net`.

### NULL Visit Fields

Receipts without visit attribution have NULL for all visit-related fields. This is expected for:
- Guest checkout
- API orders
- In-person payments

Filter `WHERE visit_id IS NOT NULL` for attributed receipts only.

### Post-Purchase Fields

Post-purchase fields are NULL if the receipt has no post-purchase activity. Use COALESCE or IS NULL checks as needed.

### Already in Dollars

All price and GMS fields are in USD dollars (not cents). No conversion needed for display.

---

## Related Tables

### Transaction Tables
- `transaction_mart.transaction_obt` - **USE FOR TRANSACTION-LEVEL** - One Big Table at transaction level
- `transaction_mart.all_transactions` - Transaction details (join if you need specific transaction fields)

### Component Tables (Avoid - Already Joined Here)
- `transaction_mart.all_receipts` - Core receipt data (already here)
- `transaction_mart.receipts_gms` - GMS data (already here)
- `transaction_mart.receipts_visits` - Visit attribution (already here)
- `transaction_mart.receipt_post_purch` - Post-purchase data (already here)
