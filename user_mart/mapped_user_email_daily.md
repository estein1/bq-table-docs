---
id: mapped_user_email_daily
title: etsy-data-warehouse-prod.user_mart.mapped_user_email_daily
sidebar_label: mapped_user_email_daily
---

# mapped_user_email_daily

Daily email engagement metrics by mapped_user_id

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.mapped_user_email_daily`

**Source Script:** `Rollups/auto/p2/daily/user_mart_emails.sql`

**Primary Key**: `mapped_user_id, _date`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: cdata-alerts@etsy.com

## Purpose

Email engagement and preference tracking for users.

## Column Reference

**Data aggregated at the mapped_user_id level (guest + registered users with same email)**

| Column | Type | Source Table | Business Logic | Description |
|--------|------|--------------|----------------|-------------|
| `mapped_user_id` | INT64 | Temp: `emails` | Primary Key | **Mapped user ID**. Primary Key. Links guest and registered users with same email |
| `email_address_id` | INT64 | `email.opens`, `email.clicks`, `braze_marketing_mart.opens`, `braze_marketing_mart.clicks`, `braze_transactional_mart.opens`, `braze_transactional_mart.clicks` | Direct | Email address ID |
| `run_date` | INT | Calculated | `UNIX_SECONDS(TIMESTAMP_TRUNC(_date, DAY))` | Date in unix format |
| `_date` | DATE | `email.opens`, `email.clicks`, etc | Date of email engagement | Date (partition key) |
| **Email Opens** | | | | |
| `opens` | INT64 | `email.opens`, `braze_marketing_mart.opens`, `braze_transactional_mart.opens` | `SUM()` where utm_source IN ('adhoc', 'newsletter', 'lifecycle') | Total email opens (non-transactional) |
| `open_days` | INT64 | Calculated | 1 if opens > 0, else 0 | Number of days with opens (1 or 0 for daily table) |
| `adhoc_opens` | INT64 | `email.opens`, `braze_marketing_mart.opens` | `SUM()` where utm_source = 'adhoc' | Adhoc campaign email opens |
| `newsletter_opens` | INT64 | `email.opens`, `braze_marketing_mart.opens` | `SUM()` where utm_source = 'newsletter' | Newsletter email opens |
| `lifecycle_opens` | INT64 | `email.opens`, `braze_marketing_mart.opens` | `SUM()` where utm_source = 'lifecycle' | Lifecycle email opens |
| `transact_opens` | INT64 | `email.opens`, `braze_marketing_mart.opens`, `braze_transactional_mart.opens` | `SUM()` where utm_source = 'transactional' | Transactional email opens |
| **Email Clicks** | | | | |
| `clicks` | INT64 | `email.clicks`, `braze_marketing_mart.clicks`, `braze_transactional_mart.clicks` | `SUM()` where utm_source IN ('adhoc', 'newsletter', 'lifecycle') | Total email clicks (non-transactional) |
| `click_days` | INT64 | Calculated | 1 if clicks > 0, else 0 | Number of days with clicks (1 or 0 for daily table) |
| `adhoc_clicks` | INT64 | `email.clicks`, `braze_marketing_mart.clicks` | `SUM()` where utm_source = 'adhoc' | Adhoc campaign email clicks |
| `newsletter_clicks` | INT64 | `email.clicks`, `braze_marketing_mart.clicks` | `SUM()` where utm_source = 'newsletter' | Newsletter email clicks |
| `lifecycle_clicks` | INT64 | `email.clicks`, `braze_marketing_mart.clicks` | `SUM()` where utm_source = 'lifecycle' | Lifecycle email clicks |
| `transact_clicks` | INT64 | `email.clicks`, `braze_marketing_mart.clicks`, `braze_transactional_mart.clicks` | `SUM()` where utm_source = 'transactional' | Transactional email clicks |

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.mapped_user_email_daily`
WHERE mapped_user_id = 123456789;
```

## Related Tables

- [user_profile](user_profile.md) - User demographics and preferences
- [mapped_user_profile](mapped_user_profile.md) - Cross-device user profiles
- All other user_mart tables
