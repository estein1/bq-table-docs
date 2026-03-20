---
id: accounting_gms_by_trans
title: etsy-data-warehouse-prod.transaction_mart.accounting_gms_by_trans
sidebar_label: accounting_gms_by_trans
---

# accounting_gms_by_trans

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.accounting_gms_by_trans`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: Simplified accounting GMS view with one row per transaction. Aggregates multiple accounting_gms records (which can have different dates/timestamps) into a single row per transaction_id.

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
| `transaction_id` | INT64 | `transaction_mart.accounting_gms` | Aggregated from accounting_gms | Unique transaction identifier. Primary key |

### GMS Metrics

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | SUM(gms_gross) grouped by transaction_id | Total accounting GMS gross across all records for this transaction. **In USD dollars** |
| `gms_net` | NUMERIC | `transaction_mart.accounting_gms` | SUM(gms_net) grouped by transaction_id | Total accounting GMS net across all records for this transaction. **In USD dollars** |
| `listing_gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | SUM(listing_gms_gross) grouped by transaction_id | Total listing GMS gross (excludes gift wrap). **In USD dollars** |
| `listing_gms_net` | NUMERIC | `transaction_mart.accounting_gms` | SUM(listing_gms_net) grouped by transaction_id | Total listing GMS net (excludes gift wrap). **In USD dollars** |
| `giftwrap_gms_gross` | NUMERIC | `transaction_mart.accounting_gms` | SUM(giftwrap_gms_gross) grouped by transaction_id | Total gift wrap GMS gross. **In USD dollars** |
| `giftwrap_gms_net` | NUMERIC | `transaction_mart.accounting_gms` | SUM(giftwrap_gms_net) grouped by transaction_id | Total gift wrap GMS net. **In USD dollars** |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.accounting_gms_by_trans` for:**
- **Simple transaction-level accounting GMS** - One row per transaction
- Joining to transaction tables where you need accounting GMS but don't care about multiple date records
- **Easier joins** - No need to worry about date/timestamp in join keys

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt`
- Daily accounting GMS breakdowns → Use `transaction_mart.accounting_gms` (has date field)
- Transaction-level operational GMS → Use `transaction_mart.transactions_gms`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `transaction_obt` instead
- `transaction_mart.accounting_gms` - This table IS that data aggregated

**✅ Do join to these mart tables**:
- `transaction_mart.all_transactions` - Core transaction data (join via transaction_id)
- `transaction_mart.transactions_gms` - Compare operational vs accounting GMS
- `transaction_mart.transactions_buyer` - Buyer data
- `transaction_mart.transactions_seller` - Seller data
- `transaction_mart.all_transactions_categories` - Category data

### Performance Tips

1. **Simple join key** - Just transaction_id (no date/timestamp needed)
2. **Pre-aggregated** - Already summed across multiple accounting records
3. **Already in dollars** - All GMS fields are NUMERIC in USD dollars (not cents)
4. **Use for simplicity** - When you don't need date breakdowns from accounting_gms
5. **May differ from transactions_gms** - Accounting vs operational perspectives can have differences

### Common Query Patterns

**Get accounting GMS for transactions:**
```sql
SELECT
    transaction_id,
    gms_gross,
    gms_net,
    listing_gms_net,
    giftwrap_gms_net
FROM `etsy-data-warehouse-prod.transaction_mart.accounting_gms_by_trans`
WHERE transaction_id IN (1234, 5678)
```

**Compare transaction vs accounting GMS:**
```sql
SELECT
    t.transaction_id,
    t.trans_gms_net AS operational_gms,
    a.gms_net AS accounting_gms,
    t.trans_gms_net - a.gms_net AS difference,
    CASE
        WHEN ABS(t.trans_gms_net - a.gms_net) < 0.01 THEN 'Match'
        WHEN ABS(t.trans_gms_net - a.gms_net) < 1.00 THEN 'Small Diff'
        ELSE 'Large Diff'
    END AS variance_category
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms` t
LEFT JOIN `etsy-data-warehouse-prod.transaction_mart.accounting_gms_by_trans` a
    ON t.transaction_id = a.transaction_id
WHERE t.trans_date = '2026-03-19'
  AND t.gms_eligible = 1
```

**Accounting GMS with buyer country:**
```sql
SELECT
    tb.buyer_country_name,
    COUNT(*) AS transaction_count,
    SUM(a.gms_net) AS total_accounting_gms,
    AVG(a.gms_net) AS avg_gms_per_transaction
FROM `etsy-data-warehouse-prod.transaction_mart.accounting_gms_by_trans` a
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_buyer` tb
    ON a.transaction_id = tb.transaction_id
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON a.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
GROUP BY tb.buyer_country_name
ORDER BY total_accounting_gms DESC
LIMIT 20
```

---

## Important Notes

### Aggregation Logic

This table aggregates `accounting_gms` which can have multiple records per transaction:
- **Multiple dates**: Same transaction can appear on different accounting dates (original sale, refunds, adjustments)
- **Multiple sources**: Same transaction may have records from ledger, FRE, and non-accounting systems
- **SUM across all**: This table sums all GMS values across all date records

### When Transactions Have Multiple Records

A single transaction can have multiple `accounting_gms` records because:
1. Original sale on one date
2. Refund on a later date
3. Corrections/adjustments
4. Different source systems logging on different dates

This table adds them all together, so:
- `gms_gross` includes all positive charges
- `gms_net` = gross minus all refunds/corrections
- Final net can be 0 (fully refunded) or negative (over-refunded due to corrections)

### Comparison to accounting_gms

| Feature | accounting_gms | accounting_gms_by_trans |
|---------|---------------|------------------------|
| Rows per transaction | Multiple (one per date/source) | One |
| Has date field | Yes | No |
| Has source field | Yes | No |
| Join complexity | Need transaction_id + date/timestamp | Just transaction_id |
| Use case | Daily accounting analysis | Simplified transaction joins |

### Already in Dollars

All GMS fields are NUMERIC in USD dollars (not cents). No conversion needed for display.

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.accounting_gms` - **Source table** - Detailed accounting GMS with dates and sources
- `transaction_mart.transactions_gms` - Operational GMS (compare to accounting)
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transactions_buyer` - Buyer demographics
- `transaction_mart.transactions_seller` - Seller location at transaction time
- `transaction_mart.accounting_seller` - Seller location from accounting perspective
- `transaction_mart.all_transactions_categories` - Category taxonomy
- `transaction_mart.transaction_obt` - Complete transaction data
- `transaction_mart.transaction_post_purch` - Post-purchase metrics

### Receipt Tables
- `transaction_mart.receipts_gms` - Receipt-level GMS
- `transaction_mart.all_receipts` - Receipt details
- `transaction_mart.receipt_obt` - Receipt-level OBT

### Other Tables
- `transaction_mart.transactions_gms_by_trans` - Alternative transaction GMS view (operational)
- `transaction_mart.correction_gms` - GMS corrections

### Source Tables (Avoid - Data Already in Mart)
- `transaction_mart.accounting_gms` - This IS that table aggregated by transaction_id
