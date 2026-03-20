---
id: mapped_user_email_ltd
title: etsy-data-warehouse-prod.user_mart.mapped_user_email_ltd
sidebar_label: mapped_user_email_ltd
---

# mapped_user_email_ltd

Lifetime email engagement metrics by mapped_user_id

## Table Overview

**Full Table Name:** `etsy-data-warehouse-prod.user_mart.mapped_user_email_ltd`

**Source Script:** `Rollups/auto/p2/daily/user_mart_emails.sql`

**Primary Key**: `mapped_user_id`

**Update Frequency**: Daily (full refresh)

**Owner**: estein@etsy.com

**Owner Team**: cdata-alerts@etsy.com

## Purpose

Email engagement and preference tracking for users.

## Query Guidance

### Common Query Pattern

```sql
SELECT *
FROM `etsy-data-warehouse-prod.user_mart.mapped_user_email_ltd`
WHERE mapped_user_id = 123456789;
```

## Related Tables

- [user_profile](user_profile.md) - User demographics and preferences
- [mapped_user_profile](mapped_user_profile.md) - Cross-device user profiles
- All other user_mart tables
