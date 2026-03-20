---
id: listing_ngrams
title: etsy-data-warehouse-prod.listing_mart.listing_ngrams
sidebar_label: listing_ngrams
---

# listing_ngrams

Tokens, bigrams, and trigrams extracted from listing titles, tags, and style attributes with stop words removed.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.listing_mart.listing_ngrams`

**Source Script:** `Rollups/auto/p2/daily/listing_mart_2.sql`

**Primary Key:** None (listing_id + ngram + source + type combination)

**Clustering:** None

**Update Frequency:** Daily (full refresh)

**Owner:** estein@etsy.com

**Owner Team:** cdata-alerts@etsy.com

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `listing_id` | INT64 | `listing_mart.listings` | Active listings only | Listing id. Active listings only |
| `phrase` | STRING | Source field | Original text | Original listing title, concatenated tags, or style attribute value |
| `ngram` | STRING | Derived | Tokenized text | Token, bigram, or trigram extracted from phrase |
| `type` | STRING | Derived | token/bigram/trigram | Type of ngram: "token" (single word), "bigram" (2 words), or "trigram" (3 words) |
| `source` | STRING | Source field | title/tag/lpv | Source of the ngram: "title", "tag", or "lpv" (listing property value for style) |

## Query Guidance

### When to Use This Table

- Text analysis and natural language processing on listing content
- Finding listings by specific words or phrases
- Building search/recommendation features
- Analyzing common terminology across titles, tags, and styles
- Token-level analysis (each word appears separately)

### Use listing_ngrams_combined Instead If

- You want deduplicated ngrams per listing (one row per listing + ngram)
- You need aggregated sources/types for each ngram

### Avoid These Joins

- ❌ Don't join to `listing_mart.listing_titles` - title data already processed here
- ❌ Don't join to `materialized.listings_tags_concat` - tag data already processed here
- ❌ Don't join to `listing_mart.listing_all_attributes` for style - already processed here

### Performance Tips

- **Filter on `listing_id`** when searching for specific listings
- **Filter on `ngram`** when searching for specific words/phrases
- **Filter on `source`** to limit to titles, tags, or styles only
- **Filter on `type = 'token'`** for single-word analysis
- This table can be large - use LIMIT or aggregate queries
- Consider using `listing_ngrams_combined` for deduplication

### Text Processing Notes

**Stop words removed:** Common words (a, the, and, etc.) from `static.stop_words`

**Special characters removed:** Single quotes, commas, semicolons, ampersands, pipes, hyphens, backslashes, slashes (when surrounded by spaces)

**Lowercase:** All text converted to lowercase

**Ngram generation:**
- **Tokens:** Individual words after processing
- **Bigrams:** 2 consecutive words
- **Trigrams:** 3 consecutive words

### Common Query Patterns

**Find listings with specific word in title:**
```sql
SELECT DISTINCT
    listing_id,
    phrase AS title
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams`
WHERE ngram = 'handmade'
  AND source = 'title'
  AND type = 'token'
LIMIT 100;
```

**Count most common bigrams in titles:**
```sql
SELECT
    ngram,
    COUNT(DISTINCT listing_id) AS listing_count
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams`
WHERE source = 'title'
  AND type = 'bigram'
GROUP BY ngram
ORDER BY listing_count DESC
LIMIT 50;
```

**Analyze tokens across all sources for a listing:**
```sql
SELECT
    listing_id,
    source,
    type,
    ngram
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams`
WHERE listing_id = 123456789
  AND type = 'token'
ORDER BY source, ngram;
```

**Find listings with multi-word phrase (using bigram):**
```sql
SELECT DISTINCT
    listing_id,
    phrase
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams`
WHERE ngram = 'vintage jewelry'
  AND type = 'bigram'
  AND source IN ('title', 'tag')
LIMIT 100;
```

**Compare title vs tag vocabulary:**
```sql
SELECT
    source,
    COUNT(DISTINCT ngram) AS unique_tokens
FROM `etsy-data-warehouse-prod.listing_mart.listing_ngrams`
WHERE type = 'token'
GROUP BY source;
```

## Related Tables

### listing_mart Tables (Join on listing_id)

- [listing_ngrams_combined](listing_ngrams_combined.md) - Deduplicated ngrams per listing
- [listing_all_queries](listing_all_queries.md) - Search queries that led to listings
- [listing_titles](listing_titles.md) - Original listing titles
- [listing_all_attributes](listing_all_attributes.md) - Full attribute data including styles
- [listings](listings.md) - Main listing data

### Reference Tables

- `static.stop_words` - Stop words excluded from ngrams
- `rollups.twograms` - UDF for bigram generation
- `rollups.threegrams` - UDF for trigram generation

### Source Tables (Avoid - Data Already in Mart)

- `listing_mart.listing_titles` - Titles already processed into ngrams
- `materialized.listings_tags_concat` - Tags already processed into ngrams
- `listing_mart.listing_all_attributes` - Style attributes already processed
