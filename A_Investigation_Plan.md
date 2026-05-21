# Section A: First-Day Investigation Plan

## 10-Step Practical Debugging Plan

Each step includes: what to check, why it matters, expected output, and the decision it enables.

---

| Step | What to Check | Why It Matters | Expected Output | Decision Enabled |
|------|---------------|----------------|-----------------|------------------|
| **1** | **Date range mismatch** — Verify that GA4 and Mixpanel queries both use UTC-aligned March 11-31 boundaries. Check timezone configurations in both platforms. | March data is time-bound; a 1-hour boundary shift can skew daily counts by thousands. | Both platforms should show identical row counts for Mar 11 00:00:00 UTC through Mar 31 23:59:59 UTC. | Determines whether the gap is real or an artifact of timezone drift. |
| **2** | **Timezone mismatch** — Compare `event_timestamp` vs `ingestion_timestamp` in BigQuery. Check if GA4 reports in user's local timezone while Mixpanel reports in UTC. | GA4 defaults to property timezone; Mixpanel to UTC. A +5:30 IST offset shifts 50% of India traffic into adjacent days. | Timezone delta distribution table per geo. | Decide whether to normalize all timestamps to UTC before comparison. |
| **3** | **App version mismatch** — Count events per `app_version` in GA4 and Mixpanel for Mar 11-31. Verify v8.2.4 filter coverage. | Mixpanel only supports v8.2.x. Events from v8.2.3 or earlier should not be counted in fair comparison. | App version distribution: v8.2.4, v8.2.3, older. If v8.2.3 contributes 5%+ of GA4 events, the gap is partly explained. | Apply strict v8.2.4 filter before any further comparison. |
| **4** | **Event name mismatch & raw vs normalized mapping** — Map GA4 `event_name` to Mixpanel `event_name`. Identify events present in one but absent in the other. | `"pod_play_started"` in GA4 may map to `"Pod Play"` in Mixpanel, but lifecycle events like `"app_background"` may exist only in GA4. | Event name mapping matrix with inclusion/exclusion flags. GA4 has 20+ events; Mixpanel may track 15. | Build normalized event taxonomy. Exclude unmatched events from head-to-head comparison. |
| **5** | **Pods vs Voicepools split & missing `content_type`** — Check how many GA4 events have NULL `content_type` versus explicit `"pod"` / `"voicepool"`. Cross-reference with Mixpanel. | GA4 combines all play events without `content_type` for some event types. Mixpanel splits them. A NULL-heavy GA4 table inflates totals unfairly. | Percentage of events with NULL `content_type` per source. Pod/Voicepool ratio comparison. | Decide whether to impute `content_type` from `screen_view` context or exclude NULL `content_type` events from fair comparison. |
| **6** | **iOS vs Android split & app release differences** — Compare platform ratios between GA4 and Mixpanel. Check if iOS Mixpanel SDK was released later than Android. | If iOS v8.2.4 shipped on Mar 15 but Android on Mar 11, the first 5 days have asymmetric platform coverage. | Platform distribution per date. Version adoption curve. | Adjust comparison window or stratify by platform separately. |
| **7** | **Duplicate event firing & backend retry duplicates** — Within a 30-second window per user-event pair, count duplicate rows in Mixpanel vs GA4. Also check `backend_logs.retry_count`. | Mixpanel client SDK may fire the same event twice on poor network. GA4 deduplicates by default. Backend retries with `retry_count > 0` create duplicates. | Duplicate rate per source per day. If Mixpanel duplicate rate > 5%, that explains a portion of the 33.3% gap. | Implement deduplication logic: keep first event in a 30s window per user-event-content tuple. |
| **8** | **Missing events and late-arriving events** — Check `ingestion_timestamp - event_timestamp` lag in BigQuery. Count events ingested > 24h after event time. Also check if GA4 export runs once daily vs Mixpanel real-time. | Late arrivals shift event counts across dates. GA4 intraday export is configurable and may lag. | Lag distribution histogram. Count of events with lag > 1h, > 24h. | Normalize to ingestion-based reporting or wait for 48h before finalizing March numbers. |
| **9** | **Backend-generated/system events vs real user actions** — Inspect `backend_logs` for automated/system events (health checks, push notifications, cron jobs) that may be logged as "events". | System events should not count in user-facing metrics. If backend logs include cron-fired `"session_start"` events, Mixpanel may ingest them. | List of `event_type` values where `user_id IS NULL` or `device_id` matches internal services. | Exclude system-generated events from user metrics. Create allowlist of user-initiated event types. |
| **10** | **Staging/dev data leaking into production** — Check `environment` dimension in raw events. Look for events tagged `"staging"`, `"dev"`, `"test"` in BigQuery or Mixpanel. | QA automation or beta testing generates events that inflate production counts but represent non-user activity. | Count of events with `environment != "production"` per source. | Filter to `environment = "production"` only. Add this as a mandatory `WHERE` clause in all reporting queries. |

---

## Investigation Flowchart

```
+-----------------------------------------------------+
|  Step 1: Date Range & Timezone Alignment            |
|  --> If mismatch found: normalize to UTC, re-run    |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 2: App Version Filter (v8.2.4 only)           |
|  --> Apply before any comparison                    |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 3: Event Taxonomy Mapping                     |
|  --> Build allowlist; exclude unmatched events      |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 4: Content Type & Platform Stratification     |
|  --> Split Pod vs Voicepool; iOS vs Android         |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 5: Deduplication & Duplicate Rate Analysis    |
|  --> 30s window dedup; quantify impact              |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 6: Late Arrival & Data Completeness           |
|  --> Wait 48h; lock monthly numbers post-delay      |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 7: Environment Filter (production only)       |
|  --> Quarantine staging/dev events                  |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 8: Identity Stitching Validation              |
|  --> Check anonymous vs logged-in user counts       |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 9: System Event Exclusion                     |
|  --> Filter cron/automated events                   |
+-----------------------------------------------------+
                        |
                        v
+-----------------------------------------------------+
|  Step 10: Cross-System Trend Validation             |
|  --> Compare daily trends across all sources        |
+-----------------------------------------------------+
```

---

[Back to Index](SUBMISSION_README.md)
