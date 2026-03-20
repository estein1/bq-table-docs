---
id: user_email_settings
title: etsy-data-warehouse-prod.user_mart.user_email_settings
sidebar_label: user_email_settings
---

# user_email_settings

User email subscription settings and preferences

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.user_email_settings`

**Source Script:** `Rollups/auto/p2/daily/user_mart_emails.sql`

**Primary Key**: `user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: cdata-alerts@etsy.com

## Purpose

Email engagement and preference tracking for users.

## Column Reference

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `user_id` | INT64 | Temp: `emails`, `user_mart.user_profile` | Primary Key | User ID. Primary Key |
| `email_address_id` | INT64 | `etsy_index.email_addresses` | Direct | Email address ID |
| `email` | STRING | `etsy_index.email_addresses` | `rtrim(lower_case_ascii_only(email))` | **PII**: Email address (lowercase, trimmed) |
| `email_domain` | STRING | Calculated | `SUBSTR(email, STRPOS(email, '@') + 1)` lowercased | Email domain (e.g., "gmail.com") |
| `is_guest` | INT64 | Temp: `emails` | 0 for registered users, 1 for guests | 1 if guest user |
| `is_confirmed` | INT64 | `etsy_index.email_addresses` | 1 if verification_state != 'unconfirmed' | 1 if email address has been verified |
| `is_reliable` | INT64 | `etsy_index.email_addresses` | 1 if reliability_state IN ('unknown', 'reliable') | 1 if email address is considered reliable |
| **Email Campaign Subscriptions** | | | | **1 = subscribed, 0 = not subscribed** |
| `is_etsy_finds` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'etsy_finds') where state = 1 | Subscribed to Etsy Finds email |
| `is_etsy_success` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'etsy_success') where state = 1 | Subscribed to Etsy Success email |
| `is_craft_buyer_prospects` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'craft_buyer_prospects') where state = 1 | Subscribed to Craft Buyer Prospects email |
| `is_new_at_etsy` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'new_at_etsy') where state = 1 | Subscribed to New at Etsy email |
| `is_coupon` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'auto_coupon') where state = 1 | Subscribed to Auto Coupon email |
| `is_etsy_advocacy` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'etsy_advocacy') where state = 1 | Subscribed to Etsy Advocacy email |
| `is_etsy_feedback` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'feedback') where state = 1 | Subscribed to Etsy Feedback email |
| `is_recommended_features` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'recommended_features') where state = 1 | Subscribed to News and Features email (sellers) |
| `is_pattern_marketing` | INT64 | `materialized.email_address_campaign`, `etsy_mail.email_campaign` | MAX(campaign_label = 'pattern_marketing') where state = 1 | Subscribed to Pattern News email (sellers) |

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.user_email_settings`
WHERE user_id = 123456789;
```

## Related Tables

- [user_profile](user_profile.md) - User demographics and preferences
- [mapped_user_profile](mapped_user_profile.md) - Cross-device user profiles
- All other user_mart tables
