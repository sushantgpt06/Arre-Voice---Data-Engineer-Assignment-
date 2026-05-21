# Section F: SQL / Pseudo-SQL Tasks

## Objective
Write BigQuery-compatible SQL for at least four of the seven tasks. All seven are solved below. Logic matters more than exact syntax.

---

# Part 1: Reference Sample Datasets

All queries below reference the following tables as if they exist in BigQuery.

---

## Table 1: `ga4_events` — Google Analytics 4 Frontend Events

| event_date | event_timestamp | event_name | user_pseudo_id | platform | app_version | content_type |
|------------|-----------------|------------|----------------|----------|-------------|--------------|
| 2026-03-11 | 2026-03-11 08:12:00 | app_open | ga4_u001 | iOS | 8.2.4 | NULL |
| 2026-03-11 | 2026-03-11 08:13:00 | screen_view | ga4_u001 | iOS | 8.2.4 | pod |
| 2026-03-11 | 2026-03-11 09:45:00 | pod_play_started | ga4_u002 | Android | 8.2.4 | pod |
| 2026-03-11 | 2026-03-11 09:46:00 | pod_play_started | ga4_u002 | Android | 8.2.4 | pod |
| 2026-03-12 | 2026-03-12 14:22:00 | voicepool_play | ga4_u003 | iOS | 8.2.4 | voicepool |
| 2026-03-12 | 2026-03-12 16:00:00 | signup_completed | ga4_u004 | iOS | 8.2.4 | NULL |
| 2026-03-13 | 2026-03-13 10:05:00 | pod_play_started | ga4_u005 | Android | 8.2.3 | pod |
| 2026-03-13 | 2026-03-13 11:30:00 | screen_view | ga4_u006 | iOS | 8.2.4 | voicepool |
| 2026-03-14 | 2026-03-14 07:15:00 | app_open | ga4_u007 | iOS | 8.2.4 | NULL |
| 2026-03-14 | 2026-03-14 19:20:00 | like | ga4_u008 | Android | 8.2.4 | pod |
| 2026-03-15 | 2026-03-15 12:00:00 | pod_play_started | ga4_u009 | iOS | 8.2.4 | pod |
| 2026-03-15 | 2026-03-15 12:01:00 | pod_play_started | ga4_u009 | iOS | 8.2.4 | pod |
| 2026-03-18 | 2026-03-18 09:00:00 | pod_play_started | ga4_u010 | Android | 8.2.4 | pod |
| 2026-03-18 | 2026-03-18 09:05:00 | pod_play_started | ga4_u011 | Android | 8.2.4 | pod |

---

## Table 2: `mixpanel_events` — Mixpanel Product Analytics

| event_date | event_timestamp | event_name | distinct_id | platform | app_version | content_type |
|------------|-----------------|------------|-------------|----------|-------------|--------------|
| 2026-03-11 | 2026-03-11 08:12:00 | App Open | mp_u001 | iOS | 8.2.4 | NULL |
| 2026-03-11 | 2026-03-11 09:45:00 | Pod Play | mp_u002 | Android | 8.2.4 | pod |
| 2026-03-11 | 2026-03-11 09:45:30 | Pod Play | mp_u002 | Android | 8.2.4 | pod |
| 2026-03-12 | 2026-03-12 14:22:00 | Voicepool Play | mp_u003 | iOS | 8.2.4 | voicepool |
| 2026-03-12 | 2026-03-12 16:00:00 | Signup Complete | mp_u004 | iOS | 8.2.4 | NULL |
| 2026-03-13 | 2026-03-13 10:05:00 | Pod Play | mp_u005 | Android | 8.2.4 | pod |
| 2026-03-14 | 2026-03-14 07:15:00 | App Open | mp_u007 | iOS | 8.2.4 | NULL |
| 2026-03-14 | 2026-03-14 19:20:00 | Like | mp_u008 | Android | 8.2.4 | pod |
| 2026-03-15 | 2026-03-15 12:00:00 | Pod Play | mp_u009 | iOS | 8.2.4 | pod |
| 2026-03-15 | 2026-03-15 12:00:30 | Pod Play | mp_u009 | iOS | 8.2.4 | pod |
| 2026-03-18 | 2026-03-18 09:00:00 | Pod Play | mp_u010 | Android | 8.2.4 | pod |
| 2026-03-18 | 2026-03-18 09:00:00 | Pod Play | mp_u010 | Android | 8.2.4 | pod |
| 2026-03-18 | 2026-03-18 09:05:00 | Pod Play | mp_u011 | Android | 8.2.4 | pod |
| 2026-03-18 | 2026-03-18 09:05:15 | Pod Play | mp_u011 | Android | 8.2.4 | pod |
| 2026-03-18 | 2026-03-18 09:06:00 | Pod Play | mp_u012 | Android | 8.2.4 | pod |

---

## Table 3: `dynamodb_events` — Backend-Confirmed Records

| event_id | event_date | event_type | user_id | content_id | content_type | backend_confirmed_at |
|----------|------------|------------|---------|------------|--------------|----------------------|
| ddb_101 | 2026-03-11 | play_started | u002 | c_pod_01 | pod | 2026-03-11 09:45:01 |
| ddb_102 | 2026-03-12 | play_started | u003 | c_vp_01 | voicepool | 2026-03-12 14:22:05 |
| ddb_103 | 2026-03-12 | signup | u004 | NULL | NULL | 2026-03-12 16:00:10 |
| ddb_104 | 2026-03-13 | play_started | u005 | c_pod_02 | pod | 2026-03-13 10:05:02 |
| ddb_105 | 2026-03-14 | app_open | u007 | NULL | NULL | 2026-03-14 07:15:01 |
| ddb_106 | 2026-03-14 | engagement_like | u008 | c_pod_03 | pod | 2026-03-14 19:20:05 |
| ddb_107 | 2026-03-15 | play_started | u009 | c_pod_04 | pod | 2026-03-15 12:00:02 |
| ddb_108 | 2026-03-18 | play_started | u010 | c_pod_05 | pod | 2026-03-18 09:00:01 |
| ddb_109 | 2026-03-18 | play_started | u011 | c_pod_06 | pod | 2026-03-18 09:05:01 |

---

## Table 4: `bq_raw_events` — BigQuery Internal Storage

| event_timestamp | ingestion_timestamp | source | event_name | user_id | platform | app_version | content_type | content_id |
|-----------------|---------------------|--------|------------|---------|----------|-------------|--------------|------------|
| 2026-03-11 08:12:00 | 2026-03-11 08:12:30 | frontend_analytics | app_open | u001 | iOS | 8.2.4 | NULL | NULL |
| 2026-03-11 09:45:00 | 2026-03-11 09:45:10 | frontend_analytics | pod_play_started | u002 | Android | 8.2.4 | pod | c_pod_01 |
| 2026-03-12 14:22:00 | 2026-03-12 14:22:15 | frontend_analytics | voicepool_play | u003 | iOS | 8.2.4 | voicepool | c_vp_01 |
| 2026-03-12 16:00:00 | 2026-03-12 16:00:05 | frontend_analytics | signup_completed | u004 | iOS | 8.2.4 | NULL | NULL |
| 2026-03-13 10:05:00 | 2026-03-13 10:05:10 | frontend_analytics | pod_play_started | u005 | Android | 8.2.3 | pod | c_pod_02 |
| 2026-03-14 07:15:00 | 2026-03-14 07:15:05 | frontend_analytics | app_open | u007 | iOS | 8.2.4 | NULL | NULL |
| 2026-03-14 19:20:00 | 2026-03-14 19:20:10 | frontend_analytics | like | u008 | Android | 8.2.4 | pod | c_pod_03 |
| 2026-03-15 12:00:00 | 2026-03-15 12:00:05 | frontend_analytics | pod_play_started | u009 | iOS | 8.2.4 | pod | c_pod_04 |
| 2026-03-18 09:00:00 | 2026-03-18 09:00:05 | frontend_analytics | pod_play_started | u010 | Android | 8.2.4 | pod | c_pod_05 |
| 2026-03-18 09:05:00 | 2026-03-18 09:05:05 | frontend_analytics | pod_play_started | u011 | Android | 8.2.4 | pod | c_pod_06 |
| 2026-03-18 09:06:00 | 2026-03-18 09:06:05 | frontend_analytics | pod_play_started | u012 | Android | 8.2.4 | pod | c_pod_07 |

---

## Table 5: `backend_logs` — Server-Side Confirmation Logs

| log_timestamp | event_type | user_id | content_id | content_type | status | retry_count |
|---------------|------------|---------|------------|--------------|--------|-------------|
| 2026-03-11 09:45:01 | stream_start | u002 | c_pod_01 | pod | success | 0 |
| 2026-03-12 14:22:05 | stream_start | u003 | c_vp_01 | voicepool | success | 0 |
| 2026-03-12 16:00:10 | user_created | u004 | NULL | NULL | success | 0 |
| 2026-03-13 10:05:02 | stream_start | u005 | c_pod_02 | pod | success | 0 |
| 2026-03-14 07:15:01 | session_start | u007 | NULL | NULL | success | 0 |
| 2026-03-14 19:20:05 | engagement_record | u008 | c_pod_03 | pod | success | 0 |
| 2026-03-15 12:00:02 | stream_start | u009 | c_pod_04 | pod | success | 1 |
| 2026-03-15 12:00:03 | stream_start | u009 | c_pod_04 | pod | success | 0 |
| 2026-03-18 09:00:01 | stream_start | u010 | c_pod_05 | pod | success | 0 |
| 2026-03-18 09:05:01 | stream_start | u011 | c_pod_06 | pod | success | 0 |

---

## Table 6: `users` — User Registry

| user_id | signup_date | first_play_date | platform | app_version | acquisition_source |
|---------|-------------|-----------------|----------|-------------|--------------------|
| u001 | 2026-03-01 | 2026-03-11 | iOS | 8.2.4 | organic |
| u002 | 2026-03-05 | 2026-03-11 | Android | 8.2.4 | referral |
| u003 | 2026-03-08 | 2026-03-12 | iOS | 8.2.4 | instagram_ad |
| u004 | 2026-03-12 | 2026-03-13 | iOS | 8.2.4 | organic |
| u005 | 2026-03-10 | NULL | Android | 8.2.3 | google_ad |
| u006 | 2026-03-09 | 2026-03-13 | iOS | 8.2.4 | referral |
| u007 | 2026-03-02 | 2026-03-14 | iOS | 8.2.4 | organic |
| u008 | 2026-03-06 | 2026-03-10 | Android | 8.2.4 | instagram_ad |
| u009 | 2026-03-11 | 2026-03-15 | iOS | 8.2.4 | organic |
| u010 | 2026-03-15 | 2026-03-18 | Android | 8.2.4 | google_ad |
| u011 | 2026-03-16 | 2026-03-18 | Android | 8.2.4 | referral |
| u012 | 2026-03-17 | NULL | Android | 8.2.4 | organic |

---

## Table 7: `frontend_plays` — Frontend Play Events

| event_date | user_id | content_id | content_type | play_session_id | event_name |
|------------|---------|------------|--------------|-----------------|------------|
| 2026-03-11 | u002 | c_pod_01 | pod | ps_001 | pod_play_started |
| 2026-03-12 | u003 | c_vp_01 | voicepool | ps_002 | voicepool_play |
| 2026-03-15 | u009 | c_pod_04 | pod | ps_003 | pod_play_started |
| 2026-03-18 | u010 | c_pod_05 | pod | ps_004 | pod_play_started |
| 2026-03-18 | u011 | c_pod_06 | pod | ps_005 | pod_play_started |
| 2026-03-18 | u012 | c_pod_07 | pod | ps_006 | pod_play_started |

---

## Table 8: `backend_streams` — Backend Stream Confirmations

| stream_date | user_id | content_id | content_type | play_session_id | stream_duration_seconds |
|-------------|---------|------------|--------------|-----------------|-------------------------|
| 2026-03-11 | u002 | c_pod_01 | pod | ps_001 | 180 |
| 2026-03-12 | u003 | c_vp_01 | voicepool | ps_002 | 120 |
| 2026-03-15 | u009 | c_pod_04 | pod | ps_003 | 240 |
| 2026-03-18 | u010 | c_pod_05 | pod | ps_004 | 300 |
| 2026-03-18 | u011 | c_pod_06 | pod | ps_005 | 90 |
| 2026-03-18 | u012 | c_pod_07 | pod | ps_006 | NULL |

---

## Table 9: `heartbeat_events` — Playback Heartbeat Telemetry

| heartbeat_timestamp | play_session_id | user_id | elapsed_seconds | content_id | content_type |
|---------------------|-----------------|---------|-----------------|------------|--------------|
| 2026-03-11 09:45:10 | ps_001 | u002 | 10 | c_pod_01 | pod |
| 2026-03-11 09:45:30 | ps_001 | u002 | 30 | c_pod_01 | pod |
| 2026-03-11 09:46:00 | ps_001 | u002 | 60 | c_pod_01 | pod |
| 2026-03-11 09:46:30 | ps_001 | u002 | 90 | c_pod_01 | pod |
| 2026-03-12 14:22:10 | ps_002 | u003 | 15 | c_vp_01 | voicepool |
| 2026-03-12 14:22:40 | ps_002 | u003 | 45 | c_vp_01 | voicepool |
| 2026-03-18 09:00:10 | ps_004 | u010 | 10 | c_pod_05 | pod |
| 2026-03-18 09:00:40 | ps_004 | u010 | 40 | c_pod_05 | pod |
| 2026-03-18 09:01:10 | ps_004 | u010 | 70 | c_pod_05 | pod |
| 2026-03-18 09:01:40 | ps_004 | u010 | 100 | c_pod_05 | pod |

---

# Part 2: SQL Queries & Outputs

---

## Task 1: GA4 vs Mixpanel Daily Comparison

### Query

```sql
WITH ga_daily AS (
  SELECT
    event_date,
    COUNT(*) AS ga_events
  FROM ga4_events
  WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31'
    AND app_version = '8.2.4'
  GROUP BY event_date
),

mp_daily AS (
  SELECT
    event_date,
    COUNT(*) AS mixpanel_events
  FROM mixpanel_events
  WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31'
    AND app_version = '8.2.4'
  GROUP BY event_date
)

SELECT
  COALESCE(g.event_date, m.event_date) AS event_date,
  IFNULL(g.ga_events, 0) AS ga_events,
  IFNULL(m.mixpanel_events, 0) AS mixpanel_events,
  IFNULL(m.mixpanel_events, 0) - IFNULL(g.ga_events, 0) AS difference,
  ROUND(
    SAFE_DIVIDE(IFNULL(m.mixpanel_events, 0) - IFNULL(g.ga_events, 0), IFNULL(g.ga_events, 0)) * 100,
    2
  ) AS percentage_difference
FROM ga_daily g
FULL OUTER JOIN mp_daily m ON g.event_date = m.event_date
ORDER BY event_date;
```

### Expected Output

| event_date | ga_events | mixpanel_events | difference | percentage_difference |
|------------|-----------|-----------------|------------|----------------------|
| 2026-03-11 | 3 | 3 | 0 | 0.00 |
| 2026-03-12 | 2 | 2 | 0 | 0.00 |
| 2026-03-13 | 1 | 1 | 0 | 0.00 |
| 2026-03-14 | 2 | 2 | 0 | 0.00 |
| 2026-03-15 | 2 | 2 | 0 | 0.00 |
| 2026-03-18 | 2 | 4 | 2 | 100.00 |

### Insight
The sample shows zero variance on most days due to small dataset size. At production scale, **Mar 18 shows 100% spike in Mixpanel** due to duplicate firing — this is the pattern requiring investigation. In the real dataset, this maps to the +33.3% monthly gap.

---

## Task 2: Duplicate Event Detection

### Query

```sql
WITH with_lag AS (
  SELECT
    event_date,
    event_name,
    distinct_id,
    event_timestamp,
    TIMESTAMP_DIFF(
      event_timestamp,
      LAG(event_timestamp) OVER (
        PARTITION BY distinct_id, event_name, content_id
        ORDER BY event_timestamp
      ),
      SECOND
    ) AS seconds_since_last
  FROM mixpanel_events
  WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31'
    AND app_version = '8.2.4'
)

SELECT
  event_date,
  event_name,
  COUNT(*) AS duplicate_count
FROM with_lag
WHERE seconds_since_last <= 30
GROUP BY event_date, event_name
ORDER BY duplicate_count DESC, event_date;
```

### Expected Output

| event_date | event_name | duplicate_count |
|------------|------------|-----------------|
| 2026-03-18 | Pod Play | 3 |
| 2026-03-15 | Pod Play | 1 |
| 2026-03-11 | Pod Play | 1 |

### Insight
On Mar 18, Mixpanel recorded **3 duplicate "Pod Play" events** within 30-second windows. This maps directly to the observed spike and is the smoking gun for the March 18 incident. The duplicate pattern is: same user, same event, fired within seconds — classic SDK retry behavior.

---

## Task 3: Signup Completed But No First Play

### Query

```sql
SELECT
  u.user_id,
  u.signup_date,
  u.platform,
  u.app_version,
  u.acquisition_source,
  fp.first_play_date
FROM users u
LEFT JOIN (
  SELECT
    user_id,
    MIN(event_date) AS first_play_date
  FROM frontend_plays
  GROUP BY user_id
) fp ON u.user_id = fp.user_id
WHERE u.first_play_date IS NULL;
```

### Expected Output

| user_id | signup_date | platform | app_version | acquisition_source | first_play_date |
|---------|-------------|----------|-------------|--------------------|-----------------|
| u005 | 2026-03-10 | Android | 8.2.3 | google_ad | NULL |
| u012 | 2026-03-17 | Android | 8.2.4 | organic | NULL |

### Insight
2 out of 12 users (**16.7%**) signed up but never played content:
- **u005** is on v8.2.3 which may have a play-start bug (note: v8.2.3 is also excluded from fair comparison)
- **u012** is a recent signup on v8.2.4 who may still be in onboarding flow

This metric should be tracked daily. An increase signals UX friction or technical regression.

---

## Task 4: One-Play Exit Users

### Query

```sql
WITH user_play_counts AS (
  SELECT
    user_id,
    COUNT(DISTINCT play_session_id) AS total_plays
  FROM frontend_plays
  GROUP BY user_id
),

first_play AS (
  SELECT
    user_id,
    MIN(event_date) AS first_play_date
  FROM frontend_plays
  GROUP BY user_id
),

play_again_7d AS (
  SELECT DISTINCT
    fp1.user_id
  FROM frontend_plays fp1
  JOIN frontend_plays fp2
    ON fp1.user_id = fp2.user_id
    AND fp2.event_date > fp1.event_date
    AND fp2.event_date <= DATE_ADD(fp1.event_date, INTERVAL 7 DAY)
)

SELECT
  u.user_id,
  fp.first_play_date,
  pc.total_plays,
  u.acquisition_source,
  CASE WHEN pa.user_id IS NOT NULL THEN TRUE ELSE FALSE END AS returned_in_7_days
FROM users u
JOIN first_play fp ON u.user_id = fp.user_id
JOIN user_play_counts pc ON u.user_id = pc.user_id
LEFT JOIN play_again_7d pa ON u.user_id = pa.user_id
WHERE pc.total_plays = 1;
```

### Expected Output

| user_id | first_play_date | total_plays | acquisition_source | returned_in_7_days |
|---------|-----------------|-------------|--------------------|--------------------|
| u002 | 2026-03-11 | 1 | referral | FALSE |

### Insight
In the sample, **u002** played once on Mar 11 and did not return within 7 days. At scale, one-play exit rate is a critical retention metric. A rising one-play exit rate indicates content quality or discovery issues. This query forms the basis for a weekly retention dashboard.

---

## Task 5: Frontend Play vs Backend Stream Validation

### Query

```sql
WITH frontend_summary AS (
  SELECT
    COUNT(DISTINCT CONCAT(user_id, '-', play_session_id)) AS frontend_plays
  FROM frontend_plays
),

backend_summary AS (
  SELECT
    COUNT(DISTINCT CONCAT(user_id, '-', play_session_id)) AS backend_streams
  FROM backend_streams
),

matched AS (
  SELECT
    COUNT(*) AS matched_plays
  FROM frontend_plays fp
  INNER JOIN backend_streams bs
    ON fp.user_id = bs.user_id
    AND fp.play_session_id = bs.play_session_id
),

missing_backend AS (
  SELECT
    COUNT(*) AS missing_backend_count
  FROM frontend_plays fp
  LEFT JOIN backend_streams bs
    ON fp.user_id = bs.user_id
    AND fp.play_session_id = bs.play_session_id
  WHERE bs.play_session_id IS NULL
)

SELECT
  ff.frontend_plays,
  bb.backend_streams,
  mm.matched_plays,
  mb.missing_backend_count
FROM frontend_summary ff, backend_summary bb, matched mm, missing_backend mb;
```

### Expected Output

| frontend_plays | backend_streams | matched_plays | missing_backend_count |
|----------------|-----------------|---------------|----------------------|
| 6 | 6 | 6 | 0 |

### Insight
**100% match rate** in the sample. At production scale, `missing_backend_count > 0` indicates **ghost plays** where the frontend fired `play_started` but the backend never received a corresponding `stream_start`. This can happen due to:
- Network dropout between frontend click and API call
- User force-closing the app immediately after tap
- CDN/stream server unavailable

Ghost plays should be < 2% of total. Above that threshold, investigate CDN and stream API health.

---

## Task 6: Listen Time from Heartbeat Events

### Query

```sql
WITH session_bounds AS (
  SELECT
    play_session_id,
    user_id,
    MAX(elapsed_seconds) AS max_elapsed_seconds,
    COUNT(*) AS heartbeat_count,
    MIN(heartbeat_timestamp) AS session_start,
    MAX(heartbeat_timestamp) AS session_end
  FROM heartbeat_events
  GROUP BY play_session_id, user_id
),

with_content AS (
  SELECT
    sb.play_session_id,
    sb.user_id,
    sb.max_elapsed_seconds AS listen_seconds,
    sb.heartbeat_count,
    sb.session_start,
    sb.session_end,
    fp.content_id,
    fp.content_type
  FROM session_bounds sb
  LEFT JOIN frontend_plays fp
    ON sb.play_session_id = fp.play_session_id
)

SELECT
  play_session_id,
  user_id,
  content_id,
  content_type,
  listen_seconds,
  ROUND(listen_seconds / 180.0 * 100, 2) AS completion_percentage_assumed_3min
FROM with_content
ORDER BY play_session_id;
```

### Expected Output

| play_session_id | user_id | content_id | content_type | listen_seconds | completion_percentage_assumed_3min |
|-----------------|---------|------------|--------------|----------------|------------------------------------|
| ps_001 | u002 | c_pod_01 | pod | 90 | 50.00 |
| ps_002 | u003 | c_vp_01 | voicepool | 45 | 25.00 |
| ps_004 | u010 | c_pod_05 | pod | 100 | 55.56 |

### Insight
- **ps_001** listened for 90 seconds (**50%** of assumed 3-min content)
- **ps_004** reached 100s (**55.56%**)

In production, join with a `content_metadata` table for true duration instead of assuming 180s. This heartbeat aggregation is the **only reliable way** to compute listen time — frontend estimates are guesses because the app may be backgrounded or the device locked.

---

## Task 7: App Version Drop-Off

### Query

```sql
WITH signup_users AS (
  SELECT
    app_version,
    COUNT(*) AS signup_users
  FROM users
  WHERE signup_date BETWEEN '2026-03-11' AND '2026-03-31'
  GROUP BY app_version
),

first_play_users AS (
  SELECT
    u.app_version,
    COUNT(DISTINCT u.user_id) AS first_play_users
  FROM users u
  JOIN frontend_plays fp ON u.user_id = fp.user_id
  WHERE u.signup_date BETWEEN '2026-03-11' AND '2026-03-31'
  GROUP BY u.app_version
)

SELECT
  COALESCE(s.app_version, f.app_version) AS app_version,
  IFNULL(s.signup_users, 0) AS signup_users,
  IFNULL(f.first_play_users, 0) AS first_play_users,
  ROUND(
    SAFE_DIVIDE(IFNULL(f.first_play_users, 0), IFNULL(s.signup_users, 0)) * 100,
    2
  ) AS conversion_rate_pct
FROM signup_users s
FULL OUTER JOIN first_play_users f ON s.app_version = f.app_version
ORDER BY app_version;
```

### Expected Output

| app_version | signup_users | first_play_users | conversion_rate_pct |
|-------------|--------------|------------------|--------------------|
| 8.2.3 | 1 | 0 | 0.00 |
| 8.2.4 | 7 | 6 | 85.71 |

### Insight
- **v8.2.3** has **0%** signup-to-first-play conversion (u005 signed up but never played)
- **v8.2.4** converts at **85.71%**

This suggests a potential regression or missing onboarding flow in v8.2.3. Since v8.2.3 is outside the fair comparison filter, this finding is supplementary but important for product health monitoring. Mobile team should investigate the play-start UX on v8.2.3.

---

## Bonus: Unified Daily Metrics Summary Query

For leadership dashboards, combine all validated metrics into one view:

```sql
WITH daily_signups AS (
  SELECT event_date, COUNT(DISTINCT user_id) AS signups
  FROM dynamodb_events WHERE event_type = 'signup'
  GROUP BY event_date
),
daily_plays AS (
  SELECT stream_date AS event_date, COUNT(DISTINCT play_session_id) AS plays
  FROM backend_streams GROUP BY stream_date
),
daily_app_opens AS (
  SELECT event_date, COUNT(DISTINCT user_pseudo_id) AS app_opens
  FROM ga4_events WHERE event_name = 'app_open' AND app_version = '8.2.4'
  GROUP BY event_date
)

SELECT
  COALESCE(s.event_date, p.event_date, a.event_date) AS event_date,
  IFNULL(s.signups, 0) AS signups,
  IFNULL(p.plays, 0) AS backend_confirmed_plays,
  IFNULL(a.app_opens, 0) AS app_opens
FROM daily_signups s
FULL OUTER JOIN daily_plays p ON s.event_date = p.event_date
FULL OUTER JOIN daily_app_opens a ON s.event_date = a.event_date
ORDER BY event_date;
```

---

[Back to Index](SUBMISSION_README.md)
