# `analytics_prod.staging.stg_marketing__campaigns`

## Ownership & access

- **Owning team:** Marketing Analytics (Slack: `#marketing-analytics`)
- **Source system:** Internal Campaign Management Tool (CMT) + ad platform exports (Google Ads, Meta, TikTok)
- **Access level:** Read for everyone in the default `data-readers` group. No additional grant needed.
- **SLA:** Available by **07:00 UTC daily**, max delay 2 hours. Listed as "Marketing Performance" in `docs/dashboards.md`.
- **Bug-report path:** Open a ticket in `YOUR-JIRA-PROJECT` with `Component = Data Quality`. **Do not** DM individual marketers.
- **Last reviewed:** 2025-01-15

## Overview

- **Full path:** `analytics_prod.staging.stg_marketing__campaigns`
- **Grain:** **One row per campaign per day per platform.** A single campaign running on Google Ads + Meta produces two rows per day.
- **Type:** Staging (light cleaning of platform exports; no business logic).

## Partitioning & filters

- **Partition column:** `report_date` (`DATE`)
- **Required filter:** `WHERE report_date >= ...`. Full table is ~80 GB; partition filter is required.
- **Recommended date range:** Last 30 days for performance reviews; trailing 13 months for YoY.

## Freshness

- **Update frequency:** Daily at 06:30 UTC.
- **Typical lag:** Same-day `report_date` is finalized at 06:30 UTC the *following* day. Same-day numbers can be retroactively corrected by ad platforms for up to **3 days** — re-querying yesterday's data tomorrow is normal and not a bug.
- **How to check:** `SELECT MAX(report_date), MAX(loaded_at) FROM analytics_prod.staging.stg_marketing__campaigns;`
- **Producing job (theirs):** `marketing__campaigns_load` (orchestrated in Marketing Analytics' Airflow instance, not ours).

## Columns

| Column | Type | Nullable | Class | Description |
|---|---|---|---|---|
| `report_date` | `DATE` | N | Internal | **Partition column.** Day the activity was reported. |
| `campaign_id` | `STRING` | N | Internal | Campaign primary key from the CMT. **Stable across platforms.** |
| `platform` | `STRING` | N | Internal | `'google_ads' | 'meta' | 'tiktok' | 'email' | 'affiliate'`. |
| `campaign_name` | `STRING` | N | Internal | Human-readable. Subject to renaming by marketers — don't join on it. |
| `objective` | `STRING` | N | Internal | `'awareness' | 'consideration' | 'conversion' | 'retention'`. |
| `region` | `STRING` | N | Internal | `'us' | 'ca' | 'uk' | 'au'`. Set per-campaign. |
| `daily_budget` | `DECIMAL(12,2)` | Y | **Confidential** | Daily spend cap. Internal-only. |
| `spend` | `DECIMAL(12,2)` | N | **Confidential** | Reported spend, in `currency`. **Don't share externally.** |
| `currency` | `STRING` | N | Internal | ISO 4217. |
| `impressions` | `BIGINT` | N | Internal | Reported impressions. |
| `clicks` | `BIGINT` | N | Internal | Reported clicks. |
| `conversions` | `BIGINT` | N | Internal | Platform-reported conversions. **Use with caution** — see gotcha below. |
| `utm_source` | `STRING` | Y | Internal | What the marketers tagged the campaign with. Joins to `fct_page_events.utm_source`. **Often missing or inconsistent for off-platform campaigns.** |
| `utm_medium` | `STRING` | Y | Internal |  |
| `utm_campaign` | `STRING` | Y | Internal |  |
| `start_date` | `DATE` | N | Internal | Campaign start. |
| `end_date` | `DATE` | Y | Internal | NULL for evergreen / always-on campaigns. |
| `is_active` | `BOOLEAN` | N | Internal | TRUE if campaign was active on `report_date`. |
| `loaded_at` | `TIMESTAMP` | N | Internal | When the row was loaded into the warehouse (UTC). |

### Allowed values

- `platform`: `google_ads | meta | tiktok | email | affiliate` (deprecated: `snapchat` — campaigns ended Q3 2024)
- `objective`: `awareness | consideration | conversion | retention`
- `region`: `us | ca | uk | au`

## Relationships

- `utm_source` / `utm_medium` / `utm_campaign` → `fct_page_events.utm_*` (N:N — same UTM tags can appear in many sessions).
- `campaign_id` → `fct_attribution.campaign_id` (1:N — many attributed orders per campaign).

## Gotchas

- **Platform `conversions` ≠ ABC Business orders.** Each ad platform has its own attribution window and methodology. Do **not** sum `conversions` and report it as "conversions"; use `fct_attribution` (ABC Business's last-touch model) for warehouse-of-record conversion counts. The `conversions` column here is useful for *platform optimization* questions ("did Meta spend efficiently this week?") but not for absolute counts.
- **Same-day data is unstable.** Ad platforms retroactively adjust spend, impressions, and clicks for up to 3 days. **A query run today for yesterday will return slightly different numbers tomorrow.** This is normal — don't open a bug.
- **Currency mixing.** `spend` is in the campaign's local currency. Summing across `region` without FX conversion is wrong.
- **`campaign_name` is not a key.** Marketers rename campaigns mid-flight ("Q1 BFCM Awareness" → "BFCM Awareness Final"). Always use `campaign_id`.
- **UTM coverage is uneven.** Email campaigns have `utm_*` set reliably. Affiliate campaigns frequently miss `utm_campaign`. Don't rely on UTMs for full attribution coverage — that's `fct_attribution`'s job.
- **Snapchat is deprecated.** Rows with `platform = 'snapchat'` exist for the historical record (2024 and earlier) but the campaigns ended Q3 2024. Don't report them in current-period analyses.

## Example queries

### Sanity check

```sql
SELECT
    MAX(report_date) AS latest_report_date
  , MAX(loaded_at) AS latest_load
FROM analytics_prod.staging.stg_marketing__campaigns;
```

### Last 30 days of spend by platform (single region)

```sql
SELECT
    report_date
  , platform
  , SUM(spend) AS total_spend
  , SUM(clicks) AS total_clicks
  , SUM(impressions) AS total_impressions
FROM analytics_prod.staging.stg_marketing__campaigns
WHERE report_date >= date_sub(current_date(), 30)
  AND region = 'us'
  AND platform != 'snapchat'  -- deprecated
GROUP BY report_date, platform
ORDER BY report_date, platform;
```

### Joining campaigns to first-order attribution (last-touch via UTM)

```sql
SELECT
    c.platform
  , c.campaign_name
  , SUM(c.spend) AS spend
  , COUNT(DISTINCT o.order_id) AS attributed_orders
  , SUM(o.order_subtotal) AS attributed_gmv
FROM analytics_prod.staging.stg_marketing__campaigns c
LEFT JOIN analytics_prod.core.fct_page_events e
  ON e.utm_source   = c.utm_source
 AND e.utm_medium   = c.utm_medium
 AND e.utm_campaign = c.utm_campaign
 AND e.event_date   = c.report_date
 AND e.is_synthetic = FALSE
LEFT JOIN analytics_prod.core.fct_orders o
  ON o.session_id   = e.session_id
 AND o.status       = 'completed'
WHERE c.report_date >= date_sub(current_date(), 30)
  AND c.region = 'us'
GROUP BY c.platform, c.campaign_name
ORDER BY attributed_gmv DESC NULLS LAST;
```

> Note: this query uses **last-touch session-level UTM matching**. For multi-touch attribution use `fct_attribution` directly.

## Migration / deprecation notes

- `2025-Q1`: Marketing Analytics announced they will deprecate the `platform = 'affiliate'` rows in favor of a separate `stg_marketing__affiliate_clicks` table sometime mid-2025. Watch `#marketing-analytics`.

## Change log

- `2025-01-15` (DS): Documented Snapchat deprecation and the platform-`conversions` vs `fct_attribution` distinction (after a stakeholder discrepancy).
- `2024-10-08` (Marketing Analytics): Added `currency` column.
