---
id: user_mart_overview
title: user_mart Schema Overview
sidebar_label: Overview
---

# user_mart Schema Overview

## Purpose

The `user_mart` schema contains denormalized, analytics-ready tables for user behavior, demographics, and engagement metrics. These tables aggregate visit, purchase, and email activity by user for efficient querying and analysis.

**Schema**: `etsy-data-warehouse-prod.user_mart`

**Owner**: estein@etsy.com

**Owner Team**: visits-support@etsy.pagerduty.com / cdata-alerts@etsy.com

**Update Frequency**: Daily (p2 rollup)

**Source Scripts**:
- `Rollups/auto/p2/daily/user_mart_1.sql` - Visit aggregations and first visits
- `Rollups/auto/p2/daily/user_mart_emails.sql` - Email engagement
- `Rollups/auto/p2/daily/post-visits/user_mart_incr.sql` - Daily visit metrics (incremental)
- `Rollups/auto/p2/daily/post-visits/user_mart_profile.sql` - User profiles
- `Rollups/auto/p2/daily/post-visits/user_mart_purch.sql` - Purchase metrics
- `Rollups/auto/p2/daily/post-visits/user_mart_purch_stats.sql` - Purchase statistics
- `Rollups/auto/p2/daily/post-visits/user_mart_visit_stats.sql` - Visit statistics

---

## Table Index

### Core Profile Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [user_profile](user_profile.md) | User demographics, LTV, GDPR settings | `user_id` | User demographic analysis, LTV segmentation |
| [mapped_user_profile](mapped_user_profile.md) | Profile by mapped_user_id for cross-device tracking | `mapped_user_id` | Cross-device user analysis |

### Daily Metrics Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [user_visit_daily](user_visit_daily.md) | Daily visit metrics by platform/device/app | `user_id`, `_date` | Daily visit tracking, platform breakdown |
| [user_visit_daily_analytic](user_visit_daily_analytic.md) | Analytics-optimized daily visits | `user_id`, `_date` | Analytics workflows |
| [user_purch_daily](user_purch_daily.md) | Daily purchase metrics | `user_id`, `_date` | Daily purchase tracking |
| [user_purch_daily_analytic](user_purch_daily_analytic.md) | Analytics-optimized daily purchases | `user_id`, `_date` | Analytics workflows |
| [mapped_user_purch_daily](mapped_user_purch_daily.md) | Daily purchases by mapped_user_id | `mapped_user_id`, `_date` | Cross-device purchase tracking |
| [mapped_user_purch_daily_analytic](mapped_user_purch_daily_analytic.md) | Analytics-optimized mapped purchases | `mapped_user_id`, `_date` | Analytics workflows |

### Lifetime (LTD) Metrics Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [user_visit_ltd](user_visit_ltd.md) | Lifetime visit metrics | `user_id` | Total user visit history |
| [user_visit_ltd_analytic](user_visit_ltd_analytic.md) | Analytics-optimized lifetime visits | `user_id` | Analytics workflows |
| [user_purch_ltd](user_purch_ltd.md) | Lifetime purchase metrics | `user_id` | Total user purchase history |
| [user_purch_ltd_analytic](user_purch_ltd_analytic.md) | Analytics-optimized lifetime purchases | `user_id` | Analytics workflows |
| [user_purch_detail_sum](user_purch_detail_sum.md) | Detailed lifetime purchase summaries | `user_id` | Product-level purchase analysis |
| [mapped_user_purch_ltd](mapped_user_purch_ltd.md) | Lifetime purchases by mapped_user_id | `mapped_user_id` | Cross-device lifetime purchases |
| [mapped_user_purch_ltd_analytic](mapped_user_purch_ltd_analytic.md) | Analytics-optimized mapped LTD purchases | `mapped_user_id` | Analytics workflows |

### Visit Statistics Tables (Time-Windowed)

| Table | Time Window | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [user_visit_stats](user_visit_stats.md) | Lifetime (since 2010) | `user_id` | LTD visit statistics |
| [user_visit_stats_30d](user_visit_stats_30d.md) | Past 30 days | `user_id` | Recent visit behavior |
| [user_visit_stats_90d](user_visit_stats_90d.md) | Past 90 days | `user_id` | Quarterly visit trends |
| [user_visit_stats_180d](user_visit_stats_180d.md) | Past 180 days | `user_id` | Semi-annual visit trends |
| [user_visit_stats_12m](user_visit_stats_12m.md) | Past 12 months | `user_id` | Annual visit trends |
| [mapped_user_visit_stats](mapped_user_visit_stats.md) | Lifetime (since 2023) | `mapped_user_id` | Cross-device visit stats |

### Purchase Statistics Tables (Time-Windowed)

| Table | Time Window | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [user_purch_stats](user_purch_stats.md) | Lifetime (since 2005) | `user_id` | LTD purchase statistics |
| [user_purch_stat_30d](user_purch_stat_30d.md) | Past 30 days | `user_id` | Recent purchase behavior |
| [user_purch_stat_90d](user_purch_stat_90d.md) | Past 90 days | `user_id` | Quarterly purchase trends |
| [user_purch_stat_180d](user_purch_stat_180d.md) | Past 180 days | `user_id` | Semi-annual purchase trends |
| [user_purch_stat_12m](user_purch_stat_12m.md) | Past 12 months | `user_id` | Annual purchase trends |
| [mapped_user_purch_stats](mapped_user_purch_stats.md) | Lifetime (since 2023) | `mapped_user_id` | Cross-device purchase stats |

### Attribution Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [user_first_visits](user_first_visits.md) | First visit attribution | `user_id` | User acquisition analysis |
| [user_first_conv_visits](user_first_conv_visits.md) | First converting visit attribution | `user_id` | Conversion attribution |

### Email Engagement Tables

| Table | Description | Primary Key | Use Case |
|-------|-------------|-------------|----------|
| [user_email_settings](user_email_settings.md) | Email subscription settings | `user_id` | Email preferences |
| [mapped_user_email_daily](mapped_user_email_daily.md) | Daily email engagement | `mapped_user_id`, `_date` | Email performance tracking |
| [mapped_user_email_ltd](mapped_user_email_ltd.md) | Lifetime email engagement | `mapped_user_id` | Email engagement history |

---

## Common Query Patterns

### User Profile Lookup

```sql
SELECT
    p.user_id,
    p.country,
    p.ltv_decile,
    p.account_age,
    v.total_visits,
    s.total_orders
FROM `etsy-data-warehouse-prod.user_mart.user_profile` p
LEFT JOIN `etsy-data-warehouse-prod.user_mart.user_visit_stats` v USING (user_id)
LEFT JOIN `etsy-data-warehouse-prod.user_mart.user_purch_stats` s USING (user_id)
WHERE p.user_id = 123456789;
```

### Recent User Activity

```sql
SELECT
    v._date,
    v.visits,
    v.conv_visits,
    p.total_orders,
    p.total_gms_gross
FROM `etsy-data-warehouse-prod.user_mart.user_visit_daily` v
LEFT JOIN `etsy-data-warehouse-prod.user_mart.user_purch_daily` p
  ON v.user_id = p.user_id AND v._date = p._date
WHERE v.user_id = 123456789
  AND v._date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
ORDER BY v._date DESC;
```

### LTV Segment Analysis

```sql
SELECT
    ltv_decile,
    COUNT(*) AS users,
    AVG(expected_ltv_52_weeks) AS avg_ltv,
    AVG(account_age) AS avg_age_days
FROM `etsy-data-warehouse-prod.user_mart.user_profile`
WHERE is_guest = 0
  AND ltv_decile IS NOT NULL
GROUP BY ltv_decile
ORDER BY ltv_decile DESC;
```

---

## Important Notes

### User ID vs Mapped User ID

**user_id**: Individual account or guest session identifier
- Use for account-specific analysis
- Tables: `user_profile`, `user_visit_daily`, `user_purch_daily`, etc.

**mapped_user_id**: Cross-device user identifier
- Maps multiple user_ids to single identity
- Use for unified customer view
- Tables: `mapped_user_profile`, `mapped_user_purch_daily`, `mapped_user_email_daily`, etc.

### Daily vs LTD vs Stats Tables

**Daily tables** (`*_daily`):
- One row per user per date
- Partitioned by `_date`
- Use for time-series analysis

**Lifetime tables** (`*_ltd`):
- One row per user
- All-time aggregations
- Use for overall user metrics

**Stats tables** (`*_stats`, `*_stat_30d`, etc.):
- Pre-aggregated statistics
- Multiple time windows available
- Faster than aggregating from daily tables

### Analytic vs Regular Tables

**Analytic tables** (`*_analytic`):
- Optimized for analytics workflows
- Additional calculated fields
- Use when working in analytics tools

### PII Considerations

- `user_profile` and `mapped_user_profile` contain PII (`primary_email`, `country`)
- Follow Etsy PII handling guidelines
- Email addresses are lowercase and trimmed

---

## Best Practices

### Performance

1. **Use stats tables** for pre-aggregated metrics instead of aggregating daily tables
2. **Filter on `_date`** (partition key) when querying daily tables
3. **Choose appropriate time window** - use 30d/90d tables for recent analysis instead of LTD
4. **Use mapped_user tables** for cross-device analysis

### Data Quality

1. **Check `is_guest`** flag to separate registered vs guest users
2. **LTD tables include all-time data** - be aware of historical scope
3. **Stats tables have different start dates** (visit stats since 2010, purchase stats since 2005)
4. **Incremental table** (`user_visit_daily`) has 1-day lookback

---

## Support

- **Questions**: #bigquery Slack channel
- **Data Issues**: estein@etsy.com or visits-support@etsy.pagerduty.com
- **Owner Team**: visits-support@etsy.pagerduty.com, cdata-alerts@etsy.com

---

## Schema Statistics

- **Total Tables**: 32
- **Daily Metrics**: 8 tables
- **Lifetime Metrics**: 9 tables
- **Time-Windowed Stats**: 12 tables (6 visit + 6 purchase)
- **Profile/Attribution**: 3 tables
