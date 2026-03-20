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

## Related Tables

- [user_profile](user_profile.md) - Profile by user_id (account level)
- `user_mart.user_mapping` - Mapping between user_id and mapped_user_id
- All other user_mart mapped_* tables
