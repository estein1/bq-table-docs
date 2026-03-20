---
id: mapped_user_profile
title: etsy-data-warehouse-prod.user_mart.mapped_user_profile
sidebar_label: mapped_user_profile
---

# mapped_user_profile

User profile aggregated by mapped_user_id for cross-device tracking. Similar to user_profile but enables analysis across multiple user_ids belonging to the same person.

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.mapped_user_profile`

**Source Script:** `Rollups/auto/p2/daily/post-visits/user_mart_profile.sql`

**Primary Key**: `mapped_user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com

**⚠️ PII**: Contains PII fields (`primary_email`, `country`)

## Purpose

Provides user profiles at the mapped_user_id level, enabling cross-device user analysis. When a user has multiple user_ids (e.g., different devices, guest vs registered), this table aggregates their profile using the mapped_user_id.

## Key Differences from user_profile

- Primary key is `mapped_user_id` instead of `user_id`
- Aggregates data across all user_ids that map to the same mapped_user_id
- Use this for cross-device analysis and unified user views
- Use user_profile for device/account-specific analysis

## Query Guidance

### When to Use This Table

- Cross-device user analysis
- Unified customer view across sessions
- LTV analysis at person level (not account level)
- User behavior across guest and registered sessions

### Common Query Pattern

```sql
SELECT
    mapped_user_id,
    country,
    region,
    ltv_decile,
    expected_ltv_52_weeks
FROM `etsy-data-warehouse-prod.user_mart.mapped_user_profile`
WHERE mapped_user_id = 123456789;
```

## Column Reference

**Data aggregated at the mapped_user_id level (guest + registered users with same email)**

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `mapped_user_id` | INT64 | `user_mart.user_profile` | Primary Key | **Mapped user ID**. Primary Key. Links guest and registered users with same email |
| `user_id` | INT64 | `user_mart.user_profile` | `MIN(user_id)` where is_guest = 0 | Registered user_id associated with mapped_user_id (if any) |
| `is_guest` | INT64 | Calculated | 1 if mapped_user_id = guest user_id with no registered users | 1 if mapped user ID is same as a guest user ID |
| `user_state` | STRING | `user_mart.user_profile` | Selected via QUALIFY ranking | User state (active/closed/dead). Selected from top-ranked user_id |
| `primary_email` | STRING | `user_mart.user_profile` | Selected via QUALIFY ranking | **PII**: User email (lowercase, trimmed) |
| `is_seller` | BOOL | `user_mart.user_profile` | Selected via QUALIFY ranking | Seller indicator (from top-ranked user_id) |
| `is_frozen` | BOOL | `user_mart.user_profile` | Selected via QUALIFY ranking | Account frozen indicator (from top-ranked user_id) |
| `is_admin` | BOOL | `user_mart.user_profile` | Selected via QUALIFY ranking | Admin account indicator (from top-ranked user_id) |
| `is_nipsa` | BOOL | `user_mart.user_profile` | Selected via QUALIFY ranking | NIPSA (Not In Public Search Areas) indicator |
| `is_redacted` | BOOL | `user_mart.user_profile` | Selected via QUALIFY ranking | Redacted account indicator |
| `gender` | STRING | `user_mart.user_profile` | Selected via QUALIFY ranking | User gender (mostly null) |
| `country` | STRING | `user_mart.user_profile` | Selected via QUALIFY ranking | **PII**: User country (2-letter ISO code) |
| `region` | STRING | `user_mart.user_profile` | Selected via QUALIFY ranking | User region |
| `language` | STRING | `user_mart.user_profile` | Selected via QUALIFY ranking | User language |
| `first_visit_top_channel` | STRING | `user_mart.user_profile` | Selected via QUALIFY ranking | Top channel of first registered visit |
| `first_conv_visit_top_channel` | STRING | `user_mart.user_profile` | Selected via QUALIFY ranking | Top channel of first converted visit |
| `join_date` | INT64 | Calculated | `MIN(join_date)` for registered users only | Join date of associated registered user (if any) |
| `first_guest_create_date` | INT64 | Calculated | `MIN(join_date)` for guest users | First creation date of associated guest users (if any) |
| `first_user_create_date` | INT64 | Calculated | `MIN(join_date)` across all users (guest + registered) | First creation date of all associated users |
| `account_age` | INT64 | Calculated | Days since first_user_create_date | Days since first user create date (registered or guest) |
| `ltv_decile` | INT64 | `user_mart.user_profile` | From mapped_user_id LTV calculation | LTV decile (1-10, 10 = highest) based on mapped_user_id |
| `expected_ltv_104_weeks` | FLOAT64 | `user_mart.user_profile` | From mapped_user_id LTV calculation | Expected GMS over next 104 weeks |
| `expected_ltv_52_weeks` | FLOAT64 | `user_mart.user_profile` | From mapped_user_id LTV calculation | Expected GMS over next 52 weeks |
| `gdpr_tp` | INT64 | `user_mart.user_profile` | Selected via QUALIFY ranking | GDPR targeted personalization preference |
| `gdpr_p` | INT64 | `user_mart.user_profile` | Selected via QUALIFY ranking | GDPR personalization preference |
| `att_opt_in` | INT64 | Temp table aggregation | `MIN(att_opt_in)` across all user_ids | Set to 0 if ANY user_id indicated ATT not authorized. 1 otherwise |
| `registered_user_count` | INT64 | Calculated | `SUM(is_guest = 0)` | Number of registered user_ids mapping to this mapped_user_id (0 or 1) |
| `guest_count` | INT64 | Calculated | `SUM(is_guest = 1)` | Number of guest user_ids mapping to this mapped_user_id |
| `is_active` | INT64 | Temp: `buyer_segments` | 1 if days_purchased_12m > 0 | Active buyer segment indicator |
| `is_habitual` | INT64 | Temp: `buyer_segments` | 1 if days_purchased_12m >= 6 AND gms_net_12m >= 200 | Habitual buyer segment indicator |
| `is_repeat` | INT64 | Temp: `buyer_segments` | 1 if days_purchased_12m >= 2 | Repeat buyer segment indicator |
| `is_high_potential` | INT64 | Temp: `buyer_segments` | 1 if first purchase in last 6mo AND (days_purchased_12m >= 2 OR gms_net_12m >= 100) | High potential buyer segment indicator |
| `buyer_segment` | STRING | Temp: `buyer_segments` | Hierarchical: Habitual > High Potential > Repeat > Active > Not Active | Buyer segment classification |

## Related Tables

- [user_profile](user_profile.md) - Profile by user_id (account level)
- `user_mart.user_mapping` - Mapping between user_id and mapped_user_id
- All other user_mart mapped_* tables
