---
id: user_profile
title: etsy-data-warehouse-prod.user_mart.user_profile
sidebar_label: user_profile
---

# user_profile

Comprehensive user profile with demographics, account information, LTV predictions, and privacy settings. Includes both registered users and guest users.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_profile`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_profile.sql`

**Primary Key**: `user_id`

**Clustering**: None specified

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

**⚠️ PII**: Contains PII fields (`primary_email`, `country`)

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | `etsy_index.users_index`, `etsy_index.guest_users_index` | Primary Key | User ID (registered or guest). Primary Key |
| `is_guest` | INT64 | Calculated | 0 for registered, 1 for guests | 1 if guest user, 0 if registered user |
| `user_state` | STRING | `etsy_index.users_index`, `etsy_index.guest_users_index` | Direct | User account state (active/closed/dead) |
| `primary_email` | STRING | `etsy_index.users_index`, `etsy_index.guest_users_index` | `lower_case_ascii_only()`, trimmed | **PII**: User email (lowercase, trimmed) |
| `is_seller` | BOOL | `etsy_index.users_index` | Direct (NULL for guests) | True if user is a seller (NULL for guests) |
| `is_frozen` | BOOL | `etsy_index.users_index` | Direct (NULL for guests) | Account frozen status (NULL for guests) |
| `is_admin` | BOOL | `etsy_index.users_index` | Direct (NULL for guests) | Admin status (NULL for guests) |
| `is_nipsa` | BOOL | `etsy_index.users_index` | Direct (NULL for guests) | NIPSA (Not In Public Search Areas) status (NULL for guests) |
| `is_redacted` | BOOL | `etsy_index.users_index` | Direct (NULL for guests) | Redacted status (NULL for guests) |
| `gender` | STRING | `etsy_shard.user_details` | Direct (NULL for guests) | User gender (mostly null) |
| `country` | STRING | `rollups.user_country`, `etsy_v2.countries`, `visit_mart.visits_transactions` (guests) | ISO country code lookup | **PII**: ISO country code (derived from purchases for guests) |
| `region` | STRING | `etsy_v2.world_regions` | Via country_region mapping | Geographic region (from world_regions) |
| `language` | STRING | Temp table: `etsy_shard.user_preferences`, `weblog.visits` | Preference ID 210 or detected from visits | User language (preferences > detected language) |
| `first_visit_top_channel` | STRING | `user_mart.user_first_visits` | NULL if joined before first visit | Top channel for first visit |
| `first_conv_visit_top_channel` | STRING | `user_mart.user_first_conv_visits` | Direct | Top channel for first converting visit |
| `join_date` | INT64 | `etsy_shard.user_details`, `etsy_index.guest_users_index` | Rounded to day: `join_date - mod(join_date, 86400)` | Unix timestamp of account creation (rounded to day) |
| `account_age` | INT64 | Calculated | `Floor((UNIX_SECONDS(current_date) - join_date) / 86400)` | Days since account creation |
| `ltv_decile` | INT64 | Temp table: `buyatt_mart.analytics_clv` | `ntile(10)` over expectedgms104 DESC | LTV decile (1-10, 10 = highest LTV) by mapped_user_id |
| `expected_ltv_104_weeks` | FLOAT64 | Temp table: `buyatt_mart.analytics_clv`, `buyatt_mart.ltv_prediction_data` | Adjusted by purchase_days and recency | Expected GMS over next 104 weeks (adjusted by purchase behavior) |
| `expected_ltv_52_weeks` | FLOAT64 | Temp table: `buyatt_mart.analytics_clv`, `buyatt_mart.ltv_prediction_data` | Adjusted by purchase_days and recency | Expected GMS over next 52 weeks (adjusted by purchase behavior) |
| `mapped_user_id` | INT64 | `user_mart.user_mapping` | Coalesce with user_id if NULL | Mapped user ID for cross-device tracking |
| `gdpr_tp` | INT64 | Temp table: `etsy_shard.user_preferences` | Preference ID 791, most recent | GDPR targeted personalization preference (NULL for guests) |
| `gdpr_p` | INT64 | Temp table: `etsy_shard.user_preferences` | Preference ID 792, most recent | GDPR personalization preference (NULL for guests) |
| `gdpr_buyer_email` | INT64 | Temp table: `etsy_shard.user_preferences` | Preference ID 793, most recent | GDPR buyer email preference (NULL for guests) |
| `att_opt_in` | INT64 | Temp table: `etsy_shard.user_preferences` | Preference ID 811, coalesce to 1 | App Tracking Transparency opt-in (defaults to 1, NULL for guests) |

## Query Guidance

### When to Use This Table

- User demographic analysis
- LTV segmentation and analysis
- Privacy/consent analysis (GDPR, ATT)
- User account status checks
- Cross-device tracking via mapped_user_id
- First visit/conversion attribution

### When to Use mapped_user_profile Instead

- Use [mapped_user_profile](mapped_user_profile.md) when you need to aggregate across devices using mapped_user_id

### Avoid These Joins

- ❌ Don't join to `etsy_index.users_index` - user data already denormalized here
- ❌ Don't join to `etsy_shard.user_details` - demographic data already here
- ❌ Don't join for LTV data - LTV predictions already included

### Performance Tips

- **Filter on `user_id`** for single-user lookups
- **Filter on `is_guest`** to separate registered vs guest users
- **LTV deciles** provide quick segmentation (1-10 scale)
- For cross-device analysis, use `mapped_user_id`

### Important Notes

**PII Handling:**
- Contains PII (`primary_email`, `country`)
- Email is lowercase and trimmed for consistency
- Follow Etsy PII handling guidelines when querying

**Guest Users:**
- Guest users have `is_guest = 1`
- Many fields NULL for guests (is_seller, is_frozen, GDPR settings, etc.)
- Country derived from purchase history for guests
- Guest users included via separate INSERT statement

**LTV Predictions:**
- From `buyatt_mart.analytics_clv`
- Adjusted based on purchase recency and frequency
- LTV decile: 1 = lowest, 10 = highest expected value

**Mapped Users:**
- `mapped_user_id` enables cross-device tracking
- Maps multiple user_ids to single identity
- See `user_mart.user_mapping` for full mapping table

### Common Query Patterns

**User profile lookup:**
```sql
SELECT
    user_id,
    is_guest,
    country,
    region,
    language,
    account_age,
    ltv_decile,
    is_seller
FROM `etsy-data-warehouse-prod.user_mart.user_profile`
WHERE user_id = 123456789;
```

**LTV segment distribution:**
```sql
SELECT
    ltv_decile,
    COUNT(*) AS user_count,
    AVG(expected_ltv_52_weeks) AS avg_ltv_52w,
    AVG(account_age) AS avg_account_age_days
FROM `etsy-data-warehouse-prod.user_mart.user_profile`
WHERE is_guest = 0
  AND ltv_decile IS NOT NULL
GROUP BY ltv_decile
ORDER BY ltv_decile DESC;
```

**Users by country and region:**
```sql
SELECT
    region,
    country,
    COUNT(*) AS user_count,
    SUM(CASE WHEN is_seller THEN 1 ELSE 0 END) AS seller_count
FROM `etsy-data-warehouse-prod.user_mart.user_profile`
WHERE is_guest = 0
  AND country IS NOT NULL
GROUP BY region, country
ORDER BY user_count DESC
LIMIT 50;
```

**GDPR consent analysis:**
```sql
SELECT
    gdpr_tp,
    gdpr_p,
    gdpr_buyer_email,
    att_opt_in,
    COUNT(*) AS user_count
FROM `etsy-data-warehouse-prod.user_mart.user_profile`
WHERE is_guest = 0
GROUP BY gdpr_tp, gdpr_p, gdpr_buyer_email, att_opt_in
ORDER BY user_count DESC;
```

**First visit attribution:**
```sql
SELECT
    first_visit_top_channel,
    COUNT(*) AS user_count,
    AVG(account_age) AS avg_account_age,
    AVG(expected_ltv_52_weeks) AS avg_ltv
FROM `etsy-data-warehouse-prod.user_mart.user_profile`
WHERE first_visit_top_channel IS NOT NULL
  AND is_guest = 0
GROUP BY first_visit_top_channel
ORDER BY user_count DESC;
```

## Related Tables

### user_mart Tables (Join on user_id)

- [mapped_user_profile](mapped_user_profile.md) - Profile aggregated by mapped_user_id
- [user_visit_daily](user_visit_daily.md) - Daily visit metrics
- [user_purch_daily](user_purch_daily.md) - Daily purchase metrics
- [user_first_visits](user_first_visits.md) - First visit details
- [user_first_conv_visits](user_first_conv_visits.md) - First converting visit details
- All other user_mart tables

### Reference Tables

- `user_mart.user_mapping` - User ID to mapped_user_id mapping
- `buyatt_mart.analytics_clv` - LTV prediction source
- `etsy_v2.countries` - Country reference data
- `etsy_v2.world_regions` - Region reference data

### Source Tables (Avoid - Data Already in Mart)

- `etsy_index.users_index` - User core data already here
- `etsy_shard.user_details` - Demographics already here
- `etsy_shard.user_preferences` - Preferences already here
