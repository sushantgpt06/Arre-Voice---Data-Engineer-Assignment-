# Section B: Comparable vs Non-Comparable Analysis

## Objective
Classify each source comparison as **Valid**, **Invalid**, or **Valid only after filtering**. Explain the required filters and reasoning.

---

## Comparison Classification Matrix

| # | Comparison | Status | Required Filters & Reason |
|---|-----------|--------|---------------------------|
| 1 | **GA March total vs Mixpanel March total** | **INVALID** | Cannot compare full month totals directly. Mixpanel starts Mar 11 and GA4 includes all app versions. The populations are non-overlapping in both date range and app version. |
| 2 | **GA Mar 11-31 v8.2.4 vs Mixpanel Mar 11-31 v8.2.4** | **VALID AFTER FILTERING** | Must filter both to: `event_date BETWEEN '2026-03-11' AND '2026-03-31'` AND `app_version = '8.2.4'`. Only then both sources cover the same user population and time window. Still requires event name normalization and deduplication. |
| 3 | **DynamoDB 85K vs GA 10.2M** | **INVALID** | DynamoDB only stores backend-confirmed user/content/event records (validation layer), not raw analytics events. It is a subset by design — comparing subset to superset is meaningless. DynamoDB should be used as a validation reference, not a count benchmark. |
| 4 | **BigQuery 5M vs Mixpanel 5.6M** | **VALID AFTER FILTERING** | BigQuery stores processed events while Mixpanel stores client-side events. Filter both to same date range, app version, and event allowlist. Difference likely stems from deduplication gaps, late arrivals, or event name coverage mismatches in BQ pipeline. |
| 5 | **Backend logs vs DynamoDB** | **VALID** | Both are backend systems. Backend logs capture all server interactions; DynamoDB captures confirmed state changes. Expect near-parity after filtering backend logs to `status = 'success'` and matching `event_type` taxonomy. Discrepancy >1% indicates logging vs DB write inconsistency. |
| 6 | **Frontend play events vs backend stream events** | **VALID AFTER FILTERING** | Frontend fires `"play_started"`; backend confirms `"stream_start"`. Filter to matched `user_id + content_id + play_session_id` within a 60-second window. Unmatched frontend events = ghost plays (user abandoned before stream). Unmatched backend events = direct deeplink/stream without frontend event. |

---

## Comparability Decision Tree

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

## Filtering Rules Summary

### Rule 1: Temporal Alignment
```
WHERE event_date BETWEEN '2026-03-11' AND '2026-03-31'
```
Mixpanel started tracking on Mar 11. Any comparison including Mar 1-10 is invalid.

### Rule 2: Version Lock
```
WHERE app_version = '8.2.4'
```
Only app version 8.2.4 has Mixpanel support. Events from other versions must be excluded.

### Rule 3: Environment Gate
```
WHERE environment = 'production'
```
Staging and dev events inflate production counts artificially.

### Rule 4: Dedup Window
```
Keep FIRST event per (user, event_name, content_id) within 30-second window
```
Client-side retries create duplicates that must be collapsed.

### Rule 5: Event Allowlist
```
WHERE event_name IN (SELECT event_name FROM event_allowlist)
```
Only compare events tracked by BOTH sources. Exclude lifecycle/system events tracked by one only.

---

## Valid Comparison Pipeline

```
Raw GA4 Events          Raw Mixpanel Events
      |                          |
      v                          v
+----------------+      +----------------+
| Filter:        |      | Filter:        |
| - Mar 11-31    |      | - Mar 11-31    |
| - v8.2.4       |      | - v8.2.4       |
| - production   |      | - production   |
+--------+-------+      +--------+-------+
         |                       |
         v                       v
+----------------+      +----------------+
| Normalize:     |      | Normalize:     |
| event names    |      | event names    |
| content_type   |      | content_type   |
+--------+-------+      +--------+-------+
         |                       |
         v                       v
+----------------+      +----------------+
| Deduplicate:   |      | Deduplicate:   |
| 30s window     |      | 30s window     |
+--------+-------+      +--------+-------+
         |                       |
         +-----------+-----------+
                     |
                     v
          +--------------------+
          | FAIR COMPARISON    |
          | (Valid & Ready)    |
          +--------------------+
```

---

[Back to Index](SUBMISSION_README.md)
