---
id: receipt_post_purch
title: etsy-data-warehouse-prod.transaction_mart.receipt_post_purch
sidebar_label: receipt_post_purch
---

# receipt_post_purch

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.receipt_post_purch`

**Source Script:** `Rollups/auto/p1/daily/transactions-mart/trans_mart_by_date.sql`

**Purpose**: Post-purchase metrics for receipts. One row per receipt that has post-purchase activity (refunds, returns, cases, or help requests). **Excludes receipts with no post-purchase activity**.

**Primary Key**: `receipt_id`

**Clustering**: Not specified

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `receipt_id` | INT64 | `transaction_mart.all_receipts` | Direct from all_receipts | Unique receipt identifier. Primary key |

### Receipt Metrics

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transactions_in_receipt` | INT64 | Calculated | COUNT of transactions for this receipt_id | Number of transactions (line items) in this receipt |

### Refunds & Returns

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `receipt_refund` | NUMERIC | `transaction_mart.receipts_gms` | gms_gross - gms_net | Total refund amount for receipt. **In USD dollars** |
| `receipt_return` | INT64 | `etsy_shard.shop_return_acceptance_rate` | 1 if receipt has return performed, else 0 | Receipt had a return performed |

### Customer Service

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `help_with_order` | INT64 | `etsy_shard.order_help_request` | 1 if receipt has help request with conversation_id > 0, else 0 | Buyer requested help with order |

### Cases

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `non_delivery_cases` | INT64 | `etsy_index.user_case_index` | SUM of cases where type='non_delivery' | Number of non-delivery cases for this receipt |
| `not_as_described_cases` | INT64 | `etsy_index.user_case_index` | SUM of cases where type='not_as_described' | Number of not-as-described cases for this receipt |
| `arrived_damaged_cases` | INT64 | `etsy_index.user_case_index` | SUM of cases where type='arrived_damaged' | Number of arrived-damaged cases for this receipt |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.receipt_post_purch` for:**
- **Post-purchase issue analysis** - Returns, refunds, cases
- Customer service metrics (help requests, cases)
- Refund analysis by receipt
- **Quality/satisfaction metrics** - Identify problematic receipts

**Don't use this table for:**
- Complete receipt analysis → Use `transaction_mart.receipt_obt` (has post-purchase pre-joined)
- Receipts without issues → This table only includes receipts with post-purchase activity
- Transaction-level refunds → Use `transaction_mart.transaction_post_purch`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `receipt_obt` instead (post-purchase data already joined)
- `etsy_shard.shop_return_acceptance_rate` - Already incorporated
- `etsy_index.user_case_index` - Already aggregated here
- `etsy_shard.order_help_request` - Already incorporated

**✅ Do join to these mart tables**:
- `transaction_mart.all_receipts` - Receipt details (join via receipt_id)
- `transaction_mart.receipts_gms` - GMS data (join via receipt_id)
- `transaction_mart.receipts_visits` - Visit attribution
- `transaction_mart.transaction_post_purch` - Transaction-level post-purchase metrics

### Performance Tips

1. **Only receipts with issues** - This table only includes receipts that have at least one of: refund, return, case, or help request
2. **Join to all_receipts** - To get all receipts (including those without issues), LEFT JOIN from all_receipts
3. **Refund amount in dollars** - receipt_refund is NUMERIC in USD dollars (not cents)
4. **Use receipt_obt for most queries** - This table is for specific post-purchase analysis; receipt_obt has this data pre-joined

### Common Query Patterns

**Get receipts with post-purchase issues:**
```sql
SELECT
    receipt_id,
    transactions_in_receipt,
    receipt_refund,
    receipt_return,
    help_with_order,
    non_delivery_cases,
    not_as_described_cases,
    arrived_damaged_cases
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_post_purch`
WHERE receipt_id IN (1234, 5678)
```

**Receipts with refunds:**
```sql
SELECT
    r.receipt_id,
    DATE(r.creation_tsz) AS order_date,
    r.market,
    p.receipt_refund,
    p.transactions_in_receipt,
    rg.trans_gms_gross,
    p.receipt_refund / NULLIF(rg.trans_gms_gross, 0) AS refund_pct
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_post_purch` p
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_receipts` r
    ON p.receipt_id = r.receipt_id
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.receipts_gms` rg
    ON p.receipt_id = rg.receipt_id
WHERE p.receipt_refund > 0
  AND DATE(r.creation_tsz) = '2026-03-19'
ORDER BY p.receipt_refund DESC
LIMIT 100
```

**Post-purchase issue rates:**
```sql
WITH all_receipts_cnt AS (
    SELECT COUNT(*) AS total_receipts
    FROM `etsy-data-warehouse-prod.transaction_mart.all_receipts`
    WHERE DATE(creation_tsz) = '2026-03-19'
)
SELECT
    'Return' AS issue_type,
    SUM(receipt_return) AS issue_count,
    total_receipts,
    SUM(receipt_return) / total_receipts AS issue_rate
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_post_purch` p
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_receipts` r
    ON p.receipt_id = r.receipt_id
CROSS JOIN all_receipts_cnt
WHERE DATE(r.creation_tsz) = '2026-03-19'
GROUP BY total_receipts

UNION ALL

SELECT
    'Help Request' AS issue_type,
    SUM(help_with_order) AS issue_count,
    total_receipts,
    SUM(help_with_order) / total_receipts AS issue_rate
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_post_purch` p
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_receipts` r
    ON p.receipt_id = r.receipt_id
CROSS JOIN all_receipts_cnt
WHERE DATE(r.creation_tsz) = '2026-03-19'
GROUP BY total_receipts
```

**Cases by type:**
```sql
SELECT
    r.market,
    COUNT(*) AS receipts_with_cases,
    SUM(p.non_delivery_cases) AS total_non_delivery,
    SUM(p.not_as_described_cases) AS total_not_as_described,
    SUM(p.arrived_damaged_cases) AS total_arrived_damaged
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_post_purch` p
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_receipts` r
    ON p.receipt_id = r.receipt_id
WHERE DATE(r.creation_tsz) = '2026-03-19'
  AND (p.non_delivery_cases > 0
       OR p.not_as_described_cases > 0
       OR p.arrived_damaged_cases > 0)
GROUP BY r.market
ORDER BY receipts_with_cases DESC
```

---

## Important Notes

### Only Receipts with Post-Purchase Activity

**IMPORTANT**: This table only includes receipts that have at least one of:
- A case (non_delivery, not_as_described, or arrived_damaged)
- A help request (conversation_id > 0)
- A return (was_return_performed = 1)

The WHERE clause in the CREATE statement:
```sql
WHERE cases.receipt_id IS NOT NULL
   OR help_with_order.receipt_id IS NOT NULL
   OR return.receipt_id IS NOT NULL
```

To analyze all receipts (including those without issues), LEFT JOIN from `all_receipts` to this table.

### Refund Calculation

`receipt_refund` = gms_gross - gms_net from receipts_gms table:
- **Positive value**: Receipt had refunds
- **Zero**: No refunds (but may have other post-purchase activity)
- **Negative**: Unusual, may indicate data issue

### Case Counts

Case count fields can be > 1 if multiple cases of the same type were opened for the same receipt:
- `non_delivery_cases`: Sum of all non-delivery cases
- `not_as_described_cases`: Sum of all not-as-described cases
- `arrived_damaged_cases`: Sum of all arrived-damaged cases

A receipt can have multiple types of cases (e.g., both non-delivery AND not-as-described).

### Help with Order

`help_with_order = 1` means the buyer initiated a help request that resulted in a conversation (`conversation_id > 0`). This indicates the buyer contacted support about the order.

### Return vs Refund

- **Receipt return**: Physical return of item(s) was performed
- **Receipt refund**: Money was refunded (may or may not involve physical return)

A receipt can have a refund without a return (e.g., seller refunded without requiring return).

---

## Related Tables

### Transaction Mart Tables (Join on receipt_id)
- `transaction_mart.receipt_obt` - **PREFER THIS** - Complete receipt data with post-purchase pre-joined
- `transaction_mart.all_receipts` - Basic receipt data
- `transaction_mart.receipts_gms` - GMS data (used to calculate receipt_refund)
- `transaction_mart.receipts_visits` - Visit attribution
- `transaction_mart.all_transactions` - Transaction-level details
- `transaction_mart.transaction_post_purch` - Transaction-level post-purchase (join via receipt_id)

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.shop_return_acceptance_rate` - Returns data (receipt_return already calculated)
- `etsy_index.user_case_index` - Cases data (already aggregated here)
- `etsy_shard.order_help_request` - Help requests (help_with_order already calculated)
