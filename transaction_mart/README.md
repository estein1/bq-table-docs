# Transaction Mart Schema

The `transaction_mart` schema contains **denormalized, analysis-ready tables** for all Etsy transaction and receipt data. These tables join data from multiple source systems to provide complete views of purchases, revenue, and customer behavior.

## 🎯 Quick Start

**For most queries, use the OBT (One Big Table) tables:**

- **Receipt-level analysis** → [`receipt_obt`](receipt_obt.md) - Complete receipt data with GMS, visits, post-purchase
- **Transaction-level analysis** → [`transaction_obt`](transaction_obt.md) - Complete transaction data with GMS, visits, reviews

These tables have everything pre-joined and are optimized for performance.

## 📊 Table Categories

### One Big Tables (OBT) - Start Here

| Table | Purpose | When to Use |
|-------|---------|-------------|
| [`receipt_obt`](receipt_obt.md) | Complete receipt-level data | **Any receipt/order analysis** |
| [`transaction_obt`](transaction_obt.md) | Complete transaction-level data | **Any transaction/line-item analysis** |

### Foundation Tables

| Table | Purpose | Primary Key |
|-------|---------|-------------|
| [`all_receipts`](all_receipts.md) | Core receipt data | `receipt_id` |
| [`all_transactions`](all_transactions.md) | Core transaction data | `transaction_id` |

### GMS (Revenue) Tables

| Table | Purpose | GMS Source |
|-------|---------|------------|
| [`receipts_gms`](receipts_gms.md) | Receipt-level GMS with buyer/seller | Operational + Accounting |
| [`transactions_gms`](transactions_gms.md) | Transaction GMS with demographics | **Accounting** GMS in main columns |
| [`transactions_gms_by_trans`](transactions_gms_by_trans.md) | Transaction GMS with demographics | **Operational** GMS in main columns |
| [`accounting_gms`](accounting_gms.md) | Accounting GMS by date/source | Accounting (ledger/FRE) |
| [`accounting_gms_by_trans`](accounting_gms_by_trans.md) | Simplified accounting GMS | Accounting (aggregated) |

### Enrichment Tables

| Table | Purpose | Join Key |
|-------|---------|----------|
| [`transactions_buyer`](transactions_buyer.md) | Buyer location & payment | `transaction_id` |
| [`transactions_seller`](transactions_seller.md) | Seller location at transaction time | `transaction_id` |
| [`accounting_seller`](accounting_seller.md) | Seller location at accounting time | `transaction_id` + `creation_tsz` |
| [`all_transactions_categories`](all_transactions_categories.md) | Category & taxonomy | `transaction_id` |

### Visit Attribution Tables

| Table | Purpose | Join Key |
|-------|---------|----------|
| [`transactions_visits`](transactions_visits.md) | Visit attribution per transaction | `transaction_id` |
| [`receipts_visits`](receipts_visits.md) | Visit attribution per receipt | `receipt_id` |

### Post-Purchase Tables

| Table | Purpose | Join Key |
|-------|---------|----------|
| [`transaction_post_purch`](transaction_post_purch.md) | Transaction refunds & reviews | `transaction_id` |
| [`receipt_post_purch`](receipt_post_purch.md) | Receipt returns, cases & help requests | `receipt_id` |

## 💡 Common Patterns

### Daily Revenue Analysis

```sql
-- Use receipt_obt for receipt-level revenue
SELECT
    DATE(creation_tsz) AS order_date,
    market,
    COUNT(*) AS receipt_count,
    SUM(trans_gms_net) AS total_gms,
    AVG(trans_gms_net) AS avg_order_value
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND is_test_buyer = 0
  AND is_test_seller = 0
GROUP BY order_date, market
```

### Category Performance

```sql
-- Use transaction_obt for transaction-level category analysis
SELECT
    new_category,
    COUNT(*) AS transaction_count,
    SUM(quantity) AS items_sold,
    SUM(trans_gms_net) AS category_gms
FROM `etsy-data-warehouse-prod.transaction_mart.transaction_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND live = 1
  AND is_test_buyer = 0
  AND gms_eligible = 1
GROUP BY new_category
ORDER BY category_gms DESC
```

### Visit Attribution

```sql
-- receipt_obt has visit data pre-joined
SELECT
    top_channel,
    platform_device,
    COUNT(*) AS receipt_count,
    SUM(trans_gms_net) AS total_gms
FROM `etsy-data-warehouse-prod.transaction_mart.receipt_obt`
WHERE DATE(creation_tsz) = '2026-03-19'
  AND visit_id IS NOT NULL
  AND is_test_buyer = 0
GROUP BY top_channel, platform_device
ORDER BY total_gms DESC
```

## 🔑 Key Concepts

### GMS Eligibility

Transactions are **excluded from GMS** if:
- `is_gift_card = 1` - Gift card purchases
- `is_test_seller = 1` or `is_test_buyer = 1` - Test users
- `is_cash = 1` - Cash payments
- `is_fraud = 1` - Fraudulent transactions
- `is_highval = 1 AND live = 0` - High-value cancelled (>$10K)

### Transaction vs Accounting GMS

- **trans_gms_*** - Operational GMS calculated from transaction prices
- **gms_*** (no prefix) - Accounting GMS from billing systems
- **For most reporting, use `trans_gms_net`**

### Money Fields

**All transaction_mart money fields are in USD dollars** (NUMERIC type), unlike listing_mart which uses cents (INT64).

### OBT Tables

**One Big Table (OBT)** tables have all related data pre-joined:
- Faster queries (no joins needed)
- Easier to use (all fields in one place)
- Better performance (optimized structure)

## 📁 Schema Overview

```
transaction_mart/
├── OBT Tables (Use These!)
│   ├── receipt_obt          # Complete receipt data
│   └── transaction_obt      # Complete transaction data
│
├── Foundation
│   ├── all_receipts         # Core receipt data
│   └── all_transactions     # Core transaction data
│
├── GMS (Revenue)
│   ├── receipts_gms         # Receipt GMS
│   ├── transactions_gms     # Transaction GMS (accounting focus)
│   ├── transactions_gms_by_trans  # Transaction GMS (operational focus)
│   ├── accounting_gms       # Accounting GMS by date
│   └── accounting_gms_by_trans    # Accounting GMS simplified
│
├── Enrichment
│   ├── transactions_buyer   # Buyer demographics
│   ├── transactions_seller  # Seller location
│   ├── accounting_seller    # Seller location (accounting view)
│   └── all_transactions_categories  # Categories & taxonomy
│
├── Attribution
│   ├── transactions_visits  # Transaction visit attribution
│   └── receipts_visits      # Receipt visit attribution
│
└── Post-Purchase
    ├── transaction_post_purch  # Refunds & reviews
    └── receipt_post_purch      # Returns & cases
```

## ⚡ Performance Tips

1. **Always filter on dates** - Tables are partitioned by date
2. **Filter test users** - `is_test_buyer = 0 AND is_test_seller = 0`
3. **Use OBT tables** - Avoid joining component tables yourself
4. **Filter by live** - `live = 1` (transactions) or `receipt_live = 1` (receipts)
5. **Use trans_gms_net** - Standard metric for operational revenue

## 🆘 Getting Help

- For table-specific details, click the table name links above
- For data quality issues: data-marts@etsy.pagerduty.com
- For schema questions: estein@etsy.com

---

**Last Updated**: 2026-03-19
**Schema Owner**: data-marts@etsy.pagerduty.com
