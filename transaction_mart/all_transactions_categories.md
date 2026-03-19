---
id: all_transactions_categories
title: transaction_mart.all_transactions_categories
sidebar_label: all_transactions_categories
---

# transaction_mart.all_transactions_categories

## Table Overview

**Purpose**: Category and taxonomy information for each transaction. One row per transaction with hierarchical category data and marketplace flags (vintage, handmade, supplies).

**Primary Key**: `transaction_id`

**Clustering**: Clustered by `transaction_id`

**Update Frequency**: Daily (p1 rollup)

**Owner**: afroehlich@etsy.com

**Owner Team**: finance-analytics@etsy.com

---

## Column Reference

### Identifiers

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `transaction_id` | INT64 | `etsy_shard.shop_transactions` | Direct from shop_transactions | Unique transaction identifier. Primary key |
| `listing_id` | INT64 | `etsy_shard.shop_transactions` | Direct from shop_transactions | Listing that was purchased |

### Category Hierarchy

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `new_category` | STRING | `materialized.transaction_taxonomy` (primary), `materialized.listing_categories_taxonomy` (fallback) | substr(coalesce(first element of split full_path, top_level_cat_new, 'other'), 1, 30) | Top-level category name. Prefers transaction taxonomy, falls back to listing taxonomy. Max 30 chars |
| `second_level_cat_new` | STRING | `materialized.transaction_taxonomy` (primary), `materialized.listing_categories_taxonomy` (fallback) | substr(second element of split full_path or second_level_cat_new, 1, 30) | Second-level category name. Max 30 chars |
| `third_level_cat_new` | STRING | `materialized.transaction_taxonomy` (primary), `materialized.listing_categories_taxonomy` (fallback) | substr(third element of split full_path or third_level_cat_new, 1, 30) | Third-level category name. Max 30 chars |
| `taxonomy_id` | INT64 | `materialized.transaction_taxonomy` (primary), `materialized.listing_categories_taxonomy` (fallback) | coalesce(transaction taxonomy_id, listing taxonomy_id) | Taxonomy ID. Prefers transaction-level, falls back to listing-level |

### Marketplace Flags

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `is_vintage` | INT64 | `materialized.transaction_marketplaces` (primary), `materialized.listing_marketplaces` (fallback) | coalesce(transaction is_vintage, listing is_vintage, 0) | Item is vintage (20+ years old) |
| `is_handmade` | INT64 | `materialized.transaction_marketplaces` (primary), `materialized.listing_marketplaces` (fallback) | coalesce(transaction is_handmade, listing is_handmade, 0) | Item is handmade |
| `is_supplies` | INT64 | `materialized.transaction_marketplaces` (primary), `materialized.listing_marketplaces` (fallback) | coalesce(transaction is_supplies, listing is_supplies, 0) | Item is craft supplies |

---

## Query Guidance

### When to Use This Table

**Use `transaction_mart.all_transactions_categories` for:**
- Transaction-level category analysis
- Marketplace type breakdown (vintage vs handmade vs supplies)
- Category hierarchy exploration
- **Enriching transaction queries** with taxonomy data

**Don't use this table for:**
- Complete transaction analysis → Use `transaction_mart.transaction_obt` (has category data pre-joined)
- Listing-level categories → Use `listing_mart.listing_attributes`
- Receipt-level category aggregations → Use `transaction_mart.receipts_gms` (has top_category_* fields)

### Avoid These Joins

**❌ Don't join to these tables** (data available in better tables):
- For complete transaction analysis, use `transaction_obt` instead (category data already joined)
- `materialized.transaction_taxonomy` - Already incorporated here
- `materialized.listing_categories_taxonomy` - Already incorporated as fallback

**✅ Do join to these mart tables** if not using OBT:
- `transaction_mart.all_transactions` - Core transaction data (join via transaction_id)
- `transaction_mart.transactions_gms` - GMS metrics
- `transaction_mart.transactions_buyer` - Buyer demographics
- `transaction_mart.transactions_seller` - Seller data

### Performance Tips

1. **Clustered by `transaction_id`** - Always filter on transaction_id when possible
2. **Use transaction_obt for most queries** - This table is for specific use cases; transaction_obt has category data already joined
3. **Category hierarchy** - new_category is top level, second_level_cat_new is second level, third_level_cat_new is third level
4. **Transaction vs listing taxonomy** - Prefers transaction-level taxonomy (more accurate) but falls back to listing-level if unavailable
5. **Max 30 characters** - All category names truncated to 30 characters

### Common Query Patterns

**Get category breakdown:**
```sql
SELECT
    tc.new_category,
    COUNT(*) AS transaction_count,
    SUM(t.quantity) AS items_sold
FROM `etsy-data-warehouse-prod.transaction_mart.all_transactions_categories` tc
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON tc.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
GROUP BY tc.new_category
ORDER BY transaction_count DESC
```

**Marketplace type analysis:**
```sql
SELECT
    CASE
        WHEN tc.is_vintage = 1 THEN 'Vintage'
        WHEN tc.is_handmade = 1 THEN 'Handmade'
        WHEN tc.is_supplies = 1 THEN 'Supplies'
        ELSE 'Other'
    END AS marketplace_type,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT t.buyer_user_id) AS unique_buyers
FROM `etsy-data-warehouse-prod.transaction_mart.all_transactions_categories` tc
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON tc.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
GROUP BY marketplace_type
ORDER BY transaction_count DESC
```

**Category hierarchy:**
```sql
SELECT
    tc.new_category,
    tc.second_level_cat_new,
    tc.third_level_cat_new,
    COUNT(*) AS transaction_count
FROM `etsy-data-warehouse-prod.transaction_mart.all_transactions_categories` tc
INNER JOIN `etsy-data-warehouse-prod.transaction_mart.all_transactions` t
    ON tc.transaction_id = t.transaction_id
WHERE DATE(t.creation_tsz) = '2026-03-19'
  AND t.live = 1
  AND tc.new_category = 'jewelry'
GROUP BY 1, 2, 3
ORDER BY transaction_count DESC
LIMIT 20
```

---

## Important Notes

### Category Source Logic

Categories are sourced with the following priority:
1. **Transaction taxonomy** (`materialized.transaction_taxonomy`) - Preferred source, most accurate
2. **Listing taxonomy** (`materialized.listing_categories_taxonomy`) - Fallback if transaction taxonomy unavailable
3. **'other'** - Default if both sources unavailable

### Full Path Format

The `full_path` from transaction_taxonomy is dot-separated (e.g., "jewelry.necklaces.pendant_necklaces"):
- Split by '.' to get hierarchy levels
- First element → `new_category`
- Second element → `second_level_cat_new`
- Third element → `third_level_cat_new`

### Marketplace Flags

Items can have multiple marketplace flags (e.g., handmade supplies):
- `is_vintage = 1`: Item is 20+ years old
- `is_handmade = 1`: Item was made by hand
- `is_supplies = 1`: Item is craft/art supplies

Not mutually exclusive - a vintage handmade item would have both flags set.

### Klarna Exclusion

Transactions from Klarna-cancelled receipts are excluded from this table via the WHERE clause in the CREATE statement.

---

## Related Tables

### Transaction Mart Tables (Join on transaction_id)
- `transaction_mart.transaction_obt` - **PREFER THIS** - Complete transaction data with categories pre-joined
- `transaction_mart.all_transactions` - Core transaction data
- `transaction_mart.transactions_gms` - GMS metrics
- `transaction_mart.transactions_buyer` - Buyer demographics
- `transaction_mart.transactions_seller` - Seller data
- `transaction_mart.transaction_post_purch` - Post-purchase metrics
- `transaction_mart.transactions_visits` - Visit attribution

### Receipt Tables (Join via receipt_id)
- `transaction_mart.receipts_gms` - Has top_category_items and top_category_gms aggregated fields
- `transaction_mart.all_receipts` - Receipt-level details
- `transaction_mart.receipt_obt` - Receipt-level OBT

### Listing Tables (Join via listing_id)
- `listing_mart.listings` - Listing details
- `listing_mart.listing_attributes` - Listing-level category and attributes

### Other Transaction Tables
- `transaction_mart.accounting_gms` - Accounting GMS
- `transaction_mart.accounting_gms_by_trans` - Alternative accounting GMS

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.shop_transactions` - Raw transactions (use all_transactions instead)
- `materialized.transaction_taxonomy` - Transaction taxonomy (already incorporated)
- `materialized.listing_categories_taxonomy` - Listing taxonomy (already used as fallback)
- `materialized.transaction_marketplaces` - Transaction marketplace flags (already incorporated)
- `materialized.listing_marketplaces` - Listing marketplace flags (already used as fallback)
