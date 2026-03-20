---
id: listing_all_queries
title: etsy-data-warehouse-prod.listing_mart.listing_all_queries
sidebar_label: listing_all_queries
---

# listing_all_queries

Search queries associated with listings, including click, cart add, and purchase counts. One row per listing + query combination.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_all_queries`

**Source Script:** `Rollups/auto/p2/daily/listing_mart_2.sql`

**Primary Key:** `listing_id` + `query`

**Clustering:** None

**Update Frequency:** Daily (full refresh)

**Owner:** estein@etsy.com

**Owner Team:** cdata-alerts@etsy.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `search.visit_level_listing_impressions` | Active listings with engagement | Listing id. Active listings only. Part of Primary Key |
| `query` | STRING | `search.visit_level_listing_impressions` | Non-empty queries | Search query. Part of Primary Key |
| `clicks` | INT64 | `search.visit_level_listing_impressions` | Sum of clicks | Total clicks for this listing + query combination |
| `carts` | INT64 | `search.visit_level_listing_impressions` | Sum of cart adds | Total cart adds for this listing + query combination |
| `purchases` | INT64 | `search.visit_level_listing_impressions` | Sum of purchases | Total purchases for this listing + query combination |

## Query Guidance

### When to Use This Table

- Understanding which search queries drive traffic to listings
- Analyzing search performance and conversion
- Finding top-performing queries for a listing
- Identifying listing-query combinations with high engagement
- SEO and search optimization analysis

### Avoid These Joins

- ❌ Don't join to `search.visit_level_listing_impressions` - data already aggregated here

### Performance Tips

- **Filter on `listing_id`** when analyzing specific listings
- **Filter on `query`** when analyzing specific search terms
- **Data volume depends on partition expiration** of source table `search.visit_level_listing_impressions`
- Only includes queries where `clicks + carts + purchases > 0` (no zero-engagement rows)
- Stop words are NOT removed from queries (unlike listing_ngrams)

### Important Notes

- **Historical data limited** by source table partition expiration
- **Stop words included** in queries (unlike ngrams tables)
- **Only engaged queries** - filters for clicks OR carts OR purchases > 0
- **Active listings only**

### Common Query Patterns

**Top queries for a listing:**
```sql
SELECT
    query,
    clicks,
    carts,
    purchases,
    SAFE_DIVIDE(purchases, clicks) AS conversion_rate
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_queries`
WHERE listing_id = 123456789
ORDER BY clicks DESC
LIMIT 20;
```

**Find listings performing well for a query:**
```sql
SELECT
    listing_id,
    clicks,
    carts,
    purchases,
    SAFE_DIVIDE(purchases, clicks) AS cvr
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_queries`
WHERE query = 'handmade jewelry'
ORDER BY purchases DESC
LIMIT 50;
```

**Listings with high cart-add rate but low purchase rate:**
```sql
SELECT
    listing_id,
    query,
    clicks,
    carts,
    purchases,
    SAFE_DIVIDE(carts, clicks) AS cart_rate,
    SAFE_DIVIDE(purchases, carts) AS cart_to_purchase_rate
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_queries`
WHERE clicks >= 10
  AND SAFE_DIVIDE(carts, clicks) > 0.2
  AND SAFE_DIVIDE(purchases, carts) < 0.1
ORDER BY carts DESC
LIMIT 100;
```

**Query performance summary for a listing:**
```sql
SELECT
    listing_id,
    COUNT(DISTINCT query) AS unique_queries,
    SUM(clicks) AS total_clicks,
    SUM(carts) AS total_carts,
    SUM(purchases) AS total_purchases,
    SAFE_DIVIDE(SUM(purchases), SUM(clicks)) AS overall_cvr
FROM `etsy-data-warehouse-prod.listing_mart.listing_all_queries`
WHERE listing_id = 123456789
GROUP BY listing_id;
```

**Most common words in converting queries:**
```sql
WITH query_words AS (
  SELECT
    LOWER(word) AS word,
    SUM(purchases) AS total_purchases
  FROM `etsy-data-warehouse-prod.listing_mart.listing_all_queries`,
  UNNEST(SPLIT(query, ' ')) AS word
  WHERE purchases > 0
  GROUP BY word
)
SELECT
    word,
    total_purchases
FROM query_words
WHERE LENGTH(word) > 2  -- Filter very short words
ORDER BY total_purchases DESC
LIMIT 50;
```

## Related Tables

### listing_mart Tables (Join on listing_id)

- [listing_ngrams](listing_ngrams.md) - Listing content tokens (compare to query terms)
- [listing_ngrams_combined](listing_ngrams_combined.md) - Deduplicated listing ngrams
- [listing_titles](listing_titles.md) - Listing titles
- [listings](listings.md) - Main listing data

### Reference Tables

- None

### Source Tables (Avoid - Data Already in Mart)

- `search.visit_level_listing_impressions` - Source data (partition expiration limits history)
