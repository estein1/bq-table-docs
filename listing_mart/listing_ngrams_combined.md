---
id: listing_ngrams_combined
title: etsy-data-warehouse-prod.listing_mart.listing_ngrams_combined
sidebar_label: listing_ngrams_combined
---

# listing_ngrams_combined

Deduplicated ngrams per listing with aggregated sources and types. One row per listing + ngram combination.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_ngrams_combined`

**Source Script:** `Rollups/auto/p2/daily/listing_mart_2.sql`

**Primary Key:** `listing_id` + `ngram`

**Clustering:** None

**Update Frequency:** Daily (full refresh)

**Owner:** estein@etsy.com

**Owner Team:** cdata-alerts@etsy.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `listing_mart.listing_ngrams` | Grouped | Listing id. Active listings only. Part of Primary Key |
| `ngram` | STRING | `listing_mart.listing_ngrams` | Grouped | Token, bigram, or trigram. Part of Primary Key |
| `sources` | STRING | `listing_mart.listing_ngrams` | Concatenated sources | Comma-separated list of sources where ngram appears (title, tag, lpv) |
| `types` | STRING | `listing_mart.listing_ngrams` | Concatenated types | Comma-separated list of types (token, bigram, trigram) |

## Query Guidance

### When to Use This Table

- Finding unique ngrams per listing (deduplicated)
- Checking if a listing has a specific word/phrase across any source
- Simpler queries when you don't need individual phrase breakdowns
- Analyzing ngram coverage across sources

### Use listing_ngrams Instead If

- You need to see the original phrase (title, tag, style value)
- You need separate rows for each occurrence of an ngram
- You're analyzing specific sources separately (title vs tag)

### Avoid These Joins

- ❌ Don't join to `listing_mart.listing_titles` - data already processed
- ❌ Don't join to `materialized.listings_tags_concat` - data already processed

### Performance Tips

- **Filter on `listing_id`** when searching for specific listings
- **Filter on `ngram`** when searching for specific words/phrases
- **Use LIKE on `sources`** to check if ngram appears in specific source (e.g., `sources LIKE '%title%'`)
- This table is smaller than `listing_ngrams` due to deduplication

### Common Query Patterns

**Check if listing contains word anywhere:**
```sql
SELECT
    listing_id,
    ngram,
    sources,
    types
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams_combined`
WHERE listing_id = 123456789
  AND ngram = 'handmade';
```

**Find listings with word in title specifically:**
```sql
SELECT DISTINCT
    listing_id
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams_combined`
WHERE ngram = 'vintage'
  AND sources LIKE '%title%'
LIMIT 100;
```

**Count listings per ngram:**
```sql
SELECT
    ngram,
    COUNT(DISTINCT listing_id) AS listing_count,
    APPROX_COUNT_DISTINCT(CASE WHEN sources LIKE '%title%' THEN listing_id END) AS in_title_count,
    APPROX_COUNT_DISTINCT(CASE WHEN sources LIKE '%tag%' THEN listing_id END) AS in_tag_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams_combined`
GROUP BY ngram
HAVING listing_count > 100
ORDER BY listing_count DESC
LIMIT 50;
```

**Get all ngrams for a listing:**
```sql
SELECT
    ngram,
    sources,
    types
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams_combined`
WHERE listing_id = 123456789
ORDER BY ngram;
```

**Find listings with multiple specific words:**
```sql
WITH target_listings AS (
  SELECT listing_id
  FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams_combined`
  WHERE ngram IN ('handmade', 'jewelry', 'silver')
  GROUP BY listing_id
  HAVING COUNT(DISTINCT ngram) = 3  -- Has all 3 words
)
SELECT
    l.listing_id,
    STRING_AGG(nc.ngram) AS all_ngrams
FROM target_listings l
JOIN `etsy-data-warehouse-prod.listing_mart.listing_ngrams_combined` nc USING (listing_id)
GROUP BY l.listing_id
LIMIT 100;
```

## Related Tables

### listing_mart Tables (Join on listing_id)

- [listing_ngrams](listing_ngrams.md) - Full ngram data with phrases and duplicates
- [listing_all_queries](listing_all_queries.md) - Search queries that led to listings
- [listing_titles](listing_titles.md) - Original listing titles
- [listings](listings.md) - Main listing data

### Reference Tables

- `static.stop_words` - Stop words excluded from ngrams

### Source Tables (Avoid - Data Already in Mart)

- `listing_mart.listing_ngrams` - Source for this aggregated table
