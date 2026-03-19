---
id: accounting_seller
title: transaction_mart.accounting_seller
sidebar_label: accounting_seller
---

# transaction_mart.accounting_seller

## Table Overview

**Purpose**: Seller location information from accounting perspective. One row per accounting_gms record with seller's location based on accounting creation timestamp (not transaction timestamp).

**Primary Key**: `transaction_id` + `creation_tsz`

**Clustering**: Not specified

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_id` | INT64 | `transaction_mart.accounting_gms` | From accounting_gms | Unique transaction identifier. Part of composite key |
| `creation_tsz` | TIMESTAMP | `transaction_mart.accounting_gms` | From accounting_gms | Accounting creation timestamp (not transaction timestamp). Part of composite key |

### Seller Location

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `seller_country_id` | INT64 | `materialized.seller_location_log` | coalesce(country_id, 0) | Seller's country ID at accounting time. Defaults to 0 if missing. **PII** |
| `seller_country` | STRING | `materialized.seller_location_log` | coalesce(country, 'UNKNOWN') | Seller's country code at accounting time. Defaults to 'UNKNOWN' if missing. **PII** |
| `seller_country_name` | STRING | `materialized.seller_location_log` | coalesce(country_name, 'UNKNOWN') | Seller's country name at accounting time. Defaults to 'UNKNOWN' if missing. **PII** |
| `seller_state` | STRING | `materialized.seller_location_log` | coalesce(state, ' ') | Seller's state/province at accounting time. Defaults to single space if missing. **PII** |
| `seller_zip` | STRING | `materialized.seller_location_log` | coalesce(clean_zip_code(postal_code, country_id), '') | Seller's postal code at accounting time, cleaned via UDF. Defaults to empty string if missing. **PII** |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.accounting_seller` for:**
- **Accounting reconciliation** - Seller location from accounting perspective
- **Financial reporting** - Matches accounting_gms timing
- Comparing transaction-time vs accounting-time seller location
- **Join with accounting_gms** - Same creation_tsz key

**Don't use this table for:**
- Transaction-time seller location → Use `transaction_mart.transactions_seller`
- Current seller location → Use `user_mart` or `shop_mart` tables
- Complete transaction analysis → Use `transaction_mart.transaction_obt`

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete analysis, use `transaction_obt` instead
- `materialized.seller_location_log` - Already incorporated with time-based join

**✅ Do join to these mart tables**:
- `transaction_mart.accounting_gms` - Accounting GMS (join via transaction_id + creation_tsz)
- `transaction_mart.all_transactions` - Core transaction data (join via transaction_id only)
- `transaction_mart.transactions_seller` - Compare transaction-time vs accounting-time location
- `transaction_mart.transactions_buyer` - Buyer data

### Performance Tips

1. **Time-based location join** - Seller location matched based on accounting creation_tsz falling between seller_location_log dates
2. **Different from transactions_seller** - Uses accounting timestamp, not transaction timestamp
3. **Composite key** - Join to accounting_gms requires both transaction_id AND creation_tsz
4. **PII fields** - All seller location fields contain PII
5. **Cleaned zip codes** - seller_zip uses clean_zip_code UDF for standardization

### Common Query Patterns

**Get accounting seller location:**
```sql
SELECT
    a.transaction_id,
    a.creation_tsz AS accounting_timestamp,
    s.seller_country_name,
    s.seller_state,
    s.seller_zip
FROM `etsy-data-warehouse-prod.transaction_mart.accounting_gms` a
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.accounting_seller` s
    ON a.transaction_id = s.transaction_id
    AND a.creation_tsz = s.creation_tsz
WHERE a.date = '2026-03-19'
LIMIT 100
```

**Compare transaction-time vs accounting-time seller location:**
```sql
SELECT
    t.transaction_id,
    ts.seller_country_name AS transaction_location,
    s.seller_country_name AS accounting_location,
    CASE
        WHEN ts.seller_country = s.seller_country THEN 'Same'
        ELSE 'Different'
    END AS location_match
FROM `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_seller` ts
    ON t.transaction_id = ts.transaction_id
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.accounting_gms` a
    ON t.transaction_id = a.transaction_id
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.accounting_seller` s
    ON a.transaction_id = s.transaction_id
    AND a.creation_tsz = s.creation_tsz
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
LIMIT 1000
```

**Accounting GMS by seller country:**
```sql
SELECT
    s.seller_country_name,
    COUNT(DISTINCT a.transaction_id) AS transaction_count,
    SUM(a.gms_net) AS total_accounting_gms
FROM `etsy-data-warehouse-prod.transaction_mart.accounting_seller` s
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.accounting_gms` a
    ON s.transaction_id = a.transaction_id
    AND s.creation_tsz = a.creation_tsz
WHERE a.date = '2026-03-19'
GROUP BY s.seller_country_name
ORDER BY total_accounting_gms DESC
LIMIT 20
```

---

## Important Notes

### Point-in-Time Location (Accounting Perspective)

Seller location is captured **at the accounting creation timestamp**, not the transaction timestamp:

- **transactions_seller**: Uses transaction.creation_tsz
- **accounting_seller**: Uses accounting_gms.creation_tsz

These can differ because accounting records may be created days/weeks after the transaction (e.g., for refunds, adjustments).

The join logic:
```sql
accounting_gms.creation_tsz BETWEEN seller_location.creation_tsz AND seller_location.thru_creation_tsz
```

### Why Two Seller Location Tables?

| Table | Time Reference | Use Case |
|-------|----------------|----------|
| `transactions_seller` | Transaction creation time | Operational analysis, actual transaction location |
| `accounting_seller` | Accounting record creation time | Financial reporting, accounting reconciliation |

For most analyses, use `transactions_seller`. Use `accounting_seller` only when you need seller location to match accounting timing exactly.

### PII Data

All seller location fields contain Personally Identifiable Information (PII):
- `seller_country_id`
- `seller_country`
- `seller_country_name`
- `seller_state`
- `seller_zip`

Use appropriate data governance and access controls when querying these fields.

### Postal Code Cleaning

The `seller_zip` field uses the `clean_zip_code` UDF which standardizes postal codes based on country format:
- Removes invalid characters
- Formats according to country standards
- Returns empty string if postal code is invalid

### Default Values

Fields use specific defaults when data is missing:
- `seller_country_id`: 0
- `seller_country`: 'UNKNOWN'
- `seller_country_name`: 'UNKNOWN'
- `seller_state`: ' ' (single space)
- `seller_zip`: '' (empty string)

---

## Related Tables

### Transaction Mart Tables
- `transaction_mart.accounting_gms` - **Primary join target** - Accounting GMS (join via transaction_id + creation_tsz)
- `transaction_mart.accounting_gms_by_trans` - Simplified accounting GMS (join via transaction_id only)
- `transaction_mart.transactions_seller` - **Compare** - Transaction-time seller location
- `transaction_mart.all_transactions` - Core transaction data (join via transaction_id)
- `transaction_mart.transactions_gms` - Operational GMS
- `transaction_mart.transactions_buyer` - Buyer demographics
- `transaction_mart.all_transactions_categories` - Category data
- `transaction_mart.transaction_obt` - Complete transaction data
- `transaction_mart.transaction_post_purch` - Post-purchase metrics

### Receipt Tables
- `transaction_mart.receipts_gms` - Receipt-level GMS
- `transaction_mart.all_receipts` - Receipt details
- `transaction_mart.receipt_obt` - Receipt-level OBT

### Other Tables
- `transaction_mart.correction_gms` - GMS corrections

### Source Tables (Avoid - Data Already in Mart)
- `materialized.seller_location_log` - Historical seller locations (already incorporated with time-based join)
- `etsy_shard.shop_locations` - Raw shop location data (use seller_location_log instead)
