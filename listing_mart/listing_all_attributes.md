---
id: listing_all_attributes
title: listing_mart.listing_all_attributes
sidebar_label: listing_all_attributes
---

# listing_mart.listing_all_attributes

## Table Overview

**Purpose**: Comprehensive attribute table containing ALL listing attributes from multiple sources. Each row represents one attribute value for a listing. A single listing can have many rows (one per attribute).

**Primary Key**: Composite key (`listing_id`, `attribute_name`, `attribute_value`)

**Clustering**: Clustered by `attribute_name`

**Update Frequency**: Daily (p2 rollup)

**Owner**: estein@etsy.com

**Owner Team**: data-marts@etsy.pagerduty.com

---

## Column Reference

### Identifiers & Status

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `listing_mart.listings` | Direct from listings | Listing identifier. Part of composite primary key |
| `is_active` | INT64 | `listing_mart.listings` | Direct from listings | Active listing indicator |

### Attribute Details

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `attribute_id` | INT64 | `listing_mart.all_properties` | From all_properties.ott_attribute_id joined on property_id | OTT attribute identifier. NULL if not mapped to OTT |
| `attribute_name` | STRING | `etsy_index.properties` OR `listing_mart.all_properties` | Lowercase property name with 'ListingMetadata_Property_' prefix removed. Examples: 'whomade', 'listingtype', 'color', 'recipient', 'source' | Attribute name (lowercase). Property name with prefix removed |
| `attribute_value` | STRING | Source varies: `listing_attributes` OR `listings_properties_values` | For listing_attributes: from all_property_values. For listings_properties_values: value transformed via CASE statements for whomade (1→'i_did'), listingtype (0→'physical'), howmade (0→'natural'), whatcontent (0→'original'), source (ios→'soe'), whattools (0→'hand_tools' array). Otherwise from property_options or raw value. HTML entities decoded. Truncated to 1000 chars | Human-readable value. Transformed for certain attributes (whomade, listingtype, howmade, whatcontent, source, whattools). HTML entities decoded |
| `attribute_type` | STRING | Derived from source | 'ott sd' if from structured_data.values, 'ott' if from property_option_inputs, 'lpv sd' if from listings_properties_values with property_options or special transforms, 'lpv' otherwise. Truncated to 10 chars | Source type indicator. Shows whether value comes from structured data or raw properties |
| `scale_name` | STRING | `listing_mart.all_property_values` | From all_property_values for listing_attributes source only. NULL for listings_properties_values source | Measurement scale if applicable. E.g., "Inches", "Centimeters". Only populated for listing_attributes source |

### Dates

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `create_date` | INT64 | `etsy_shard.listing_attributes` OR `etsy_shard.listings_properties_values` | From create_date/created field of source table | Attribute creation timestamp. **Unix timestamp (seconds)** |
| `update_date` | INT64 | `etsy_shard.listing_attributes` OR `etsy_shard.listings_properties_values` | From update_date/modified field of source table. For listings_properties_values, most recent by modified date is selected | Attribute last update timestamp. **Unix timestamp (seconds)** |

---

## Data Quality

### Data Sources

This table aggregates attributes from two primary sources:

1. **listing_attributes** (etsy_shard.listing_attributes):
   - Structured variation attributes
   - Uses all_properties and all_property_values for mapping
   - Includes scale_name

2. **listings_properties_values** (etsy_shard.listings_properties_values):
   - Metadata properties
   - Most recent value per listing/property (by modified date)
   - Excludes certain system properties (category, taxonomy, shorturl, etc.)
   - Handles array values (e.g., whattools can have multiple values)

### Excluded Properties

These properties are excluded from listings_properties_values source:
- `listingmetadata_property_category`
- `listingmetadata_property_taxonomy`
- `listingmetadata_property_cat2mappedtaxonomy`
- `listingmetadata_property_shorturl`
- `listingmetadata_property_originalcategoryid`
- `listingmetadata_property_squarecatalogitemid`
- `listingmetadata_property_style` (handled separately)

---

## Business Logic

### Attribute Name Transformation

- Prefix `'ListingMetadata_Property_'` is removed
- Converted to lowercase
- Example: `'ListingMetadata_Property_WhoMade'` → `'whomade'`

### Value Transformations

Certain attributes have coded values that are transformed:

**whomade**:
- `'1'` → `'i_did'`
- `'2'` → `'collective'`
- `'3'` → `'someone_else'`
- Other → `'unknown'`

**listingtype**:
- `'0'` → `'physical'`
- `'1'` → `'download'`
- Other → `'unknown'`

**howmade**:
- `'0'` → `'natural'`
- `'1'` → `'from_scratch'`
- `'2'` → `'assembled'`
- `'3'` → `'altered'`
- `'4'` → `'curated'`
- Other → `'unknown'`

**whatcontent**:
- `'0'` → `'original'`
- `'1'` → `'compiled'`
- `'2'` → `'ai_gen'`
- Other → `'unknown'`

**whattools** (can have multiple values per listing):
- `'0'` → `'hand_tools'`
- `'1'` → `'computerized_tools'`
- `'2'` → `'ai'`
- `'3'` → `'no_tools'`
- Other → `'unknown: [value]'`

**source** (listing creation source):
- Values containing 'sellonetsy', 'iphone', or 'ipad' → `'soe'`
- Other → Original value

**listing_source_detail** (detailed source tracking):
- Values containing 'buttersellonetsy' → `'soe-esa'`
- Values containing 'sellonetsy', 'iphone', or 'ipad' → `'soe-legacy'`
- Other → Original value

### Array Values

- `whattools` can have comma-separated multiple values
- Each value creates a separate row
- Other attributes have single values

### Deduplication

- For listings_properties_values source, most recent value by `modified` date is selected
- Only one row per (listing_id, property_id) combination from this source

---

## Common Use Cases

### Analytics Queries

**Get all attributes for a listing:**
```sql
SELECT
    attribute_name,
    attribute_value,
    attribute_type
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE listing_id = 12345
ORDER BY attribute_name
```

**Handmade vs AI-generated content:**
```sql
SELECT
    attribute_value AS how_made,
    COUNT(DISTINCT listing_id) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE attribute_name = 'howmade'
AND is_active = 1
GROUP BY attribute_value
ORDER BY listing_count DESC
```

**Digital downloads:**
```sql
SELECT
    listing_id
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE attribute_name = 'listingtype'
AND attribute_value = 'download'
AND is_active = 1
```

**Listings created via Sell on Etsy:**
```sql
SELECT
    COUNT(DISTINCT listing_id) AS soe_listings
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE attribute_name = 'source'
AND attribute_value = 'soe'
AND is_active = 1
```

**Pivot attributes for analysis:**
```sql
SELECT
    listing_id,
    MAX(CASE WHEN attribute_name = 'whomade' THEN attribute_value END) AS who_made,
    MAX(CASE WHEN attribute_name = 'howmade' THEN attribute_value END) AS how_made,
    MAX(CASE WHEN attribute_name = 'listingtype' THEN attribute_value END) AS listing_type
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE is_active = 1
GROUP BY listing_id
```

**Listings using AI tools:**
```sql
SELECT DISTINCT
    listing_id
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE attribute_name = 'whattools'
AND attribute_value IN ('ai', 'computerized_tools')
AND is_active = 1
```

---

## Display Formatting Rules

### Attribute Names
- Already lowercase
- Display in title case for UI: "Who Made", "How Made"
- Replace underscores with spaces: `who_made` → "Who Made"

### Attribute Values
- Already human-readable (transformed from codes)
- Display as-is
- Special handling for multi-value attributes (whattools)

### Dates
- `create_date`, `update_date` → **Unix timestamps in seconds**
- Convert using `TIMESTAMP_SECONDS()` for display

### Attribute Type
- Primarily for internal use
- Can display as badge showing source: "Structured Data", "Property Value"

---

## Important Notes

### Multiple Rows Per Listing

- **This is NOT a one-row-per-listing table**
- Each listing can have 10-50+ rows (one per attribute)
- Use GROUP BY or MAX/MIN aggregations to pivot to one row per listing

### Array Attributes

- `whattools` creates multiple rows with same attribute_name
- Example: A listing using both hand_tools and AI will have 2 rows

### Sparse Data

- Not all listings have all attributes
- Use LEFT JOIN when combining with other tables
- Check for NULL attribute_value

---

## Query Guidance

### When to Use This Table

**Use `listing_mart.listing_all_attributes` for:**
- Comprehensive attribute search (ALL attributes, not just curated ones)
- Finding listings by whomade, howmade, whatcontent, source, style
- Searching attributes not in `listing_mart.listing_attributes`
- **Multiple rows per listing** (one per attribute)

**Don't use this table for:**
- Just recipient, color, taxonomy → Use `listing_mart.listing_attributes` (faster, curated)
- Variation-specific attributes → Use `listing_mart.listing_variation_attributes`
- Summary queries → This table has many rows per listing

### Avoid These Joins

**❌ Don't join to these tables** (data already here):
- `etsy_shard.listing_attributes` - Already included
- `etsy_shard.listings_properties_values` - Already included
- `listing_mart.all_properties` - Already joined
- `listing_mart.all_property_values` - Already joined

**✅ Do join to these mart tables**:
- `listing_mart.listings` - For price, availability, shop info
- `listing_mart.listing_titles` - For listing title
- `listing_mart.listing_attributes` - For curated attributes if needed together

### Performance Tips

1. **Clustered by `attribute_name`** - Filter on attribute_name for best performance, NOT listing_id
2. **Many rows per listing** - Use DISTINCT listing_id or aggregate carefully
3. **Attribute names are lowercase** - Use `attribute_name = 'color'` not 'Color'
4. **Special value transformations** - whomade, listingtype, howmade, whatcontent, source, whattools are decoded to human-readable values
5. **whattools is an array** - Single listing can have multiple whattools values (multiple rows)

### Common Query Patterns

**Find listings by attribute:**
```sql
SELECT DISTINCT listing_id
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE is_active = 1
  AND attribute_name = 'whomade'
  AND attribute_value = 'i_did'
```

**Get all attributes for a listing:**
```sql
SELECT
  attribute_name,
  attribute_value,
  attribute_type
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE listing_id = 12345
ORDER BY attribute_name, attribute_value
```

**Find handmade items:**
```sql
SELECT DISTINCT listing_id
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE is_active = 1
  AND attribute_name = 'howmade'
  AND attribute_value IN ('from_scratch', 'assembled')
```

**Attribute distribution:**
```sql
SELECT
  attribute_name,
  attribute_value,
  COUNT(DISTINCT listing_id) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE is_active = 1
  AND attribute_name IN ('whomade', 'listingtype', 'howmade')
GROUP BY attribute_name, attribute_value
ORDER BY attribute_name, listing_count DESC
```

**Combine multiple attributes:**
```sql
SELECT
  listing_id,
  MAX(IF(attribute_name = 'whomade', attribute_value, NULL)) AS whomade,
  MAX(IF(attribute_name = 'howmade', attribute_value, NULL)) AS howmade,
  MAX(IF(attribute_name = 'source', attribute_value, NULL)) AS source
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_attributes`
WHERE listing_id IN (12345, 67890)
GROUP BY listing_id
```

---

## Related Tables

### Listing Mart Tables (Join on listing_id)
- `listing_mart.listings` - Core listing data (price, availability, shop info, status flags)
- `listing_mart.listing_titles` - Titles and descriptions
- `listing_mart.listing_attributes` - Curated subset of key attributes (recipient, color, taxonomy, digital, personalizable)
- `listing_mart.listing_images` - Image URLs
- `listing_mart.listing_variations` - Variation summary
- `listing_mart.listing_variation_attributes` - Individual variation details
- `listing_mart.listing_products` - Product-level inventory and SKUs
- `listing_mart.listing_current_discounts` - Active promotions

### Reference Tables (Used to Build This Table)
- `listing_mart.all_properties` - Property ID to name lookup (attribute_name sourced from here)
- `listing_mart.all_property_values` - Value ID to value lookup (attribute_value sourced from here)
- `listing_mart.all_custom_property_names` - Custom variation names

### Source Tables (Avoid - Data Already in Mart)
- `etsy_shard.listing_attributes` - Variation attributes (already included here)
- `etsy_shard.listings_properties_values` - Metadata properties (already included with value transformations)
- `etsy_index.property_options` - Property option names (already joined for value lookup)
