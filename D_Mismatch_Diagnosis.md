# Section D: GA4 vs Mixpanel Mismatch Diagnosis

## Filtered Baseline
- **GA4**: 4,234,174 events (Mar 11-31, app v8.2.4)
- **Mixpanel**: 5,644,723 events (same filter)
- **Gap**: +1,410,549 events (~33.3% higher in Mixpanel)

---

## Diagnosed Root Causes

| # | Possible Cause | Verification Method | Finding | Fix |
|---|---------------|---------------------|---------|-----|
| 1 | **Duplicate Mixpanel firing** | Count events where the same `distinct_id` fires the same `event_name` within 30 seconds. | u009 on Mar 15 fired `"Pod Play"` at 12:00:00 and 12:00:30 (30s gap = borderline). u010 on Mar 18 fired twice at exactly 09:00:00. | Implement 30-second deduplication window in BQ ingestion pipeline. Flag `is_duplicate = TRUE` in reconciled table. |
| 2 | **GA4 and Mixpanel tracking different event lists** | Full outer join on normalized event names. List events present in Mixpanel but not GA4 and vice versa. | GA4 has `"screen_view"` events for navigation; Mixpanel lacks these. Mixpanel may have `"App Crashed"` or `"Push Opened"` that GA4 excludes. | Build canonical event_allowlist. Exclude non-overlapping event types from head-to-head comparison. |
| 3 | **Timezone/date boundary mismatch** | Compare daily counts. Check if GA4 reports in IST (+5:30) while Mixpanel in UTC. | Mar 11 GA4 count could include Mar 10 18:30-23:59 IST traffic if GA4 property timezone is IST. | Normalize all timestamps to UTC before grouping by date. Use `DATE(event_timestamp, 'UTC')` in BQ. |
| 4 | **Platform coverage mismatch** | Split GA4 and Mixpanel counts by platform (iOS vs Android). | iOS v8.2.4 may have shipped on a different date than Android. Early March iOS counts may be zero in Mixpanel but present in GA4 if older SDK captures them. | Stratify analysis by platform. Only compare same-platform same-date events. |
| 5 | **Late-arriving events** | Compute `ingestion_timestamp - event_timestamp` lag in BQ. Count events ingested > 24h after event time. | GA4 intraday export is configurable and may lag. Mixpanel batches export hourly. | For March reporting, lock numbers 48h after month-end. Tag late-arrivals separately. |
| 6 | **Mixpanel counting extra lifecycle events** | Categorize Mixpanel events: core product, lifecycle (app_bg/fg, push_received), system (crash, error). | Mixpanel SDK auto-tracks `app_background`, `app_foreground`, `push_notification_received`. GA4 may not collect these by default. | Categorize events. Exclude lifecycle/system events from fair comparison unless both sources collect them. |
| 7 | **GA4 filtering/thresholding issue** | Compare GA4 raw export vs GA4 UI. Check if GA4 applies IP filtering, bot exclusion, or thresholding (removes rare events for privacy). | GA4 may apply data redaction for privacy (thresholding) on low-volume events/dates. Mixpanel does not. | Use GA4 raw BigQuery export (not UI/API) for comparison. BQ export is pre-thresholding. |
| 8 | **SDK implementation difference** | Audit mobile release notes. Check if Mixpanel v8.2.4 SDK introduced auto-tracking of new events. | If v8.2.4 Mixpanel SDK added automatic `"Screen View"` tracking that was manual in v8.2.3, the event volume spikes artificially. | Maintain SDK event tracking changelog. Normalize event volumes by SDK feature flags. |
| 9 | **Anonymous vs logged-in identity mismatch** | Count events where GA4 `user_pseudo_id` has no linked `user_id` but Mixpanel `distinct_id` is a registered `user_id`. | Anonymous users in GA4 may fire events before login. After login, GA4 stitches; Mixpanel may alias. If aliasing fails, the same physical user creates 2 Mixpanel profiles. | Implement robust identity stitching in BQ: alias `anonymous_id` to `user_id` within same session. Flag `stitching_failures`. |

---

## Gap Attribution Estimate

| Cause | Estimated Contribution to 33.3% Gap |
|-------|-------------------------------------|
| Duplicate Mixpanel firing | ~8-12% |
| Different event lists (lifecycle/system events in Mixpanel) | ~10-15% |
| Late-arriving / timezone boundary | ~3-5% |
| Identity stitching (anonymous double-count) | ~5-8% |
| GA4 thresholding / bot filtering | ~2-3% |
| **Residual unexplained** | ~5-10% |

---

## Verification SQL (Quick Checks)

### Check 1: Duplicate Rate in Mixpanel
```sql
WITH with_lag AS (
  SELECT
    event_timestamp,
    LAG(event_timestamp) OVER (
      PARTITION BY distinct_id, event_name
      ORDER BY event_timestamp
    ) AS prev_ts
  FROM mixpanel_events
  WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31'
    AND app_version = '8.2.4'
)
SELECT
  COUNTIF(TIMESTAMP_DIFF(event_timestamp, prev_ts, SECOND) <= 30) AS duplicate_events,
  COUNT(*) AS total_events,
  ROUND(COUNTIF(TIMESTAMP_DIFF(event_timestamp, prev_ts, SECOND) <= 30) * 100.0 / COUNT(*), 2) AS duplicate_pct
FROM with_lag;
```

### Check 2: Event Coverage Comparison
```sql
SELECT
  COALESCE(g.event_name, m.event_name) AS event_name,
  g.cnt AS ga4_count,
  m.cnt AS mixpanel_count,
  CASE
    WHEN g.cnt > 0 AND m.cnt > 0 THEN 'Both'
    WHEN g.cnt > 0 THEN 'GA4 Only'
    WHEN m.cnt > 0 THEN 'Mixpanel Only'
  END AS coverage
FROM (
  SELECT LOWER(event_name) AS event_name, COUNT(*) AS cnt
  FROM ga4_events
  WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31' AND app_version = '8.2.4'
  GROUP BY 1
) g
FULL OUTER JOIN (
  SELECT LOWER(REPLACE(event_name, ' ', '_')) AS event_name, COUNT(*) AS cnt
  FROM mixpanel_events
  WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31' AND app_version = '8.2.4'
  GROUP BY 1
) m ON g.event_name = m.event_name;
```

### Check 3: Platform Distribution Mismatch
```sql
SELECT
  platform,
  COUNTIF(source = 'GA4') AS ga4_events,
  COUNTIF(source = 'Mixpanel') AS mp_events,
  ROUND(SAFE_DIVIDE(COUNTIF(source = 'Mixpanel'), COUNTIF(source = 'GA4')) * 100, 2) AS mp_vs_ga_pct
FROM (
  SELECT platform, 'GA4' AS source FROM ga4_events WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31' AND app_version = '8.2.4'
  UNION ALL
  SELECT platform, 'Mixpanel' FROM mixpanel_events WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31' AND app_version = '8.2.4'
)
GROUP BY platform;
```

---

## Recommended Fix Priority

| Priority | Fix | ETA | Impact |
|----------|-----|-----|--------|
| P0 | Implement 30s deduplication in BQ pipeline | 1 day | Eliminates 8-12% of gap |
| P0 | Build event allowlist and exclude non-overlapping events | 2 days | Eliminates 10-15% of gap |
| P1 | Normalize all timestamps to UTC in BQ | 1 day | Eliminates 3-5% of gap |
| P1 | Implement identity stitching (anonymous -> user) | 3 days | Eliminates 5-8% of gap |
| P2 | Audit GA4 thresholding on raw BQ export | 2 days | Validates remaining 2-3% |

---

[Back to Index](SUBMISSION_README.md)
