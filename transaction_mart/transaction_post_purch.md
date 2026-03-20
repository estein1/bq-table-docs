---
id: transaction_post_purch
title: etsy-data-warehouse-prod.transaction_mart.transaction_post_purch
sidebar_label: transaction_post_purch
---

# transaction_post_purch

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.transaction_post_purch`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: Post-purchase metrics for transactions. One row per transaction that has post-purchase activity (refunds, cancellations, or reviews). **Excludes transactions with no post-purchase activity**.

**Primary Key**: `transaction_id`

**Clustering**: Not specified

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_id` | INT64 | `transaction_mart.transactions_gms_by_trans` | Direct from transactions_gms_by_trans | Unique transaction identifier. Primary key |
| `receipt_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Parent receipt ID |
| `listing_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Listing that was purchased |

### Transaction Status

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `live` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Transaction is live (not cancelled). 0 = cancelled, 1 = live |

### Reviews

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `rating` | INT64 | `etsy_shard.shop_transaction_review` | Direct from shop_transaction_review | Review rating (1-5 stars) |
| `review` | STRING | `etsy_shard.shop_transaction_review` | Direct from shop_transaction_review | Review text left by buyer |

### Refunds

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_refund_amount` | NUMERIC | `transaction_mart.transactions_gms_by_trans` | listing_gms_gross - listing_gms_net | Listing refund amount (excludes gift wrap). **In USD dollars** |
| `refund_date` | DATE | `transaction_mart.transactions_gms` | MIN(acctg_date) where listing_gms_net < listing_gms_gross | Date of refund (earliest accounting date with refund) |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.transaction_post_purch` for:**
- **Transaction-level refund analysis**
- Review/rating analysis
- Cancelled transaction analysis
- **Item-level post-purchase metrics**

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt` (has post-purchase pre-joined)
- Transactions without issues → This table only includes transactions with post-purchase activity
- Receipt-level post-purchase → Use `transaction_mart.receipt_post_purch`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `transaction_obt` instead (post-purchase data already joined)
- `etsy_shard.shop_transaction_review` - Already incorporated
- `transaction_mart.transactions_gms_by_trans` - Used to build this, prefer OBT

**✅ Do join to these mart tables**:
- `transaction_mart.all_transactions` - Transaction details (join via transaction_id)
- `transaction_mart.transactions_gms_by_trans` - GMS data
- `transaction_mart.transactions_visits` - Visit attribution
- `transaction_mart.receipt_post_purch` - Receipt-level post-purchase (join via receipt_id)

### Performance Tips

1. **Only transactions with issues** - This table only includes transactions with: live=0 (cancelled), refunds, or reviews
2. **Join to all_transactions** - To get all transactions (including those without issues), LEFT JOIN from all_transactions
3. **Refund amount in dollars** - listing_refund_amount is NUMERIC in USD dollars (not cents)
4. **Use transaction_obt for most queries** - This table is for specific post-purchase analysis; transaction_obt has this data pre-joined

### Common Query Patterns

**Get transactions with refunds:**
```sql
SELECT
    transaction_id,
    receipt_id,
    listing_id,
    live,
    listing_refund_amount,
    refund_date,
    rating,
    review
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_post_purch`
WHERE transaction_id IN (1234, 5678)
```

**Refund analysis:**
```sql
SELECT
    DATE(t.creation_tsz) AS order_date,
    p.refund_date,
    DATE_DIFF(p.refund_date, DATE(t.creation_tsz), DAY) AS days_to_refund,
    COUNT(*) AS refund_count,
    SUM(p.listing_refund_amount) AS total_refund_amount,
    AVG(p.listing_refund_amount) AS avg_refund_amount
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_post_purch` p
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON p.transaction_id = t.transaction_id
WHERE p.listing_refund_amount > 0
  AND DATE(t.creation_tsz) BETWEEN '2026-03-01' AND '2026-03-19'
GROUP BY order_date, p.refund_date, days_to_refund
ORDER BY order_date, refund_date
```

**Review rating distribution:**
```sql
SELECT
    p.rating,
    COUNT(*) AS review_count,
    AVG(tg.trans_gms_net) AS avg_gms,
    SUM(CASE WHEN p.listing_refund_amount > 0 THEN 1 ELSE 0 END) AS refunded_count
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_post_purch` p
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON p.transaction_id = t.transaction_id
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans` tg
    ON p.transaction_id = tg.transaction_id
WHERE p.rating IS NOT NULL
  AND DATE(t.creation_tsz) = '2026-03-19'
GROUP BY p.rating
ORDER BY p.rating
```

**Cancelled transactions:**
```sql
SELECT
    t.market,
    COUNT(*) AS cancelled_count,
    SUM(tg.trans_gms_gross) AS cancelled_gms_gross,
    AVG(tg.trans_gms_gross) AS avg_cancelled_value
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_post_purch` p
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON p.transaction_id = t.transaction_id
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_gms_by_trans` tg
    ON p.transaction_id = tg.transaction_id
WHERE p.live = 0
  AND DATE(t.creation_tsz) = '2026-03-19'
GROUP BY t.market
ORDER BY cancelled_count DESC
```

**Transactions with negative reviews and refunds:**
```sql
SELECT
    p.transaction_id,
    p.listing_id,
    p.rating,
    LEFT(p.review, 100) AS review_snippet,
    p.listing_refund_amount,
    p.refund_date,
    DATE(t.creation_tsz) AS order_date
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_post_purch` p
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON p.transaction_id = t.transaction_id
WHERE p.rating <= 2
  AND p.listing_refund_amount > 0
  AND DATE(t.creation_tsz) = '2026-03-19'
ORDER BY p.listing_refund_amount DESC
LIMIT 100
```

---

## Important Notes

### Only Transactions with Post-Purchase Activity

**IMPORTANT**: This table only includes transactions that have at least one of:
- `live = 0` (transaction cancelled)
- `listing_gms_gross > listing_gms_net` (transaction refunded)
- `rating IS NOT NULL` (transaction reviewed)

The WHERE clause in the CREATE statement:
```sql
WHERE live = 0
   OR listing_gms_gross > listing_gms_net
   OR rating IS NOT NULL
```

To analyze all transactions (including those without issues), LEFT JOIN from `all_transactions` to this table.

### Refund Calculation

`listing_refund_amount` = listing_gms_gross - listing_gms_net:
- **Positive value**: Transaction had a refund
- **Zero**: No refund (but may be cancelled or have review)
- **Excludes gift wrap**: Only listing refund, not gift wrap refund

### Refund Date

`refund_date` is the earliest accounting date where the refund appears:
- Comes from `accounting_gms` table's `acctg_date`
- May be days/weeks after the original transaction date
- Only populated if `listing_gms_net < listing_gms_gross`

### Rating Scale

`rating` is on a 1-5 scale:
- 1 star = worst
- 5 stars = best
- NULL = no review

Not all transactions have reviews (buyers are not required to review).

### Review Text

`review` contains the actual text of the buyer's review:
- Can be NULL even if rating is present (buyer left rating but no text)
- May contain PII (buyer names mentioned, etc.) - handle appropriately
- Text length varies - can be very long

### Cancelled vs Refunded

- **live = 0**: Transaction was cancelled (never completed)
- **listing_refund_amount > 0**: Transaction was completed but later refunded

A transaction can be cancelled (live=0) AND have a refund if it was initially completed, refunded, then marked as cancelled.

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transaction_obt` - **PREFER THIS** - Complete transaction data with post-purchase pre-joined
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transactions_gms_by_trans` - GMS data (used to calculate listing_refund_amount)
- `transaction_mart.transactions_visits` - Visit attribution
- `transaction_mart.receipt_post_purch` - Receipt-level post-purchase (join via receipt_id)

### Receipt Tables (Join via receipt_id)
- `transaction_mart.all_receipts` - Receipt details
- `transaction_mart.receipts_gms` - Receipt-level GMS
- `transaction_mart.receipt_obt` - Receipt-level OBT

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.shop_transaction_review` - Reviews (rating and review already here)
- `transaction_mart.transactions_gms` - Detailed GMS with dates (refund_date derived from here)
