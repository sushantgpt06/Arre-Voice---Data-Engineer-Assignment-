# Section E: Final BigQuery Reporting Table Design

## Objective
Design the final table that product and leadership should use: `product_analytics.fact_reconciled_events`. The design must explain how duplicate events are flagged, invalid events are excluded, frontend/backend events are matched, and `final_metric_source` is selected.

---

## Table Specification

| Property | Value |
|----------|-------|
| **Dataset** | `product_analytics` |
| **Table Name** | `fact_reconciled_events` |
| **Purpose** | Single source of truth for all product analytics events |
| **Partitioning** | `event_date` (daily, UTC) |
| **Clustering** | `event_name`, `platform`, `final_metric_source` |
| **Refresh Strategy** | Incremental daily load with 48h late-arrival window |
| **Retention** | 90 days in hot storage, archive to GCS coldline afterwards |

---

## Schema

| # | Column Name | Data Type | Mode | Description | Derivation |
|---|------------|-----------|------|-------------|------------|
| 1 | `event_date` | DATE | REQUIRED | UTC event date | `DATE(event_timestamp, 'UTC')` |
| 2 | `event_timestamp` | TIMESTAMP | REQUIRED | Original event timestamp from source | Earliest among matched sources |
| 3 | `ingestion_timestamp` | TIMESTAMP | REQUIRED | When event first landed in BigQuery | From BQ raw events table |
| 4 | `source` | STRING | REPEATED | Comma-delimited list of sources recording this event | Aggregated across all matched rows |
| 5 | `event_name` | STRING | REQUIRED | Canonical normalized event name | Via `event_taxonomy` mapping table |
| 6 | `raw_event_name` | STRING | NULLABLE | Original event name from source | Source-specific |
| 7 | `user_id` | STRING | NULLABLE | Resolved user ID (post-login) | Priority: DynamoDB > Mixpanel > GA4 alias |
| 8 | `anonymous_id` | STRING | NULLABLE | Pre-login anonymous identifier | GA4 `user_pseudo_id` / Mixpanel pre-alias `distinct_id` |
| 9 | `device_id` | STRING | NULLABLE | Device fingerprint | GA4 `device_id` |
| 10 | `session_id` | STRING | NULLABLE | Session identifier | GA4 `ga_session_id` |
| 11 | `play_session_id` | STRING | NULLABLE | Audio playback session | Backend logs `play_session_id` |
| 12 | `content_id` | STRING | NULLABLE | Content identifier | Priority: DynamoDB > Backend logs > Frontend |
| 13 | `content_type` | STRING | NULLABLE | `pod` / `voicepool` / NULL | Priority: DynamoDB > Backend logs > Inferred from screen_view |
| 14 | `platform` | STRING | REQUIRED | `iOS` / `Android` / `Web` | Priority: GA4 > Mixpanel |
| 15 | `app_version` | STRING | REQUIRED | Mobile app version | Priority: Backend logs > GA4 |
| 16 | `environment` | STRING | REQUIRED | `production` / `staging` / `dev` | Enforced = `'production'` |
| 17 | `is_backend_confirmed` | BOOL | REQUIRED | TRUE if backend confirmed | Computed: EXISTS in DynamoDB OR backend_logs |
| 18 | `is_duplicate` | BOOL | REQUIRED | TRUE if duplicate within 30s window | Computed via ROW_NUMBER per user-event-content |
| 19 | `is_valid_event` | BOOL | REQUIRED | TRUE if passes all validation rules | Computed (see logic below) |
| 20 | `reconciliation_status` | STRING | REQUIRED | Match classification | `matched` / `missing_backend` / `missing_frontend` / `duplicate` / `invalid` |
| 21 | `final_metric_source` | STRING | REQUIRED | Source-of-truth for this metric | Per-metric SOT rule (see Section C) |

---

## Validation Logic

### `is_duplicate` Computation
```
ROW_NUMBER() OVER (
  PARTITION BY anonymous_id, event_name, content_id
  ORDER BY event_timestamp ASC
) = 1 --> is_duplicate = FALSE
ROW_NUMBER() > 1 AND seconds_since_last <= 30 --> is_duplicate = TRUE
```

### `is_valid_event` Computation
```
is_valid_event = TRUE IF ALL OF:
  - environment = 'production'
  - is_duplicate = FALSE
  - event_name IN event_allowlist
  - NOT (event_name IN ('heartbeat','ping','background_sync') AND content_id IS NULL)
```

### `reconciliation_status` Computation
```
IF is_duplicate = TRUE              --> 'duplicate'
ELSE IF is_valid_event = FALSE      --> 'invalid'
ELSE IF is_backend_confirmed = FALSE AND event_name LIKE '%play%' --> 'missing_backend'
ELSE IF source DOES NOT CONTAIN 'GA4' AND source DOES NOT CONTAIN 'Mixpanel' --> 'missing_frontend'
ELSE                                 --> 'matched'
```

### `final_metric_source` Computation
```
IF event_name IN ('signup_completed','otp_verified','user_created')
                                      --> 'DynamoDB'
ELSE IF event_name IN ('pod_play_started','voicepool_play','play_completed','listen_time')
                                      --> 'backend_logs'
ELSE IF event_name IN ('like','comment','share')
                                      --> 'Mixpanel'
ELSE IF event_name IN ('app_open','screen_view','campaign_attribution')
                                      --> 'GA4'
ELSE                                  --> 'BigQuery'
```

---

## DDL: Table Creation

```sql
-- ============================================================
-- PRODUCT_ANALYTICS.FACT_RECONCILED_EVENTS
-- ============================================================
-- Purpose: Single source of truth for reconciled product events
-- Refresh: Incremental daily (lookback 48h for late arrivals)
-- ============================================================

CREATE TABLE IF NOT EXISTS product_analytics.fact_reconciled_events (
  event_date              DATE        NOT NULL,
  event_timestamp         TIMESTAMP   NOT NULL,
  ingestion_timestamp     TIMESTAMP   NOT NULL,
  source                  STRING      NOT NULL,
  event_name              STRING      NOT NULL,
  raw_event_name          STRING,
  user_id                 STRING,
  anonymous_id            STRING,
  device_id               STRING,
  session_id              STRING,
  play_session_id         STRING,
  content_id              STRING,
  content_type            STRING,
  platform                STRING      NOT NULL,
  app_version             STRING      NOT NULL,
  environment             STRING      NOT NULL,
  is_backend_confirmed    BOOL        NOT NULL,
  is_duplicate            BOOL        NOT NULL,
  is_valid_event          BOOL        NOT NULL,
  reconciliation_status   STRING      NOT NULL,
  final_metric_source     STRING      NOT NULL
)
PARTITION BY event_date
CLUSTER BY event_name, platform, final_metric_source;
```

---

## DML: Incremental Load (Production Pattern)

```sql
-- ============================================================
-- INCREMENTAL MERGE FOR FACT_RECONCILED_EVENTS
-- Run daily at 03:00 UTC (48h after event_date for late arrivals)
-- ============================================================

MERGE product_analytics.fact_reconciled_events T
USING (
  -- Full reconciliation logic (simplified; production uses modular CTEs)
  SELECT
    DATE(e.event_timestamp, 'UTC') AS event_date,
    e.event_timestamp,
    e.ingestion_timestamp,
    e.source,
    t.canonical_name AS event_name,
    e.raw_event_name,
    COALESCE(d.user_id, e.user_id, m.distinct_id) AS user_id,
    COALESCE(e.anonymous_id, m.distinct_id, ga.user_pseudo_id) AS anonymous_id,
    ga.device_id,
    ga.session_id,
    bl.play_session_id,
    COALESCE(d.content_id, bl.content_id, e.content_id) AS content_id,
    COALESCE(d.content_type, bl.content_type, e.content_type) AS content_type,
    COALESCE(ga.platform, m.platform, e.platform) AS platform,
    COALESCE(bl.app_version, ga.app_version, m.app_version, e.app_version) AS app_version,
    'production' AS environment,
    (d.event_id IS NOT NULL OR bl.log_id IS NOT NULL) AS is_backend_confirmed,
    -- Duplicate flag computed in separate CTE
    dup.is_duplicate,
    -- Validity computed in separate CTE
    val.is_valid_event,
    CASE
      WHEN dup.is_duplicate THEN 'duplicate'
      WHEN val.is_valid_event = FALSE THEN 'invalid'
      WHEN d.event_id IS NULL AND bl.log_id IS NULL AND t.canonical_name LIKE '%play%' THEN 'missing_backend'
      ELSE 'matched'
    END AS reconciliation_status,
    CASE
      WHEN t.canonical_name IN ('signup_completed','otp_verified','user_created') THEN 'DynamoDB'
      WHEN t.canonical_name IN ('pod_play_started','voicepool_play','play_completed','listen_time') THEN 'backend_logs'
      WHEN t.canonical_name IN ('like','comment','share') THEN 'Mixpanel'
      WHEN t.canonical_name IN ('app_open','screen_view','campaign_attribution') THEN 'GA4'
      ELSE 'BigQuery'
    END AS final_metric_source
  FROM staging.raw_events e
  LEFT JOIN event_taxonomy t ON LOWER(e.raw_event_name) = LOWER(t.source_name)
  LEFT JOIN ga4_events ga ON e.event_id = ga.event_id
  LEFT JOIN mixpanel_events m ON e.event_id = m.mp_event_id
  LEFT JOIN dynamodb_events d ON e.event_id = d.event_id
  LEFT JOIN backend_logs bl ON e.event_id = bl.event_id
  LEFT JOIN dedup_flags dup ON e.event_id = dup.event_id
  LEFT JOIN validity_flags val ON e.event_id = val.event_id
  WHERE DATE(e.event_timestamp, 'UTC') BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 3 DAY)
                                              AND DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
    AND e.environment = 'production'
) S
ON T.event_date = S.event_date AND T.anonymous_id = S.anonymous_id AND T.event_timestamp = S.event_timestamp
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

---

## Supporting Tables Required

| Table | Purpose | Columns |
|-------|---------|---------|
| `event_taxonomy` | Maps raw source event names to canonical names | `source_name`, `canonical_name`, `category`, `is_user_initiated` |
| `event_allowlist` | Defines events included in fair comparison | `event_name`, `source`, `included_in_comparison` |
| `identity_mapping` | Anonymous to user ID stitching | `anonymous_id`, `user_id`, `alias_timestamp`, `confidence_score` |
| `content_metadata` | Content duration and type enrichment | `content_id`, `content_type`, `duration_seconds`, `creator_id` |

---

## Downstream Reporting Views

### Daily Metrics View (for Leadership Dashboards)
```sql
CREATE VIEW product_analytics.vw_daily_metrics AS
SELECT
  event_date,
  final_metric_source,
  event_name,
  platform,
  COUNT(DISTINCT user_id) AS unique_users,
  COUNT(*) AS event_count
FROM product_analytics.fact_reconciled_events
WHERE is_valid_event = TRUE
  AND environment = 'production'
GROUP BY event_date, final_metric_source, event_name, platform;
```

---

[Back to Index](SUBMISSION_README.md)
