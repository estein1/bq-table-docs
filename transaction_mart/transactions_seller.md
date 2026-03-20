---
id: transactions_seller
title: etsy-data-warehouse-prod.transaction_mart.transactions_seller
sidebar_label: transactions_seller
---

# transactions_seller

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.transaction_mart.transactions_seller`

**Source Script:** `Rollups/auto/p1/daily/transactions-mart/trans_mart_by_date.sql`

**Purpose**: Seller location information for each transaction. One row per transaction with seller's location at the time of the transaction.

**Primary Key**: `transaction_id`

**Clustering**: Not specified (built as temporary table first)

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Unique transaction identifier. Primary key |
| `receipt_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Parent receipt ID |
| `seller_user_id` | INT64 | `transaction_mart.all_transactions` | Direct from all_transactions | Seller's user ID |

### Seller Location

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `seller_country_id` | INT64 | `materialized.seller_location_log` | coalesce(country_id, 0) | Seller's country ID at transaction time. Defaults to 0 if missing. **PII** |
| `seller_country` | STRING | `materialized.seller_location_log` | coalesce(country, 'UNKNOWN') | Seller's country code at transaction time. Defaults to 'UNKNOWN' if missing. **PII** |
| `seller_country_name` | STRING | `materialized.seller_location_log` | coalesce(country_name, 'UNKNOWN') | Seller's country name at transaction time. Defaults to 'UNKNOWN' if missing. **PII** |
| `seller_state` | STRING | `materialized.seller_location_log` | coalesce(state, '  ') | Seller's state/province at transaction time. Defaults to 2 spaces if missing. **PII** |
| `seller_zip` | STRING | `materialized.seller_location_log` | coalesce(clean_zip_code(postal_code, country_id), '') | Seller's postal code at transaction time, cleaned via UDF. Defaults to empty string if missing. **PII** |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.transactions_seller` for:**
- Transaction-level seller demographics
- Seller location analysis at transaction time
- Cross-border transaction analysis (buyer country vs seller country)
- **Enriching transaction queries** with seller location data

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt` (has seller data pre-joined)
- Current seller location → Use `user_mart` or `shop_mart` tables
- Seller shop details → Use `shop_mart` tables

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete transaction analysis, use `transaction_obt` instead (seller data already joined)
- `materialized.seller_location_log` - Historical seller locations (already incorporated with time-based join)

**✅ Do join to these mart tables** if not using OBT:
- `transaction_mart.all_transactions` - Core transaction data (join via transaction_id)
- `transaction_mart.transactions_gms` - GMS metrics
- `transaction_mart.transactions_buyer` - Buyer location data for cross-border analysis
- `transaction_mart.all_transactions_categories` - Category data

### Performance Tips

1. **Time-based location join** - Seller location is matched based on transaction creation_tsz falling between location log's creation_tsz and thru_creation_tsz
2. **Point-in-time accuracy** - Captures seller's location at the time of transaction, not current location
3. **PII fields** - All seller location fields contain PII
4. **Use transaction_obt for most queries** - This table is for specific use cases; transaction_obt has seller data already joined
5. **Cleaned zip codes** - seller_zip uses clean_zip_code UDF for standardization

### Common Query Patterns

**Get seller location for transactions:**
```sql
SELECT
    transaction_id,
    seller_user_id,
    seller_country_name,
    seller_state,
    seller_zip
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_seller`
WHERE transaction_id IN (1234, 5678)
```

**Cross-border transaction analysis:**
```sql
SELECT
    tb.buyer_country_name,
    ts.seller_country_name,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT t.receipt_id) AS receipt_count
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_seller` ts
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_buyer` tb
    ON ts.transaction_id = tb.transaction_id
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON ts.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
  AND tb.buyer_country_name != ts.seller_country_name
GROUP BY tb.buyer_country_name, ts.seller_country_name
ORDER BY transaction_count DESC
LIMIT 20
```

**Top seller countries:**
```sql
SELECT
    ts.seller_country_name,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT ts.seller_user_id) AS unique_sellers
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_seller` ts
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON ts.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
GROUP BY ts.seller_country_name
ORDER BY transaction_count DESC
LIMIT 20
```

**Domestic vs international transactions:**
```sql
SELECT
    CASE
        WHEN tb.buyer_country = ts.seller_country THEN 'Domestic'
        ELSE 'International'
    END AS transaction_type,
    COUNT(*) AS transaction_count,
    AVG(t.usd_subtotal_price) AS avg_price
FROM `etsy-data-warehouse-prod.transaction_mart.transactions_seller` ts
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.transactions_buyer` tb
    ON ts.transaction_id = tb.transaction_id
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON ts.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
GROUP BY transaction_type
```

---

## Important Notes

### Point-in-Time Location

Seller location is captured **at the time of the transaction**, not the seller's current location. This is critical for historical accuracy.

The join logic:
```sql
transaction.creation_tsz BETWEEN seller_location.creation_tsz AND seller_location.thru_creation_tsz
```

This ensures you get the seller's location that was active when the transaction occurred.

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
- `seller_state`: '  ' (two spaces)
- `seller_zip`: '' (empty string)

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transaction_obt` - **PREFER THIS** - Complete transaction data with seller location pre-joined
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transactions_gms` - GMS metrics
- `transaction_mart.transactions_buyer` - Buyer location data (for cross-border analysis)
- `transaction_mart.all_transactions_categories` - Category data
- `transaction_mart.transaction_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - Visit attribution

### Receipt Tables (Join via receipt_id)
- `transaction_mart.all_receipts` - Receipt-level details
- `transaction_mart.receipts_gms` - Receipt-level GMS
- `transaction_mart.receipt_obt` - Receipt-level OBT

### Accounting Tables
- `transaction_mart.accounting_gms` - Accounting GMS (may have different seller location)
- `transaction_mart.accounting_seller` - Accounting perspective seller data
- `transaction_mart.accounting_gms_by_trans` - Alternative accounting GMS

### Source Tables (Avoid - Data Already in Mart)
- `materialized.seller_location_log` - Historical seller locations (already incorporated with time-based join)
- `etsy_shard.shop_locations` - Raw shop location data (use seller_location_log instead)
