# BigQuery Table Documentation

Comprehensive documentation for BigQuery tables at Etsy, with source lineage, business logic, and query guidance.

## Purpose

This repository provides detailed documentation for BigQuery tables to help:
- **Analysts** understand table structure and data sources
- **Data Engineers** trace data lineage and transformations
- **Claude/AI assistants** generate better, more accurate queries
- **New team members** onboard quickly with clear examples

## Documentation Structure

Each table's documentation includes:
- **Column Reference**: Detailed column definitions with source tables and business logic
- **Query Guidance**: When to use this table vs others, what joins to avoid, performance tips
- **Common Query Patterns**: Copy-paste ready SQL examples
- **Related Tables**: Complete list of related tables and how they connect

## Available Schemas

### listing_mart

**Schema**: `etsy-data-warehouse-prod.listing_mart`

Denormalized, analytics-ready tables for Etsy listing data. Includes pricing, variations, attributes, and images.

**📖 [View listing_mart Documentation](listing_mart/README.md)**

#### Tables

**Core Tables:**
- [listings](listing_mart/listings.md) - Main listing data with pricing and status
- [listing_titles](listing_mart/listing_titles.md) - Titles and descriptions
- [listing_attributes](listing_mart/listing_attributes.md) - Attributes, taxonomy, color, recipient
- [listing_images](listing_mart/listing_images.md) - Image URLs (up to 20 per listing)

**Variation & Product Tables:**
- [listing_variations](listing_mart/listing_variations.md) - Variation summary per listing
- [listing_variation_attributes](listing_mart/listing_variation_attributes.md) - Individual variation details
- [listing_products](listing_mart/listing_products.md) - Product-level inventory and SKUs

**Attribute Reference Tables:**
- [all_properties](listing_mart/all_properties.md) - Property ID → name mapping
- [all_property_values](listing_mart/all_property_values.md) - Value ID → value mapping
- [all_custom_property_names](listing_mart/all_custom_property_names.md) - Custom variation names

**Comprehensive Tables:**
- [listing_all_attributes](listing_mart/listing_all_attributes.md) - ALL listing attributes
- [listing_current_discounts](listing_mart/listing_current_discounts.md) - Active sales/discounts

**Indicators & Classification:**
- [listing_indicators](listing_mart/listing_indicators.md) - POD, how-it's-made, flagship segments, personalization

**Text Analysis & Search:**
- [listing_ngrams](listing_mart/listing_ngrams.md) - Tokens, bigrams, trigrams from titles/tags/styles
- [listing_ngrams_combined](listing_mart/listing_ngrams_combined.md) - Deduplicated ngrams
- [listing_all_queries](listing_mart/listing_all_queries.md) - Search queries with engagement metrics

**Performance & Metrics:**
- [listing_gms](listing_mart/listing_gms.md) - GMS, sales, orders (total, past year, normalized)
- [listing_post_purch](listing_mart/listing_post_purch.md) - Returns, refunds, ratings, cases
- [listing_counts](listing_mart/listing_counts.md) - Counts of attributes, tags, views, favorites

**Materialized Views:**
- [Active Listings Views](listing_mart/materialized_views/README.md) - 11 `_active` views for faster queries

### transaction_mart

**Schema**: `etsy-data-warehouse-prod.transaction_mart`

Denormalized, analytics-ready tables for all Etsy transactions and receipts. Includes GMS, buyer/seller demographics, visit attribution, and post-purchase metrics.

**📖 [View transaction_mart Documentation](transaction_mart/README.md)**

#### Tables

**⭐ One Big Tables (OBT) - Start Here:**
- [receipt_obt](transaction_mart/receipt_obt.md) - **Complete receipt data** (GMS, visits, post-purchase)
- [transaction_obt](transaction_mart/transaction_obt.md) - **Complete transaction data** (GMS, visits, reviews)

**Foundation Tables:**
- [all_receipts](transaction_mart/all_receipts.md) - Core receipt data
- [all_transactions](transaction_mart/all_transactions.md) - Core transaction data

**GMS (Revenue) Tables:**
- [receipts_gms](transaction_mart/receipts_gms.md) - Receipt-level GMS with aggregations
- [transactions_gms](transaction_mart/transactions_gms.md) - Transaction GMS (accounting focus)
- [transactions_gms_by_trans](transaction_mart/transactions_gms_by_trans.md) - Transaction GMS (operational focus)
- [accounting_gms](transaction_mart/accounting_gms.md) - Accounting GMS by date/source
- [accounting_gms_by_trans](transaction_mart/accounting_gms_by_trans.md) - Accounting GMS simplified

**Enrichment Tables:**
- [transactions_buyer](transaction_mart/transactions_buyer.md) - Buyer demographics & payment
- [transactions_seller](transaction_mart/transactions_seller.md) - Seller location at transaction time
- [accounting_seller](transaction_mart/accounting_seller.md) - Seller location at accounting time
- [all_transactions_categories](transaction_mart/all_transactions_categories.md) - Categories & taxonomy

**Visit Attribution Tables:**
- [transactions_visits](transaction_mart/transactions_visits.md) - Visit attribution per transaction
- [receipts_visits](transaction_mart/receipts_visits.md) - Visit attribution per receipt

**Post-Purchase Tables:**
- [transaction_post_purch](transaction_mart/transaction_post_purch.md) - Refunds & reviews
- [receipt_post_purch](transaction_mart/receipt_post_purch.md) - Returns, cases & help requests

#### Quick Example: Daily Revenue

```sql
-- Use receipt_obt for receipt-level analysis
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

**Note**: All transaction_mart money fields are in **USD dollars** (not cents like listing_mart).

### user_mart

**Schema**: `etsy-data-warehouse-prod.user_mart`

User behavior, demographics, and engagement metrics aggregated by user. Includes visit patterns, purchase history, LTV predictions, and email engagement.

**📖 [View user_mart Documentation](user_mart/README.md)**

#### Tables

**Core Profile Tables:**
- [user_profile](user_mart/user_profile.md) - User demographics, LTV, GDPR settings
- [mapped_user_profile](user_mart/mapped_user_profile.md) - Profile by mapped_user_id (cross-device)

**Daily Metrics:**
- [user_visit_daily](user_mart/user_visit_daily.md) - Daily visit metrics by platform/device
- [user_purch_daily](user_mart/user_purch_daily.md) - Daily purchase metrics
- [user_visit_daily_analytic](user_mart/user_visit_daily_analytic.md) - Analytics-optimized daily visits
- [user_purch_daily_analytic](user_mart/user_purch_daily_analytic.md) - Analytics-optimized daily purchases

**Lifetime (LTD) Metrics:**
- [user_visit_ltd](user_mart/user_visit_ltd.md) - Lifetime visit metrics
- [user_purch_ltd](user_mart/user_purch_ltd.md) - Lifetime purchase metrics
- [user_purch_detail_sum](user_mart/user_purch_detail_sum.md) - Detailed purchase summaries

**Time-Windowed Statistics (30d, 90d, 180d, 12m, LTD):**
- Visit stats: user_visit_stats, user_visit_stats_30d, user_visit_stats_90d, user_visit_stats_180d, user_visit_stats_12m
- Purchase stats: user_purch_stats, user_purch_stat_30d, user_purch_stat_90d, user_purch_stat_180d, user_purch_stat_12m

**Attribution & Email:**
- [user_first_visits](user_mart/user_first_visits.md) - First visit attribution
- [user_first_conv_visits](user_mart/user_first_conv_visits.md) - First converting visit
- [user_email_settings](user_mart/user_email_settings.md) - Email preferences
- [mapped_user_email_daily](user_mart/mapped_user_email_daily.md) - Daily email engagement

**Total**: 32 tables

## Quick Start

### Example: Get Active Listings with Prices

```sql
SELECT
    l.listing_id,
    lt.title,
    l.price_usd / 100.0 AS price_dollars,  -- Prices stored in cents!
    la.top_category,
    li.listing_image_url
FROM `etsy-data-warehouse-prod.listing_mart.listings` l
JOIN `etsy-data-warehouse-prod.listing_mart.listing_titles` lt USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_attributes` la USING (listing_id)
JOIN `etsy-data-warehouse-prod.listing_mart.listing_images` li USING (listing_id)
WHERE l.is_active = 1
  AND l.listing_id IN (12345, 67890)  -- Always filter on listing_id (clustered)
```

## Important Notes

### Money Fields

**listing_mart**: All price fields are stored in **CENTS** (INT64) - always divide by 100 for display:
```sql
price_usd / 100.0 AS price_dollars  -- ✅ Correct
price_usd AS price_dollars          -- ❌ Wrong - shows cents!
```

**transaction_mart**: All price/GMS fields are already in **DOLLARS** (NUMERIC) - no conversion needed:
```sql
trans_gms_net AS total_gms  -- ✅ Already in dollars
```

### Exchange Rates
**Market rates stored as rate*1e7** - divide by 10,000,000:
```sql
market_rate / 10000000.0 AS actual_rate  -- ✅ Correct
```

### Dates
**Unix timestamps in seconds** - convert with `TIMESTAMP_SECONDS()`:
```sql
TIMESTAMP_SECONDS(create_date) AS created_at
DATE(TIMESTAMP_SECONDS(create_date)) AS created_date
```

## Contributing

This documentation is maintained by the Data Marts team. To update:

1. Edit the markdown files directly
2. Follow the existing structure (Column Reference, Query Guidance, Related Tables)
3. Include real SQL examples
4. Submit a PR for review

## Support

- **Questions**: #bigquery Slack channel
- **Data Issues**: estein@etsy.com or data-marts@etsy.pagerduty.com
- **Owner Team**: data-marts@etsy.pagerduty.com

## License

Internal Etsy documentation.
