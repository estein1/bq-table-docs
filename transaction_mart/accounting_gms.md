---
id: accounting_gms
title: etsy-data-warehouse-prod.transaction_mart.accounting_gms
sidebar_label: accounting_gms
---

# accounting_gms

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.accounting_gms`

**Source Script:** `Rollups/auto/p1/daily/transaction_mart_*.sql`

**Purpose**: Accounting perspective GMS (Gross Merchandise Sales) for each transaction. Calculated from billing ledger, payments adjustments, and FRE (Financial Reporting Engine) data. **This is the accounting system of record for GMS**.

**Primary Key**: `transaction_id` + `date` + `creation_tsz`

**Partitioning**: Partitioned by `date`

**Clustering**: Clustered by `transaction_id`

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_id` | INT64 | Multiple sources | Direct from source tables | Unique transaction identifier. Part of composite key |
| `date` | DATE | Multiple sources | Date from source system (ledger date, event date, or transaction date) | Accounting date. **Used for partitioning**. Part of composite key |
| `creation_tsz` | TIMESTAMP | Multiple sources | Timestamp from source system | Transaction creation timestamp. Part of composite key |

### GMS Eligibility

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gms_eligible` | INT64 | Derived from transaction flags | Based on gms_eligible flag from transaction | Transaction is eligible for GMS (excludes gift cards, test users, cash, fraud, high-value cancelled) |

### GMS Metrics

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `gms_gross` | NUMERIC | `bill_mart_ledger.gms_billing`, `etsy_payments.fre_gms` | (transaction fees + gift wrap fees) / transaction rate | Total GMS gross. **In USD dollars** |
| `gms_net` | NUMERIC | `bill_mart_ledger.gms_billing`, `etsy_payments.fre_gms` | (fees - refunds) / transaction rate | Total GMS net (gross minus refunds). **In USD dollars** |
| `listing_gms_gross` | NUMERIC | `bill_mart_ledger.gms_billing`, `etsy_payments.fre_gms` | transaction fees / transaction rate | Listing GMS gross (excludes gift wrap). **In USD dollars** |
| `listing_gms_net` | NUMERIC | `bill_mart_ledger.gms_billing`, `etsy_payments.fre_gms` | (transaction fees - transaction refunds) / transaction rate | Listing GMS net (gross minus refunds, no gift wrap). **In USD dollars** |
| `giftwrap_gms_gross` | NUMERIC | `bill_mart_ledger.gms_billing`, `etsy_payments.fre_gms` | gift wrap fees / transaction rate | Gift wrap GMS gross. **In USD dollars** |
| `giftwrap_gms_net` | NUMERIC | `bill_mart_ledger.gms_billing`, `etsy_payments.fre_gms` | (gift wrap fees - gift wrap refunds) / transaction rate | Gift wrap GMS net (gross minus refunds). **In USD dollars** |
| `trans_correction_gms` | NUMERIC | `transaction_mart.correction_gms` | Transaction-level corrections | GMS corrections at transaction level |
| `rcpt_correction_gms` | NUMERIC | `transaction_mart.correction_gms` | Receipt-level corrections | GMS corrections at receipt level |

### Metadata

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `source` | STRING | Derived | 'ledger', 'ledger - pp/refund', 'non_accting', or 'fre_gms' | Source system for this GMS record |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.accounting_gms` for:**
- **Accounting reconciliation** - Official GMS from accounting perspective
- **Financial reporting** - Matches billing/ledger systems
- Comparing transaction-level vs accounting-level GMS
- **Preferred for finance/accounting analysis**

**Don't use this table for:**
- Standard transaction analysis → Use `transaction_mart.transactions_gms` (transaction-level GMS)
- Complete transaction analysis → Use `transaction_mart.transaction_obt`
- Receipt-level GMS → Use `transaction_mart.receipts_gms`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `transactions_gms` or `transaction_obt` instead
- `bill_mart_ledger.gms_billing` - Already incorporated here
- `etsy_payments.fre_gms` - Already incorporated here

**✅ Do join to these mart tables**:
- `transaction_mart.all_transactions` - Core transaction data (join via transaction_id)
- `transaction_mart.transactions_gms` - Compare transaction vs accounting GMS
- `transaction_mart.transactions_buyer` - Buyer data
- `transaction_mart.transactions_seller` - Seller data (or use accounting_seller)
- `transaction_mart.accounting_seller` - Accounting perspective seller location

### Performance Tips

1. **Partitioned by `date`** - Always filter on date for best performance
2. **Clustered by `transaction_id`** - Filter on transaction_id when possible
3. **Multiple sources** - Data comes from ledger (Etsy Payments), non-accounting (IPP/Pattern), and FRE (Financial Reporting Engine)
4. **Already in dollars** - All GMS fields are NUMERIC in USD dollars (not cents)
5. **Use for accounting** - This is the accounting system of record; use transactions_gms for operational reporting

### Common Query Patterns

**Get accounting GMS for transactions:**
```sql
SELECT
    transaction_id,
    date AS accounting_date,
    gms_gross,
    gms_net,
    listing_gms_net,
    giftwrap_gms_net,
    source
FROM `etsy-data-warehouse-prod.transaction_mart.accounting_gms`
WHERE transaction_id IN (1234, 5678)
```

**Compare transaction vs accounting GMS:**
```sql
SELECT
    t.transaction_id,
    t.trans_gms_net AS transaction_gms,
    a.gms_net AS accounting_gms,
    t.trans_gms_net - a.gms_net AS difference,
    a.source AS accounting_source
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_gms` t
LEFT JOIN `etsy-data-warehouse-prod.transaction_mart.accounting_gms` a
    ON t.transaction_id = a.transaction_id
WHERE t.trans_date = '2026-03-19'
  AND t.gms_eligible = 1
  AND ABS(t.trans_gms_net - COALESCE(a.gms_net, 0)) > 0.01
LIMIT 100
```

**GMS by source system:**
```sql
SELECT
    source,
    COUNT(*) AS transaction_count,
    SUM(gms_gross) AS total_gms_gross,
    SUM(gms_net) AS total_gms_net,
    SUM(gms_gross - gms_net) AS total_refunds
FROM `etsy-data-warehouse-prod.transaction_mart.accounting_gms`
WHERE date = '2026-03-19'
GROUP BY source
ORDER BY total_gms_net DESC
```

**Daily accounting GMS:**
```sql
SELECT
    date AS accounting_date,
    SUM(gms_gross) AS total_gms_gross,
    SUM(gms_net) AS total_gms_net,
    SUM(listing_gms_net) AS listing_gms,
    SUM(giftwrap_gms_net) AS giftwrap_gms
FROM `etsy-data-warehouse-prod.transaction_mart.accounting_gms`
WHERE date BETWEEN '2026-03-01' AND '2026-03-19'
GROUP BY date
ORDER BY date
```

---

## Important Notes

### Source Systems

This table combines data from multiple source systems:

1. **'ledger'** - Historical data from `bill_mart_ledger.gms_billing` (prior to Oct 1, 2025)
   - Etsy Payments transactions
   - Calculated from transaction fees and refunds

2. **'ledger - pp/refund'** - Ledger data for PayPal and refunds (after Oct 1, 2025)
   - PayPal transactions
   - Refunds from historical transactions

3. **'non_accting'** - Non-accounting systems (IPP and Pattern)
   - IPP (In-Person Payments) via `etsy_payments.payments_adjustments`
   - Pattern transactions
   - Square payments

4. **'fre_gms'** - Financial Reporting Engine (after Oct 1, 2025)
   - Official financial reporting system
   - Uses `etsy_payments.fre_gms` table

### Date Logic

The `date` field comes from different sources depending on the system:
- **Ledger**: billing date from gms_billing
- **FRE**: reference_update_date (when payment status updated to posted)
- **Adjustments**: adjustment create_date
- **Non-accounting**: transaction date

This can cause the same transaction to have different accounting dates in different systems.

### Switch Date

October 1, 2025 (`switch_date`) is when the system transitioned from legacy billing to FRE:
- Data before Oct 1, 2025: From ledger and static historical table
- Data after Oct 1, 2025: From FRE + ledger (PayPal/refunds) + non-accounting (IPP/Pattern)

### Transaction vs Accounting GMS

Key differences between `transactions_gms` and `accounting_gms`:
- **transactions_gms**: Operational view, calculated from transaction prices
- **accounting_gms**: Financial view, calculated from billing/ledger systems
- **Timing**: May record on different dates
- **Source**: accounting_gms is from billing systems (authoritative for finance)
- **Rounding**: Both use NUMERIC but may have slight differences

### Gift Wrap Allocation

Gift wrap fees are allocated to the first transaction (transaction_seq = 1) in each receipt.

### Corrections

`trans_correction_gms` and `rcpt_correction_gms` capture adjustments from the `correction_gms` table - used to correct historical GMS errors.

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transactions_gms` - Transaction-level GMS (operational view)
- `transaction_mart.accounting_gms_by_trans` - Alternative accounting GMS view
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transactions_buyer` - Buyer data
- `transaction_mart.transactions_seller` - Transaction-time seller location
- `transaction_mart.accounting_seller` - Accounting-perspective seller location
- `transaction_mart.all_transactions_categories` - Category data
- `transaction_mart.transaction_obt` - Transaction-level OBT
- `transaction_mart.transaction_post_purch` - Post-purchase metrics

### Receipt Tables
- `transaction_mart.receipts_gms` - Receipt-level GMS aggregations
- `transaction_mart.all_receipts` - Receipt-level details
- `transaction_mart.receipt_obt` - Receipt-level OBT

### Other Tables
- `transaction_mart.correction_gms` - GMS corrections/adjustments
- `transaction_mart.transactions_gms_by_trans` - Alternative transaction GMS view

### Source Tables (Avoid - Data Already in Mart)
- `bill_mart_ledger.gms_billing` - Billing ledger (already incorporated)
- `etsy_payments.fre_gms` - Financial Reporting Engine GMS (already incorporated)
- `etsy_payments.payments_adjustments` - Payment adjustments (already incorporated for IPP)
- `etsy_payments.payments_adjustments_items` - Adjustment items (already incorporated)
- `etsy-data-warehouse-prod.static.gms_billing_prior_10012025` - Historical static data (already incorporated)
