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
