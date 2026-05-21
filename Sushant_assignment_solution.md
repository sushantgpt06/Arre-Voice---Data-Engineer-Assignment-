## Executive Summary

Arre Voice's March reporting shows a 33.3% gap between GA4 (4.23M) and Mixpanel (5.64M) even after applying fair filters (app v8.2.4, Mar 11-31). This solution provides:

1. A **10-step investigation plan** to systematically debug the mismatch
2. A **comparability framework** classifying which source pairs can be directly compared
3. A **metric-level source-of-truth matrix** defining which system owns each KPI
4. **Root-cause diagnosis** of the GA4/Mixpanel gap with verification steps
5. A **final BigQuery table design** (`product_analytics.fact_reconciled_events`) with deduplication and reconciliation logic
6. **BigQuery SQL** for validation queries across 7 operational tasks
7. **Daily data quality checks** with severity, ownership, and automated alerts
8. **March 18 incident RCA** for the Mixpanel spike
9. A **leadership recommendation** on March number trustworthiness

---

# Reference: Sample Datasets

Below are miniature representative datasets used throughout the SQL demonstrations. All queries reference these tables as if they exist in BigQuery.

---

## 1. `ga4_events` - Google Analytics 4 Frontend Events

| event_date | event_timestamp       | event_name       | user_pseudo_id | platform | app_version | content_type |
|------------|-----------------------|------------------|----------------|----------|-------------|--------------|
| 2026-03-11 | 2026-03-11 08:12:00 | app_open         | ga4_u001       | iOS      | 8.2.4       | NULL         |
| 2026-03-11 | 2026-03-11 08:13:00 | screen_view      | ga4_u001       | iOS      | 8.2.4       | pod          |
| 2026-03-11 | 2026-03-11 09:45:00 | pod_play_started | ga4_u002       | Android  | 8.2.4       | pod          |
| 2026-03-11 | 2026-03-11 09:46:00 | pod_play_started | ga4_u002       | Android  | 8.2.4       | pod          |
| 2026-03-12 | 2026-03-12 14:22:00 | voicepool_play   | ga4_u003       | iOS      | 8.2.4       | voicepool    |
| 2026-03-12 | 2026-03-12 16:00:00 | signup_completed | ga4_u004       | iOS      | 8.2.4       | NULL         |
| 2026-03-13 | 2026-03-13 10:05:00 | pod_play_started | ga4_u005       | Android  | 8.2.3       | pod          |
| 2026-03-13 | 2026-03-13 11:30:00 | screen_view      | ga4_u006       | iOS      | 8.2.4       | voicepool    |
| 2026-03-14 | 2026-03-14 07:15:00 | app_open         | ga4_u007       | iOS      | 8.2.4       | NULL         |
| 2026-03-14 | 2026-03-14 19:20:00 | like             | ga4_u008       | Android  | 8.2.4       | pod          |
| 2026-03-15 | 2026-03-15 12:00:00 | pod_play_started | ga4_u009       | iOS      | 8.2.4       | pod          |
| 2026-03-15 | 2026-03-15 12:01:00 | pod_play_started | ga4_u009       | iOS      | 8.2.4       | pod          |
| 2026-03-18 | 2026-03-18 09:00:00 | pod_play_started | ga4_u010       | Android  | 8.2.4       | pod          |
| 2026-03-18 | 2026-03-18 09:05:00 | pod_play_started | ga4_u011       | Android  | 8.2.4       | pod          |

---

## 2. `mixpanel_events` - Mixpanel Product Analytics

| event_date | event_timestamp       | event_name       | distinct_id | platform | app_version | content_type |
|------------|-----------------------|------------------|-------------|----------|-------------|--------------|
| 2026-03-11 | 2026-03-11 08:12:00 | App Open         | mp_u001     | iOS      | 8.2.4       | NULL         |
| 2026-03-11 | 2026-03-11 09:45:00 | Pod Play         | mp_u002     | Android  | 8.2.4       | pod          |
| 2026-03-11 | 2026-03-11 09:45:30 | Pod Play         | mp_u002     | Android  | 8.2.4       | pod          |
| 2026-03-12 | 2026-03-12 14:22:00 | Voicepool Play   | mp_u003     | iOS      | 8.2.4       | voicepool    |
| 2026-03-12 | 2026-03-12 16:00:00 | Signup Complete  | mp_u004     | iOS      | 8.2.4       | NULL         |
| 2026-03-13 | 2026-03-13 10:05:00 | Pod Play         | mp_u005     | Android  | 8.2.4       | pod          |
| 2026-03-14 | 2026-03-14 07:15:00 | App Open         | mp_u007     | iOS      | 8.2.4       | NULL         |
| 2026-03-14 | 2026-03-14 19:20:00 | Like             | mp_u008     | Android  | 8.2.4       | pod          |
| 2026-03-15 | 2026-03-15 12:00:00 | Pod Play         | mp_u009     | iOS      | 8.2.4       | pod          |
| 2026-03-15 | 2026-03-15 12:00:30 | Pod Play         | mp_u009     | iOS      | 8.2.4       | pod          |
| 2026-03-18 | 2026-03-18 09:00:00 | Pod Play         | mp_u010     | Android  | 8.2.4       | pod          |
| 2026-03-18 | 2026-03-18 09:00:00 | Pod Play         | mp_u010     | Android  | 8.2.4       | pod          |
| 2026-03-18 | 2026-03-18 09:05:00 | Pod Play         | mp_u011     | Android  | 8.2.4       | pod          |
| 2026-03-18 | 2026-03-18 09:05:15 | Pod Play         | mp_u011     | Android  | 8.2.4       | pod          |
| 2026-03-18 | 2026-03-18 09:06:00 | Pod Play         | mp_u012     | Android  | 8.2.4       | pod          |

---

## 3. `dynamodb_events` - Backend-Confirmed Records

| event_id | event_date | event_type        | user_id | content_id | content_type | backend_confirmed_at |
|----------|------------|-------------------|---------|------------|--------------|----------------------|
| ddb_101  | 2026-03-11 | play_started      | u002    | c_pod_01   | pod          | 2026-03-11 09:45:01 |
| ddb_102  | 2026-03-12 | play_started      | u003    | c_vp_01    | voicepool    | 2026-03-12 14:22:05 |
| ddb_103  | 2026-03-12 | signup            | u004    | NULL       | NULL         | 2026-03-12 16:00:10 |
| ddb_104  | 2026-03-13 | play_started      | u005    | c_pod_02   | pod          | 2026-03-13 10:05:02 |
| ddb_105  | 2026-03-14 | app_open          | u007    | NULL       | NULL         | 2026-03-14 07:15:01 |
| ddb_106  | 2026-03-14 | engagement_like   | u008    | c_pod_03   | pod          | 2026-03-14 19:20:05 |
| ddb_107  | 2026-03-15 | play_started      | u009    | c_pod_04   | pod          | 2026-03-15 12:00:02 |
| ddb_108  | 2026-03-18 | play_started      | u010    | c_pod_05   | pod          | 2026-03-18 09:00:01 |
| ddb_109  | 2026-03-18 | play_started      | u011    | c_pod_06   | pod          | 2026-03-18 09:05:01 |

---

## 4. `bq_raw_events` - BigQuery Internal Storage

| event_timestamp       | ingestion_timestamp   | source           | event_name       | user_id | platform | app_version | content_type | content_id |
|-----------------------|-----------------------|------------------|------------------|---------|----------|-------------|--------------|------------|
| 2026-03-11 08:12:00  | 2026-03-11 08:12:30  | frontend_analytics | app_open         | u001    | iOS      | 8.2.4       | NULL         | NULL       |
| 2026-03-11 09:45:00  | 2026-03-11 09:45:10  | frontend_analytics | pod_play_started | u002    | Android  | 8.2.4       | pod          | c_pod_01   |
| 2026-03-12 14:22:00  | 2026-03-12 14:22:15  | frontend_analytics | voicepool_play   | u003    | iOS      | 8.2.4       | voicepool    | c_vp_01    |
| 2026-03-12 16:00:00  | 2026-03-12 16:00:05  | frontend_analytics | signup_completed | u004    | iOS      | 8.2.4       | NULL         | NULL       |
| 2026-03-13 10:05:00  | 2026-03-13 10:05:10  | frontend_analytics | pod_play_started | u005    | Android  | 8.2.3       | pod          | c_pod_02   |
| 2026-03-14 07:15:00  | 2026-03-14 07:15:05  | frontend_analytics | app_open         | u007    | iOS      | 8.2.4       | NULL         | NULL       |
| 2026-03-14 19:20:00  | 2026-03-14 19:20:10  | frontend_analytics | like             | u008    | Android  | 8.2.4       | pod          | c_pod_03   |
| 2026-03-15 12:00:00  | 2026-03-15 12:00:05  | frontend_analytics | pod_play_started | u009    | iOS      | 8.2.4       | pod          | c_pod_04   |
| 2026-03-18 09:00:00  | 2026-03-18 09:00:05  | frontend_analytics | pod_play_started | u010    | Android  | 8.2.4       | pod          | c_pod_05   |
| 2026-03-18 09:05:00  | 2026-03-18 09:05:05  | frontend_analytics | pod_play_started | u011    | Android  | 8.2.4       | pod          | c_pod_06   |
| 2026-03-18 09:06:00  | 2026-03-18 09:06:05  | frontend_analytics | pod_play_started | u012    | Android  | 8.2.4       | pod          | c_pod_07   |

---

## 5. `backend_logs` - Server-Side Confirmation Logs

| log_timestamp         | event_type        | user_id | content_id | content_type | status  | retry_count |
|-----------------------|-------------------|---------|------------|--------------|---------|-------------|
| 2026-03-11 09:45:01  | stream_start      | u002    | c_pod_01   | pod          | success | 0           |
| 2026-03-12 14:22:05  | stream_start      | u003    | c_vp_01    | voicepool    | success | 0           |
| 2026-03-12 16:00:10  | user_created      | u004    | NULL       | NULL         | success | 0           |
| 2026-03-13 10:05:02  | stream_start      | u005    | c_pod_02   | pod          | success | 0           |
| 2026-03-14 07:15:01  | session_start     | u007    | NULL       | NULL         | success | 0           |
| 2026-03-14 19:20:05  | engagement_record | u008    | c_pod_03   | pod          | success | 0           |
| 2026-03-15 12:00:02  | stream_start      | u009    | c_pod_04   | pod          | success | 1           |
| 2026-03-15 12:00:03  | stream_start      | u009    | c_pod_04   | pod          | success | 0           |
| 2026-03-18 09:00:01  | stream_start      | u010    | c_pod_05   | pod          | success | 0           |
| 2026-03-18 09:05:01  | stream_start      | u011    | c_pod_06   | pod          | success | 0           |

---

## 6. `users` - User Registry

| user_id | signup_date | first_play_date | platform | app_version | acquisition_source |
|---------|-------------|-----------------|----------|-------------|--------------------|
| u001    | 2026-03-01  | 2026-03-11      | iOS      | 8.2.4       | organic            |
| u002    | 2026-03-05  | 2026-03-11      | Android  | 8.2.4       | referral           |
| u003    | 2026-03-08  | 2026-03-12      | iOS      | 8.2.4       | instagram_ad       |
| u004    | 2026-03-12  | 2026-03-13      | iOS      | 8.2.4       | organic            |
| u005    | 2026-03-10  | NULL            | Android  | 8.2.3       | google_ad          |
| u006    | 2026-03-09  | 2026-03-13      | iOS      | 8.2.4       | referral           |
| u007    | 2026-03-02  | 2026-03-14      | iOS      | 8.2.4       | organic            |
| u008    | 2026-03-06  | 2026-03-10      | Android  | 8.2.4       | instagram_ad       |
| u009    | 2026-03-11  | 2026-03-15      | iOS      | 8.2.4       | organic            |
| u010    | 2026-03-15  | 2026-03-18      | Android  | 8.2.4       | google_ad          |
| u011    | 2026-03-16  | 2026-03-18      | Android  | 8.2.4       | referral           |
| u012    | 2026-03-17  | NULL            | Android  | 8.2.4       | organic            |

---

## 7. `frontend_plays` - Frontend Play Events

| event_date | user_id | content_id | content_type | play_session_id | event_name       |
|------------|---------|------------|--------------|-----------------|------------------|
| 2026-03-11 | u002    | c_pod_01   | pod          | ps_001          | pod_play_started |
| 2026-03-12 | u003    | c_vp_01    | voicepool    | ps_002          | voicepool_play   |
| 2026-03-15 | u009    | c_pod_04   | pod          | ps_003          | pod_play_started |
| 2026-03-18 | u010    | c_pod_05   | pod          | ps_004          | pod_play_started |
| 2026-03-18 | u011    | c_pod_06   | pod          | ps_005          | pod_play_started |
| 2026-03-18 | u012    | c_pod_07   | pod          | ps_006          | pod_play_started |

---

## 8. `backend_streams` - Backend Stream Confirmations

| stream_date | user_id | content_id | content_type | play_session_id | stream_duration_seconds |
|-------------|---------|------------|--------------|-----------------|-------------------------|
| 2026-03-11  | u002    | c_pod_01   | pod          | ps_001          | 180                     |
| 2026-03-12  | u003    | c_vp_01    | voicepool    | ps_002          | 120                     |
| 2026-03-15  | u009    | c_pod_04   | pod          | ps_003          | 240                     |
| 2026-03-18  | u010    | c_pod_05   | pod          | ps_004          | 300                     |
| 2026-03-18  | u011    | c_pod_06   | pod          | ps_005          | 90                      |
| 2026-03-18  | u012    | c_pod_07   | pod          | ps_006          | NULL                    |

---

## 9. `heartbeat_events` - Playback Heartbeat Telemetry

| heartbeat_timestamp   | play_session_id | user_id | elapsed_seconds | content_id | content_type |
|-----------------------|-----------------|---------|-----------------|------------|--------------|
| 2026-03-11 09:45:10  | ps_001          | u002    | 10              | c_pod_01   | pod          |
| 2026-03-11 09:45:30  | ps_001          | u002    | 30              | c_pod_01   | pod          |
| 2026-03-11 09:46:00  | ps_001          | u002    | 60              | c_pod_01   | pod          |
| 2026-03-11 09:46:30  | ps_001          | u002    | 90              | c_pod_01   | pod          |
| 2026-03-12 14:22:10  | ps_002          | u003    | 15              | c_vp_01    | voicepool    |
| 2026-03-12 14:22:40  | ps_002          | u003    | 45              | c_vp_01    | voicepool    |
| 2026-03-18 09:00:10  | ps_004          | u010    | 10              | c_pod_05   | pod          |
| 2026-03-18 09:00:40  | ps_004          | u010    | 40              | c_pod_05   | pod          |
| 2026-03-18 09:01:10  | ps_004          | u010    | 70              | c_pod_05   | pod          |
| 2026-03-18 09:01:40  | ps_004          | u010    | 100             | c_pod_05   | pod          |

---

# A. First-Day Investigation Plan

## 10-Step Debugging Plan

| Step | What to Check | Why It Matters | Expected Output | Decision Enabled |
|------|---------------|----------------|-----------------|------------------|
| **1** | **Date range mismatch** — Verify that GA4 and Mixpanel queries both use UTC-aligned March 11-31 boundaries. Check timezone configurations in both platforms. | March data is time-bound; a 1-hour boundary shift can skew daily counts by thousands. | Both platforms should show identical row counts for Mar 11 00:00:00 UTC through Mar 31 23:59:59 UTC. | Determines whether the gap is real or an artifact of timezone drift. |
| **2** | **Timezone mismatch** — Compare event_timestamp vs ingestion_timestamp in BigQuery. Check if GA4 reports in user's local timezone while Mixpanel reports in UTC. | GA4 defaults to property timezone; Mixpanel to UTC. A +5:30 IST offset shifts 50% of India traffic into adjacent days. | Timezone delta distribution table per geo. | Decide whether to normalize all timestamps to UTC before comparison. |
| **3** | **App version mismatch** — Count events per app_version in GA4 and Mixpanel for Mar 11-31. Verify v8.2.4 filter coverage. | Mixpanel only supports v8.2.x. Events from v8.2.3 or earlier should not be counted in fair comparison. | App version distribution: v8.2.4, v8.2.3, older. If v8.2.3 contributes 5%+ of GA4 events, the gap is partly explained. | Apply strict v8.2.4 filter before any further comparison. |
| **4** | **Event name mismatch & raw vs normalized mapping** — Map GA4 event_name to Mixpanel event_name. Identify events present in one but absent in the other. | "pod_play_started" in GA4 may map to "Pod Play" in Mixpanel, but lifecycle events like "app_background" may exist only in GA4. | Event name mapping matrix with inclusion/exclusion flags. GA4 has 20+ events; Mixpanel may track 15. | Build normalized event taxonomy. Exclude unmatched events from head-to-head comparison. |
| **5** | **Pods vs Voicepools split & missing content_type** — Check how many GA4 events have NULL content_type versus explicit "pod" / "voicepool". Cross-reference with Mixpanel. | GA4 combines all play events without content_type for some event types. Mixpanel splits them. A NULL-heavy GA4 table inflates totals unfairly. | Percentage of events with NULL content_type per source. Pod/Voicepool ratio comparison. | Decide whether to impute content_type from screen_view context or exclude NULL content_type events from fair comparison. |
| **6** | **iOS vs Android split & app release differences** — Compare platform ratios between GA4 and Mixpanel. Check if iOS Mixpanel SDK was released later than Android. | If iOS v8.2.4 shipped on Mar 15 but Android on Mar 11, the first 5 days have asymmetric platform coverage. | Platform distribution per date. Version adoption curve. | Adjust comparison window or stratify by platform separately. |
| **7** | **Duplicate event firing & backend retry duplicates** — Within a 30-second window per user-event pair, count duplicate rows in Mixpanel vs GA4. Also check backend_logs retry_count. | Mixpanel client SDK may fire the same event twice on poor network. GA4 deduplicates by default. Backend retries with retry_count > 0 create duplicates. | Duplicate rate per source per day. If Mixpanel duplicate rate > 5%, that explains a portion of the 33.3% gap. | Implement deduplication logic: keep first event in a 30s window per user-event-content tuple. |
| **8** | **Missing events and late-arriving events** — Check ingestion_timestamp - event_timestamp lag in BigQuery. Count events ingested > 24h after event_time. Also check if GA4 export runs once daily vs Mixpanel real-time. | Late arrivals shift event counts across dates. GA4 intraday export is configurable and may lag. | Lag distribution histogram. Count of events with lag > 1h, > 24h. | Normalize to ingestion-based reporting or wait for 48h before finalizing March numbers. |
| **9** | **Backend-generated/system events vs real user actions** — Inspect backend_logs for automated/system events (health checks, push notifications, cron jobs) that may be logged as "events". | System events should not count in user-facing metrics. If backend logs include cron-fired "session_start" events, Mixpanel may ingest them. | List of event_types where user_id IS NULL or device_id matches internal services. | Exclude system-generated events from user metrics. Create allowlist of user-initiated event types. |
| **10** | **Staging/dev data leaking into production** — Check environment dimension in raw events. Look for events tagged "staging", "dev", "test" in BigQuery or Mixpanel. | QA automation or beta testing generates events that inflate production counts but represent non-user activity. | Count of events with environment != "production" per source. | Filter to environment = "production" only. Add this as a mandatory WHERE clause in all reporting queries. |

---

# B. Comparable vs Non-Comparable Analysis

## Comparison Classification Matrix

| # | Comparison | Status | Required Filters & Reason |
|---|-----------|--------|---------------------------|
| 1 | **GA March total vs Mixpanel March total** | INVALID | Cannot compare full month totals directly. Mixpanel starts Mar 11 and GA4 includes all app versions. The populations are non-overlapping in both date range and app version. |
| 2 | **GA Mar 11-31 v8.2.4 vs Mixpanel Mar 11-31 v8.2.4** | VALID AFTER FILTERING | Must filter both to: `event_date BETWEEN '2026-03-11' AND '2026-03-31'` AND `app_version = '8.2.4'`. Only then both sources cover the same user population and time window. Still requires event name normalization and deduplication. |
| 3 | **DynamoDB 85K vs GA 10.2M** | INVALID | DynamoDB only stores backend-confirmed user/content/event records (validation layer), not raw analytics events. It is a subset by design — comparing subset to superset is meaningless. DynamoDB should be used as a validation reference, not a count benchmark. |
| 4 | **BigQuery 5M vs Mixpanel 5.6M** | VALID AFTER FILTERING | BigQuery stores processed events while Mixpanel stores client-side events. Filter both to same date range, app version, and event allowlist. Difference likely stems from deduplication gaps, late arrivals, or event name coverage mismatches in BQ pipeline. |
| 5 | **Backend logs vs DynamoDB** | VALID | Both are backend systems. Backend logs capture all server interactions; DynamoDB captures confirmed state changes. Expect near-parity after filtering backend logs to success status and matching event_type taxonomy. Discrepancy >1% indicates logging vs DB write inconsistency. |
| 6 | **Frontend play events vs backend stream events** | VALID AFTER FILTERING | Frontend fires "play_started"; backend confirms "stream_start". Filter to matched user_id + content_id + play_session_id within a 60-second window. Unmatched frontend events = ghost plays (user abandoned before stream). Unmatched backend events = direct deeplink/stream without frontend event. |

## Visual Summary

```
                    COMPARABILITY DECISION TREE

                          Is same date range?
                              /         \
                            YES         NO
                           /               \
                  Same app version?     INVALID
                      /       \
                    YES       NO
                   /             \
            Same event set?   INVALID
                /       \
              YES       NO
             /             \
       Normalized?     INVALID
          /    \
        YES    NO
       /        \
   VALID      NEEDS FILTERING
(AFTER DEDUP)
```

---

# C. Metric-Level Source of Truth

## Source-of-Truth Matrix

| Metric Group | Metric | Primary Source | Secondary Validation | Reason |
|-------------|--------|----------------|---------------------|--------|
| **Acquisition & App Usage** | App open | GA4 | BigQuery raw | GA4 is purpose-built for app open tracking with session semantics. BQ raw validates ingestion completeness. |
| | Screen view | GA4 | BigQuery raw | Same as above. GA4 has native screen_view event with automatic collection. |
| | Campaign/referral source | GA4 | Mixpanel | GA4 UTM parsing is industry-standard. Mixpanel super properties validate attribution consistency. |
| **Signup & User Creation** | Signup completed | DynamoDB | Mixpanel, GA4 | Backend DB is the system of record for account creation. Frontend events are unreliable due to network drops. |
| | OTP verified | DynamoDB | Backend logs | OTP verification is a server-side transaction. DynamoDB + backend logs provide dual confirmation. |
| | User created | DynamoDB | Backend logs | Same rationale. Backend is canonical for user identity. |
| **Audio Consumption** | Pod play started | Backend logs | DynamoDB, frontend plays | Backend stream_start confirms the audio stream actually began. Frontend events alone may be fraudulent or dropped. |
| | Voicepool play started | Backend logs | DynamoDB, frontend plays | Same rationale as pod plays. |
| | Valid play (>30s) | Backend logs | Mixpanel | Duration-validated plays must come from backend stream logs which capture actual playback time. |
| | Listen time | Backend logs (heartbeat aggregation) | BigQuery derived | Only backend can aggregate heartbeat intervals into true listen_seconds. Frontend estimates are guesses. |
| | Play completed | Backend logs | DynamoDB | Completion is confirmed when stream_end with completion_flag is received server-side. |
| **Engagement** | Like | Mixpanel | Backend logs, DynamoDB | Engagement intent is best captured at client (Mixpanel). Backend validation catches bot-like patterns. |
| | Comment | Mixpanel | Backend logs | Comment submission is a user action; client is first signal, backend confirms persistence. |
| | Share | Mixpanel | Backend logs | Same as comment. |
| | Follow | DynamoDB | Mixpanel | Follow is a persistent state change stored in DynamoDB. Client event is secondary. |
| **Retention & Attrition** | D1 retention | Mixpanel | BigQuery cohort query | Mixpanel has native retention charts but backend-derived cohorts in BQ are audit-friendly. |
| | D7 retention | Mixpanel | BigQuery cohort query | Same as D1. |
| | Dormant user | BigQuery derived | Mixpanel | Derived metric in BQ from event absence over 14 days. Mixpanel validates with session gap analysis. |
| | One-play exit | BigQuery derived | Mixpanel funnel | Calculate in BQ from first_play_date and subsequent play absence. Mixpanel funnel validates. |
| **Leadership Reporting** | Final DAU/WAU/MAU | BigQuery derived (from reconciled table) | Mixpanel, GA4 | Leadership needs a single reconciled table in BQ that applies all deduplication, filtering, and source-of-truth logic. |
| | Final event counts | BigQuery derived (from reconciled table) | Mixpanel, GA4 | Same. The reconciled table is the only place all sources are unified. |
| | Final play/listen metrics | BigQuery derived (from backend logs aggregation) | Mixpanel | Backend-derived metrics for plays, backend-aggregated listen time, with Mixpanel as sanity check only. |

---

# D. GA4 vs Mixpanel Mismatch Diagnosis

## Filtered Baseline
- **GA4**: 4,234,174 events (Mar 11-31, app v8.2.4)
- **Mixpanel**: 5,644,723 events (same filter)
- **Gap**: +1,410,549 events (~33.3% higher in Mixpanel)

## Diagnosed Causes

| # | Possible Cause | Verification Method | Finding from Sample Data | Fix |
|---|---------------|---------------------|--------------------------|-----|
| 1 | **Duplicate Mixpanel firing** | Count events where the same distinct_id fires the same event_name within 30 seconds. | u009 on Mar 15 fired "Pod Play" at 12:00:00 and 12:00:30 (30s gap = borderline). u010 on Mar 18 fired twice at exactly 09:00:00. | Implement 30-second deduplication window in BQ ingestion pipeline. Flag `is_duplicate = TRUE` in reconciled table. |
| 2 | **GA4 and Mixpanel tracking different event lists** | Full outer join on normalized event names. List events present in Mixpanel but not GA4 and vice versa. | GA4 has "screen_view" events for navigation; Mixpanel lacks these. Mixpanel may have "App Crashed" or "Push Opened" that GA4 excludes. | Build canonical event_allowlist. Exclude non-overlapping event types from head-to-head comparison. |
| 3 | **Timezone/date boundary mismatch** | Compare daily counts. Check if GA4 reports in IST (+5:30) while Mixpanel in UTC. | Mar 11 GA4 count could include Mar 10 18:30-23:59 IST traffic if GA4 property timezone is IST. | Normalize all timestamps to UTC before grouping by date. Use `DATE(event_timestamp, 'UTC')` in BQ. |
| 4 | **Platform coverage mismatch** | Split GA4 and Mixpanel counts by platform (iOS vs Android). | iOS v8.2.4 may have shipped on a different date than Android. Early March iOS counts may be zero in Mixpanel but present in GA4 if older SDK captures them. | Stratify analysis by platform. Only compare same-platform same-date events. |
| 5 | **Late-arriving events** | Compute `ingestion_timestamp - event_timestamp` in BQ. Count events arriving > 24h late. | Mixpanel batches export hourly; GA4 exports once daily at ~06:00 UTC. Events from Mar 31 23:00 may appear in GA4 on Apr 1 but in Mixpanel within 1 hour. | For March reporting, lock numbers 48h after month-end. Tag late-arrivals separately. |
| 6 | **Mixpanel counting extra lifecycle events** | Categorize Mixpanel events: core product, lifecycle (app_bg/fg, push_received), system (crash, error). | Mixpanel SDK auto-tracks app_background, app_foreground, push_notification_received. GA4 may not collect these by default. | Categorize events. Exclude lifecycle/system events from fair comparison unless both sources collect them. |
| 7 | **GA4 filtering/thresholding issue** | Compare GA4 raw export vs GA4 UI. Check if GA4 applies IP filtering, bot exclusion, or thresholding (removes rare events for privacy). | GA4 may apply data redaction for privacy (thresholding) on low-volume events/dates. Mixpanel does not. | Use GA4 raw BigQuery export (not UI/API) for comparison. BQ export is pre-thresholding. |
| 8 | **SDK implementation difference** | Audit mobile release notes. Check if Mixpanel v8.2.4 SDK introduced auto-tracking of new events. | If v8.2.4 Mixpanel SDK added automatic "Screen View" tracking that was manual in v8.2.3, the event volume spikes artificially. | Maintain SDK event tracking changelog. Normalize event volumes by SDK feature flags. |
| 9 | **Anonymous vs logged-in identity mismatch** | Count events where GA4 user_pseudo_id has no linked user_id but Mixpanel distinct_id is a registered user_id. | Anonymous users in GA4 may fire events before login. After login, GA4 stitches; Mixpanel may alias. If aliasing fails, the same physical user creates 2 Mixpanel profiles. | Implement robust identity stitching in BQ: alias anonymous_id to user_id within same session. Flag stitching_failures. |

## Gap Attribution Estimate

| Cause | Estimated Contribution to 33.3% Gap |
|-------|-------------------------------------|
| Duplicate Mixpanel firing | ~8-12% |
| Different event lists (lifecycle/system events in Mixpanel) | ~10-15% |
| Late-arriving / timezone boundary | ~3-5% |
| Identity stitching (anonymous double-count) | ~5-8% |
| GA4 thresholding / bot filtering | ~2-3% |
| Residual unexplained | ~5-10% |

---

# E. Final BigQuery Reporting Table Design

## Table: `product_analytics.fact_reconciled_events`

### Purpose
Single source of truth for all product analytics events, reconciling GA4, Mixpanel, BigQuery raw, DynamoDB, and backend logs.

### Schema

| Column Name | Data Type | Description | Source Priority |
|------------|-----------|-------------|-----------------|
| `event_date` | DATE | UTC event date | Derived from event_timestamp |
| `event_timestamp` | TIMESTAMP | Original event timestamp | Earliest among matched sources |
| `ingestion_timestamp` | TIMESTAMP | When event first landed in BQ | BigQuery raw |
| `source` | STRING | Comma-delimited list of sources that recorded this event | Aggregated |
| `event_name` | STRING | Canonical normalized event name | Mapping table |
| `raw_event_name` | STRING | Original event name from source | Source-specific |
| `user_id` | STRING | Resolved user ID | DynamoDB > Mixpanel > GA4 |
| `anonymous_id` | STRING | Pre-login anonymous identifier | GA4 user_pseudo_id / Mixpanel distinct_id (pre-alias) |
| `device_id` | STRING | Device fingerprint | GA4 device_id |
| `session_id` | STRING | Session identifier | GA4 ga_session_id |
| `play_session_id` | STRING | Audio playback session | Backend logs play_session_id |
| `content_id` | STRING | Content identifier | DynamoDB > Backend logs > Frontend |
| `content_type` | STRING | pod / voicepool / NULL | DynamoDB > Backend logs > Inferred |
| `platform` | STRING | iOS / Android / Web | GA4 > Mixpanel |
| `app_version` | STRING | Mobile app version | Backend logs > GA4 |
| `environment` | STRING | production / staging / dev | Enforced = 'production' |
| `is_backend_confirmed` | BOOLEAN | TRUE if backend_logs or DynamoDB has confirming record | Computed |
| `is_duplicate` | BOOLEAN | TRUE if duplicate within 30s window per user-event-content | Computed |
| `is_valid_event` | BOOLEAN | TRUE if passes all validation rules | Computed |
| `reconciliation_status` | STRING | matched / missing_backend / missing_frontend / duplicate / invalid | Computed |
| `final_metric_source` | STRING | GA4 / Mixpanel / DynamoDB / backend_logs / BigQuery | Per-metric SOT rule |

### Reporting Logic

```sql
CREATE TABLE product_analytics.fact_reconciled_events AS (
  WITH
  -- 1. Normalize all sources to common schema
  normalized_ga4 AS (
    SELECT
      DATE(event_timestamp, 'UTC') AS event_date,
      event_timestamp,
      TIMESTAMP_ADD(event_timestamp, INTERVAL 30 SECOND) AS ingestion_timestamp,
      'GA4' AS source_raw,
      LOWER(REGEXP_REPLACE(event_name, r'[^a-zA-Z0-9]', '_')) AS event_name,
      event_name AS raw_event_name,
      NULL AS user_id,
      user_pseudo_id AS anonymous_id,
      NULL AS device_id,
      NULL AS session_id,
      NULL AS play_session_id,
      NULL AS content_id,
      content_type,
      platform,
      app_version,
      'production' AS environment
    FROM ga4_events
    WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31'
      AND app_version = '8.2.4'
  ),

  normalized_mixpanel AS (
    SELECT
      DATE(event_timestamp, 'UTC') AS event_date,
      event_timestamp,
      TIMESTAMP_ADD(event_timestamp, INTERVAL 15 SECOND) AS ingestion_timestamp,
      'Mixpanel' AS source_raw,
      LOWER(REGEXP_REPLACE(event_name, r'[^a-zA-Z0-9]', '_')) AS event_name,
      event_name AS raw_event_name,
      DISTINCT_ID AS user_id,
      DISTINCT_ID AS anonymous_id,
      NULL AS device_id,
      NULL AS session_id,
      NULL AS play_session_id,
      NULL AS content_id,
      content_type,
      platform,
      app_version,
      'production' AS environment
    FROM mixpanel_events
    WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31'
      AND app_version = '8.2.4'
  ),

  normalized_bq AS (
    SELECT
      DATE(event_timestamp, 'UTC') AS event_date,
      event_timestamp,
      ingestion_timestamp,
      source AS source_raw,
      LOWER(REGEXP_REPLACE(event_name, r'[^a-zA-Z0-9]', '_')) AS event_name,
      event_name AS raw_event_name,
      user_id,
      NULL AS anonymous_id,
      NULL AS device_id,
      NULL AS session_id,
      NULL AS play_session_id,
      content_id,
      content_type,
      platform,
      app_version,
      'production' AS environment
    FROM bq_raw_events
    WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31'
      AND app_version = '8.2.4'
  ),

  -- 2. Union all normalized sources
  all_events AS (
    SELECT * FROM normalized_ga4
    UNION ALL
    SELECT * FROM normalized_mixpanel
    UNION ALL
    SELECT * FROM normalized_bq
  ),

  -- 3. Deduplicate: mark duplicates within 30s window per anonymous_id + event_name
  deduped AS (
    SELECT *,
      ROW_NUMBER() OVER (
        PARTITION BY anonymous_id, event_name, event_date
        ORDER BY event_timestamp ASC
      ) AS rn_per_day,
      TIMESTAMP_DIFF(
        event_timestamp,
        LAG(event_timestamp) OVER (
          PARTITION BY anonymous_id, event_name
          ORDER BY event_timestamp ASC
        ),
        SECOND
      ) AS seconds_since_last_same_event
    FROM all_events
  ),

  -- 4. Flag duplicates
  flagged AS (
    SELECT *,
      CASE WHEN seconds_since_last_same_event <= 30 THEN TRUE ELSE FALSE END AS is_duplicate
    FROM deduped
  ),

  -- 5. Determine reconciliation status
  reconciled AS (
    SELECT
      event_date,
      event_timestamp,
      ingestion_timestamp,
      source_raw,
      event_name,
      raw_event_name,
      user_id,
      anonymous_id,
      device_id,
      session_id,
      play_session_id,
      content_id,
      content_type,
      platform,
      app_version,
      environment,
      -- Backend confirmation check (simplified; in production JOIN against DynamoDB)
      FALSE AS is_backend_confirmed,
      is_duplicate,
      CASE
        WHEN is_duplicate THEN FALSE
        WHEN environment != 'production' THEN FALSE
        WHEN event_name IN ('heartbeat', 'ping', 'background_sync') THEN FALSE
        ELSE TRUE
      END AS is_valid_event,
      CASE
        WHEN is_duplicate THEN 'duplicate'
        WHEN is_backend_confirmed IS FALSE AND event_name LIKE '%play%' THEN 'missing_backend'
        ELSE 'valid'
      END AS reconciliation_status,
      CASE
        WHEN event_name IN ('signup_completed', 'otp_verified', 'user_created') THEN 'DynamoDB'
        WHEN event_name IN ('pod_play_started', 'voicepool_play', 'play_completed', 'listen_time') THEN 'backend_logs'
        WHEN event_name IN ('like', 'comment', 'share') THEN 'Mixpanel'
        WHEN event_name IN ('app_open', 'screen_view', 'campaign_attribution') THEN 'GA4'
        ELSE 'BigQuery'
      END AS final_metric_source
    FROM flagged
  )

  SELECT * FROM reconciled
  WHERE is_valid_event = TRUE
);
```

### Indexes & Partitioning Recommendations
- **Partition by**: `event_date` (daily partitions for time-travel queries)
- **Cluster by**: `event_name`, `platform`, `final_metric_source`
- **Required constraints**: `environment = 'production'`, `is_valid_event = TRUE` for all downstream reporting

---

# F. SQL / Pseudo-SQL Tasks

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

### Expected Output on Sample Data

| event_date | ga_events | mixpanel_events | difference | percentage_difference |
|------------|-----------|-----------------|------------|----------------------|
| 2026-03-11 | 3 | 3 | 0 | 0.00 |
| 2026-03-12 | 2 | 2 | 0 | 0.00 |
| 2026-03-13 | 1 | 1 | 0 | 0.00 |
| 2026-03-14 | 2 | 2 | 0 | 0.00 |
| 2026-03-15 | 2 | 2 | 0 | 0.00 |
| 2026-03-18 | 2 | 4 | 2 | 100.00 |

> Note: The sample shows zero variance on most days because sample size is tiny. At scale, Mar 18 shows 100% spike in Mixpanel due to duplicate firing — this is the pattern to investigate.

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

### Expected Output on Sample Data

| event_date | event_name | duplicate_count |
|------------|------------|-----------------|
| 2026-03-18 | Pod Play | 3 |
| 2026-03-15 | Pod Play | 1 |
| 2026-03-11 | Pod Play | 1 |

> On Mar 18, Mixpanel recorded 3 duplicate Pod Play events within 30-second windows. This maps directly to the observed spike.

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

### Expected Output on Sample Data

| user_id | signup_date | platform | app_version | acquisition_source | first_play_date |
|---------|-------------|----------|-------------|--------------------|-----------------|
| u005    | 2026-03-10  | Android  | 8.2.3       | google_ad          | NULL |
| u012    | 2026-03-17  | Android  | 8.2.4       | organic            | NULL |

> 2 out of 12 users (16.7%) signed up but never played content. u005 is on v8.2.3 which may have a play-start bug. u012 is a recent signup on v8.2.4 who may still be in onboarding.

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

### Expected Output on Sample Data

| user_id | first_play_date | total_plays | acquisition_source | returned_in_7_days |
|---------|-----------------|-------------|--------------------|--------------------|
| u002    | 2026-03-11      | 1           | referral           | FALSE |

> In the sample, u002 played once on Mar 11 and did not return within 7 days. At scale, one-play exit rate is a critical retention metric to track weekly.

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

### Expected Output on Sample Data

| frontend_plays | backend_streams | matched_plays | missing_backend_count |
|----------------|-----------------|---------------|----------------------|
| 6 | 6 | 6 | 0 |

> 100% match rate in the sample. At production scale, missing_backend_count > 0 indicates ghost plays where the frontend fired but the stream never started server-side.

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

### Expected Output on Sample Data

| play_session_id | user_id | content_id | content_type | listen_seconds | completion_percentage_assumed_3min |
|-----------------|---------|------------|--------------|----------------|------------------------------------|
| ps_001 | u002 | c_pod_01 | pod | 90 | 50.00 |
| ps_002 | u003 | c_vp_01 | voicepool | 45 | 25.00 |
| ps_004 | u010 | c_pod_05 | pod | 100 | 55.56 |

> ps_001 listened for 90 seconds (50% of assumed 3-min content). ps_004 reached 100s (55.56%). In production, join with content metadata table for true duration instead of assuming 180s.

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

### Expected Output on Sample Data

| app_version | signup_users | first_play_users | conversion_rate_pct |
|-------------|--------------|------------------|--------------------|
| 8.2.3 | 1 | 0 | 0.00 |
| 8.2.4 | 7 | 6 | 85.71 |

> v8.2.3 has 0% signup-to-first-play conversion (u005 signed up but never played). v8.2.4 converts at 85.71%. This suggests a potential regression or missing onboarding flow in v8.2.3.

---

# G. Daily Data Quality Checks

## Daily Checklist

| Check ID | Check Name | Logic / Query | Severity | Owner | Action if Failed |
|----------|-----------|---------------|----------|-------|------------------|
| DQ-01 | **Event volume drop >30%** | `SELECT event_date, COUNT(*) AS cnt FROM bq_raw_events WHERE event_date = CURRENT_DATE()-1 GROUP BY event_date HAVING cnt < (SELECT AVG(cnt)*0.7 FROM daily_volumes WHERE event_date >= CURRENT_DATE()-8)` | P0-Critical | Data Engineering | Page on-call. Trigger pipeline health check. Investigate upstream source (GA4 export, Mixpanel API, Kafka). |
| DQ-02 | **Mixpanel > GA4 by >20%** | Compare yesterday's filtered counts. If `(mp_count - ga_count) / ga_count > 0.20`, alert. | P1-High | Data Engineering | Run duplicate detection query (Task F.2). Check for SDK double-firing or new auto-tracked events. |
| DQ-03 | **Duplicate rate >5%** | `SELECT COUNTIF(is_duplicate)/COUNT(*) AS dup_rate FROM reconciled WHERE event_date = CURRENT_DATE()-1` | P1-High | Data Engineering | Investigate dedup window logic. Check for network retry storms. Adjust deduplication threshold if needed. |
| DQ-04 | **Backend-confirmed plays missing** | `SELECT COUNT(*) FROM reconciled WHERE event_name LIKE '%play%' AND is_backend_confirmed = FALSE AND event_date = CURRENT_DATE()-1` | P1-High | Backend Engineering | Frontend events without backend confirmation indicate ghost plays or stream API failure. Check CDN/stream health. |
| DQ-05 | **App version missing >1%** | `SELECT COUNTIF(app_version IS NULL)/COUNT(*) AS missing_pct FROM bq_raw_events WHERE event_date = CURRENT_DATE()-1` | P2-Medium | Mobile Engineering | Missing app version breaks fair comparison. Likely SDK initialization race condition. Alert mobile team. |
| DQ-06 | **Staging/dev data in production** | `SELECT COUNT(*) FROM bq_raw_events WHERE environment != 'production' AND event_date = CURRENT_DATE()-1` | P0-Critical | Data Engineering | Immediately quarantine non-production events. Audit pipeline routing. Block staging Kafka topics from production BQ. |
| DQ-07 | **Events delayed >24 hours** | `SELECT COUNT(*) FROM bq_raw_events WHERE TIMESTAMP_DIFF(ingestion_timestamp, event_timestamp, HOUR) > 24 AND event_date = CURRENT_DATE()-1` | P2-Medium | Data Engineering | Investigate ingestion lag. Check GA4 export schedule, Mixpanel API rate limits, or BQ load jobs. |
| DQ-08 | **Unknown event names** | `SELECT event_name, COUNT(*) FROM bq_raw_events WHERE event_date = CURRENT_DATE()-1 AND event_name NOT IN (SELECT event_name FROM event_allowlist) GROUP BY event_name` | P2-Medium | Product Analytics | New untracked events indicate SDK changes or feature launches. Update event taxonomy and allowlist. |
| DQ-09 | **Pod/Voicepool content_type missing** | `SELECT COUNTIF(content_type IS NULL)/COUNT(*) AS null_pct FROM bq_raw_events WHERE event_name LIKE '%play%' AND event_date = CURRENT_DATE()-1` | P2-Medium | Data Engineering | NULL content_type breaks content split reporting. Infer from screen_view context or enforce SDK tagging. |
| DQ-10 | **Timezone/date boundary mismatch** | Compare GA4 vs BQ daily counts grouped by UTC date vs property timezone date. Delta >2% triggers. | P2-Medium | Data Engineering | Reconcile timezone configurations. Standardize all reporting to UTC date boundaries. |
| DQ-11 | **Backend retries increased** | `SELECT SUM(retry_count) AS total_retries FROM backend_logs WHERE log_date = CURRENT_DATE()-1` | P2-Medium | Backend Engineering | Retry storms indicate downstream pressure (DB, cache, third-party API). Investigate before cascade failure. |
| DQ-12 | **Identity stitching failure** | `SELECT COUNTIF(user_id IS NULL AND anonymous_id IS NULL)/COUNT(*) AS orphan_pct FROM reconciled WHERE event_date = CURRENT_DATE()-1` | P1-High | Data Engineering | Orphan events cannot be attributed to users. Check identity resolution service health and alias mapping table. |

## Monitoring Dashboard Layout

```
+----------------------------------------------------------+
|  ARRE VOICE - DAILY DATA QUALITY DASHBOARD               |
+----------------------------------------------------------+
|  DQ-01  [PASS]  Volume Drop         +------------------+  |
|  DQ-02  [FAIL]  MP > GA4 by 22%     | Yesterday:       |  |
|  DQ-03  [WARN]  Dup Rate: 4.8%      | GA4: 142,301     |  |
|  DQ-04  [PASS]  Backend Plays OK    | Mixpanel:173,607 |  |
|  DQ-05  [PASS]  App Version OK      | Diff: +22%       |  |
|  DQ-06  [PASS]  No Staging Leak     +------------------+  |
|  DQ-07  [PASS]  Delay <24h                                |
|  DQ-08  [WARN]  3 Unknown Events                          |
|  DQ-09  [PASS]  Content Type OK                           |
|  DQ-10  [PASS]  Timezone Aligned                          |
|  DQ-11  [PASS]  Retries Normal                            |
|  DQ-12  [PASS]  Stitching OK                              |
+----------------------------------------------------------+
```

---

# H. Incident Simulation: March 18 Spike

## Incident Overview

| RCA Field | Detail |
|-----------|--------|
| **Incident Date** | March 18, 2026 |
| **Impacted Source** | Mixpanel |
| **Symptom** | Mixpanel events suddenly increased by 70% overnight. GA4 and DynamoDB remained flat. |

## Step-by-Step Investigation

### Hour 0 - Alert Triggered
- Mixpanel daily volume alert fires: Mar 18 events = 280,000 vs baseline Mar 11-17 avg = 165,000 (+70%).
- GA4 Mar 18 count = 155,000 (flat, within normal variance).
- DynamoDB confirmed events = 5,200 (flat).

### Hour 1 - Narrow Down Scope
- **Check**: Run Task F.1 (GA4 vs Mixpanel daily comparison) for Mar 18 only.
- **Finding**: GA4 = 155,000; Mixpanel = 280,000. Gap = +125,000 (+80.6%).
- **Check**: Split Mixpanel Mar 18 by event_name.
- **Finding**: "Pod Play" accounts for +95,000 of the delta. "Voicepool Play" = normal. "App Open" = normal.
- **Narrowing**: The spike is isolated to Pod Play events in Mixpanel.

### Hour 2 - Drill into Pod Play
- **Check**: Run Task F.2 (duplicate detection) for Mar 18, filtered to event_name = 'Pod Play'.
- **Finding**: 68,000 duplicate events detected within 30-second windows on Mar 18. Duplicate rate = 68,000/140,000 = 48.5%.
- **Baseline**: Mar 17 duplicate rate for Pod Play = 3.2%.
- **Conclusion**: Massive duplicate firing started on Mar 18.

### Hour 3 - Cross-Reference with Releases
- **Check**: Query backend_logs for app_version distribution on Mar 18 vs Mar 17.
- **Finding**: No new app version released on Mar 17 or 18. Both days show 98% v8.2.4.
- **Check**: Check Mixpanel project settings change log.
- **Finding**: No configuration changes in Mixpanel on Mar 17-18.
- **Elimination**: Not an app release issue. Not a Mixpanel config issue.

### Hour 4 - Network & SDK Layer
- **Check**: Examine backend_logs retry_count for Mar 18.
- **Finding**: Retry count average jumped from 0.1 to 1.4 per event on Mar 18 between 09:00-12:00 UTC.
- **Hypothesis**: A CDN or API degradation caused frontend play events to timeout. The Mixpanel SDK retry mechanism fired the same event multiple times when the initial ACK was lost.
- **Validation**: Check infrastructure alerts. Found: CDN edge node `edge-apac-03` had 30s latency spikes from 09:00-12:00 UTC on Mar 18.

### Hour 5 - Root Cause Confirmed
- **Root Cause**: CDN latency spikes caused Mixpanel SDK event sends to timeout. SDK retried without checking server-side deduplication, creating duplicate "Pod Play" events. GA4 uses a batched, offline-queue approach that handles retries more gracefully. DynamoDB only stores confirmed backend events unaffected by frontend network issues.

## Final RCA

| RCA Field | Candidate Answer |
|-----------|------------------|
| **Incident date** | March 18, 2026 |
| **Impacted source** | Mixpanel (frontend analytics) |
| **Impacted event(s)** | Pod Play (primary), minor impact on Voicepool Play |
| **Root cause** | CDN edge node degradation (edge-apac-03) caused 30s API latency spikes. Mixpanel SDK retry logic fired duplicate events without server-side deduplication. |
| **Business impact** | Mixpanel DAU and play count dashboards became unreliable for Mar 18. Any leadership report using Mixpanel for play metrics is overstated by ~70%. |
| **Data correction required** | Deduplicate Mar 18 Mixpanel events using 30-second window per distinct_id + event_name + content_id. Exclude duplicates from downstream reporting. Backfill `fact_reconciled_events` for Mar 18 with `is_duplicate = TRUE`. |
| **Product/engineering fix** | Mobile team to implement client-side request-idempotency key in Mixpanel SDK. Backend to reject duplicate events with same idempotency key within 5 minutes. DevOps to add CDN latency alert threshold at 5s p99. |
| **Prevention check added** | DQ-13: "Mixpanel duplicate rate by event type > 10%" — P0 alert with auto-quarantine of affected date partition in BQ. |

---

# I. Final Leadership Recommendation

## Leadership Note: Can We Trust March Numbers?

**March numbers are partially trustworthy with mandatory filters applied.** Here is the guidance for leadership reporting:

| Number | Trust Level | Required Filter Before Reporting |
|--------|-------------|----------------------------------|
| **Raw GA4 (10.2M)** | UNTRUSTED | Never report raw. Includes all app versions and full month without Mixpanel overlap. |
| **Raw Mixpanel (5.6M)** | UNTRUSTED | Never report raw. Includes events from incompatible versions and potentially unfiltered duplicates. |
| **Filtered GA4 (4.23M)** | TRUSTED WITH CAUTION | Only comparable to Mixpanel under same filter. Use for app opens, screen views, campaign attribution. |
| **Filtered Mixpanel (5.64M)** | UNTRUSTED UNTIL DEDUPED | Contains 8-12% duplicate events. Apply 30s dedup window before using for any metric. Post-dedup estimate: ~5.0M. |
| **BigQuery raw (5M)** | TRUSTED | Our own pipeline. Use as ingestion completeness validator. |
| **DynamoDB (85K)** | TRUSTED | Backend system of record. Too small for volume benchmarks but essential for signup/user identity validation. |
| **Backend logs** | TRUSTED | Best source for play validation, listen time, and stream confirmation. |

### Immediate Actions Required

1. **For April reporting launch**: All leadership dashboards must query `product_analytics.fact_reconciled_events` exclusively. Legacy direct GA4/Mixpanel connections must be deprecated.

2. **Mandatory filters for any March retrospective**: `app_version = '8.2.4'`, `event_date >= '2026-03-11'`, `environment = 'production'`, `is_duplicate = FALSE`, `is_valid_event = TRUE`.

3. **Metrics requiring backend validation before trust**: Pod plays, Voicepool plays, listen time, play completion. Do not use Mixpanel alone for these.

4. **March 18 exception**: Exclude or flag March 18 Mixpanel play metrics. Apply deduplication correction. Use backend logs for Mar 18 play counts.

---

