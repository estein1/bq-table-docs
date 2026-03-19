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
**All price fields are stored in CENTS** - always divide by 100 for display:
```sql
price_usd / 100.0 AS price_dollars  -- ✅ Correct
price_usd AS price_dollars          -- ❌ Wrong - shows cents!
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
