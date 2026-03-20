---
id: listing_post_purch
title: etsy-data-warehouse-prod.listing_mart.listing_post_purch
sidebar_label: listing_post_purch
---

# listing_post_purch

Post-purchase metrics for listings including returns, refunds, ratings, and cases. Only contains listings with at least one post-purchase event.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_post_purch`

**Source Script:** `Rollups/auto/p2/daily/listing_mart_bucket3.sql`

**Primary Key:** `listing_id`

**Clustering:** None

**Update Frequency:** Daily (full refresh)

**Owner:** estein@etsy.com

**Owner Team:** cdata-alerts@etsy.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `transaction_mart.transaction_post_purch` | Primary Key | Listing id. Primary Key. Only listings with post-purchase data |
| `is_active` | INT64 | `listing_mart.listings` | Direct | 1 if listing is active, 0 if inactive |
| `returned_receipts` | INT64 | `transaction_mart.receipt_post_purch` | Sum of receipt_return | Number of receipts with returns |
| `help_receipts` | INT64 | `transaction_mart.receipt_post_purch` | Sum of help_with_order | Number of receipts with Help with Order requests |
| `non_delivery_receipts` | INT64 | `transaction_mart.receipt_post_purch` | Sum of non_delivery_cases (capped at 1 per receipt) | Number of receipts with non-delivery cases |
| `not_as_described_receipts` | INT64 | `transaction_mart.receipt_post_purch` | Sum of not_as_described_cases (capped at 1 per receipt) | Number of receipts with not-as-described cases |
| `arrived_damaged_receipts` | INT64 | `transaction_mart.receipt_post_purch` | Sum of arrived_damaged_cases (capped at 1 per receipt) | Number of receipts with arrived-damaged cases |
| `transaction_refunds` | INT64 | `transaction_mart.transaction_post_purch` | Sum where listing_refund_amount > 0 | Number of transactions with any refund amount |
| `transaction_refund_amount` | NUMERIC | `transaction_mart.transaction_post_purch` | Sum of listing_refund_amount, rounded | **Total refund amount (USD dollars)** for this listing |
| `rating_1` | INT64 | `transaction_mart.transaction_post_purch` | Count where rating = 1 | Number of 1-star ratings |
| `rating_2` | INT64 | `transaction_mart.transaction_post_purch` | Count where rating = 2 | Number of 2-star ratings |
| `rating_3` | INT64 | `transaction_mart.transaction_post_purch` | Count where rating = 3 | Number of 3-star ratings |
| `rating_4` | INT64 | `transaction_mart.transaction_post_purch` | Count where rating = 4 | Number of 4-star ratings |
| `rating_5` | INT64 | `transaction_mart.transaction_post_purch` | Count where rating = 5 | Number of 5-star ratings |

## Query Guidance

### When to Use This Table

- Analyzing listing quality and customer satisfaction
- Identifying problematic listings (high returns, low ratings, cases)
- Calculating refund rates and amounts
- Understanding post-purchase issues
- Quality control and seller performance analysis

### Important Limitation

**Sparse table:** Only includes listings with at least one post-purchase event (return, refund, rating, or case). Listings with no post-purchase data are excluded.

### Avoid These Joins

- ❌ Don't join to `transaction_mart.transaction_post_purch` - data already aggregated
- ❌ Don't join to `transaction_mart.receipt_post_purch` - data already aggregated

### Performance Tips

- **Filter on `listing_id`** when searching for specific listings
- NOT NULL on specific columns to find listings with that issue (e.g., `WHERE returned_receipts > 0`)
- Join to `listing_gms` to calculate rates (returns / total_orders)

### Important Notes

**Money in USD Dollars:**
- `transaction_refund_amount` is in **USD dollars** (rounded to 2 decimals)

**Receipt-level vs Transaction-level:**
- Returns and cases are receipt-level (one receipt can have multiple transactions)
- Refunds and ratings are transaction-level

**Case counting:**
- Cases are capped at 1 per receipt using `LEAST(case_count, 1)` to avoid double-counting

### Common Query Patterns

**Listings with high return rates:**
```sql
SELECT
    p.listing_id,
    p.returned_receipts,
    g.total_orders,
    SAFE_DIVIDE(p.returned_receipts, g.total_orders) AS return_rate
FROM `etsy-data-warehouse-prod.listing_mart.listing_post_purch` p
JOIN `etsy-data-warehouse-prod.listing_mart.listing_gms` g USING (listing_id)
WHERE g.total_orders >= 20  -- Minimum order threshold
ORDER BY return_rate DESC
LIMIT 100;
```

**Average rating per listing:**
```sql
SELECT
    listing_id,
    (rating_1 * 1 + rating_2 * 2 + rating_3 * 3 + rating_4 * 4 + rating_5 * 5) /
      NULLIF(rating_1 + rating_2 + rating_3 + rating_4 + rating_5, 0) AS avg_rating,
    rating_1 + rating_2 + rating_3 + rating_4 + rating_5 AS total_ratings,
    rating_5,
    rating_1
FROM `etsy-data-warehouse-prod.listing_mart.listing_post_purch`
WHERE rating_1 + rating_2 + rating_3 + rating_4 + rating_5 >= 10  -- Min 10 ratings
ORDER BY avg_rating DESC
LIMIT 100;
```

**Listings with high refund amounts:**
```sql
SELECT
    p.listing_id,
    p.transaction_refunds,
    p.transaction_refund_amount,
    g.total_gms,
    SAFE_DIVIDE(p.transaction_refund_amount, g.total_gms) AS refund_rate
FROM `etsy-data-warehouse-prod.listing_mart.listing_post_purch` p
JOIN `etsy-data-warehouse-prod.listing_mart.listing_gms` g USING (listing_id)
WHERE p.transaction_refund_amount > 0
ORDER BY p.transaction_refund_amount DESC
LIMIT 100;
```

**Case analysis by type:**
```sql
SELECT
    listing_id,
    non_delivery_receipts,
    not_as_described_receipts,
    arrived_damaged_receipts,
    non_delivery_receipts + not_as_described_receipts + arrived_damaged_receipts AS total_case_receipts
FROM `etsy-data-warehouse-prod.listing_mart.listing_post_purch`
WHERE non_delivery_receipts + not_as_described_receipts + arrived_damaged_receipts > 0
ORDER BY total_case_receipts DESC
LIMIT 100;
```

**Listings with many help requests:**
```sql
SELECT
    p.listing_id,
    p.help_receipts,
    g.total_orders,
    SAFE_DIVIDE(p.help_receipts, g.total_orders) AS help_request_rate
FROM `etsy-data-warehouse-prod.listing_mart.listing_post_purch` p
JOIN `etsy-data-warehouse-prod.listing_mart.listing_gms` g USING (listing_id)
WHERE p.help_receipts > 0
ORDER BY help_request_rate DESC
LIMIT 100;
```

**Rating distribution:**
```sql
SELECT
    SUM(rating_1) AS total_1_star,
    SUM(rating_2) AS total_2_star,
    SUM(rating_3) AS total_3_star,
    SUM(rating_4) AS total_4_star,
    SUM(rating_5) AS total_5_star,
    SUM(rating_1 + rating_2 + rating_3 + rating_4 + rating_5) AS total_ratings
FROM `etsy-data-warehouse-prod.listing_mart.listing_post_purch`;
```

## Related Tables

### listing_mart Tables (Join on listing_id)

- [listing_gms](listing_gms.md) - Revenue and sales metrics (for calculating rates)
- [listings](listings.md) - Main listing data
- [listing_attributes](listing_attributes.md) - Listing attributes
- All other listing_mart tables

### transaction_mart Tables

- `transaction_mart.transaction_post_purch` - Transaction-level post-purchase data
- `transaction_mart.receipt_post_purch` - Receipt-level post-purchase data
- `transaction_mart.transaction_obt` - Complete transaction data with post-purchase

### Source Tables (Avoid - Data Already in Mart)

- `transaction_mart.transaction_post_purch` - Transaction data already aggregated
- `transaction_mart.receipt_post_purch` - Receipt data already aggregated
