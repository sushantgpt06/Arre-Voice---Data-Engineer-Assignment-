# Section C: Metric-Level Source of Truth

## Objective
Define the **primary source**, **secondary validation source**, and **reason** for each metric group.

---

## Source-of-Truth Matrix

### 1. Acquisition & App Usage

| Metric | Primary Source | Secondary Validation | Reason |
|--------|----------------|---------------------|--------|
| App open | GA4 | BigQuery raw | GA4 is purpose-built for app open tracking with session semantics. BQ raw validates ingestion completeness. |
| Screen view | GA4 | BigQuery raw | Same as above. GA4 has native `screen_view` event with automatic collection. |
| Campaign/referral source | GA4 | Mixpanel | GA4 UTM parsing is industry-standard. Mixpanel super properties validate attribution consistency. |

### 2. Signup & User Creation

| Metric | Primary Source | Secondary Validation | Reason |
|--------|----------------|---------------------|--------|
| Signup completed | DynamoDB | Mixpanel, GA4 | Backend DB is the system of record for account creation. Frontend events are unreliable due to network drops. |
| OTP verified | DynamoDB | Backend logs | OTP verification is a server-side transaction. DynamoDB + backend logs provide dual confirmation. |
| User created | DynamoDB | Backend logs | Same rationale. Backend is canonical for user identity. |

### 3. Audio Consumption

| Metric | Primary Source | Secondary Validation | Reason |
|--------|----------------|---------------------|--------|
| Pod play started | Backend logs | DynamoDB, frontend plays | Backend `stream_start` confirms the audio stream actually began. Frontend events alone may be fraudulent or dropped. |
| Voicepool play started | Backend logs | DynamoDB, frontend plays | Same rationale as pod plays. |
| Valid play (>30s) | Backend logs | Mixpanel | Duration-validated plays must come from backend stream logs which capture actual playback time. |
| Listen time | Backend logs (heartbeat aggregation) | BigQuery derived | Only backend can aggregate heartbeat intervals into true `listen_seconds`. Frontend estimates are guesses. |
| Play completed | Backend logs | DynamoDB | Completion is confirmed when `stream_end` with `completion_flag` is received server-side. |

### 4. Engagement

| Metric | Primary Source | Secondary Validation | Reason |
|--------|----------------|---------------------|--------|
| Like | Mixpanel | Backend logs, DynamoDB | Engagement intent is best captured at client (Mixpanel). Backend validation catches bot-like patterns. |
| Comment | Mixpanel | Backend logs | Comment submission is a user action; client is first signal, backend confirms persistence. |
| Share | Mixpanel | Backend logs | Same as comment. |
| Follow | DynamoDB | Mixpanel | Follow is a persistent state change stored in DynamoDB. Client event is secondary. |

### 5. Retention & Attrition

| Metric | Primary Source | Secondary Validation | Reason |
|--------|----------------|---------------------|--------|
| D1 retention | Mixpanel | BigQuery cohort query | Mixpanel has native retention charts but backend-derived cohorts in BQ are audit-friendly. |
| D7 retention | Mixpanel | BigQuery cohort query | Same as D1. |
| Dormant user | BigQuery derived | Mixpanel | Derived metric in BQ from event absence over 14 days. Mixpanel validates with session gap analysis. |
| One-play exit | BigQuery derived | Mixpanel funnel | Calculate in BQ from `first_play_date` and subsequent play absence. Mixpanel funnel validates. |

### 6. Leadership Reporting

| Metric | Primary Source | Secondary Validation | Reason |
|--------|----------------|---------------------|--------|
| Final DAU/WAU/MAU | BigQuery derived (from reconciled table) | Mixpanel, GA4 | Leadership needs a single reconciled table in BQ that applies all deduplication, filtering, and source-of-truth logic. |
| Final event counts | BigQuery derived (from reconciled table) | Mixpanel, GA4 | Same. The reconciled table is the only place all sources are unified. |
| Final play/listen metrics | BigQuery derived (from backend logs aggregation) | Mixpanel | Backend-derived metrics for plays, backend-aggregated listen time, with Mixpanel as sanity check only. |

---

## Source Reliability Hierarchy

```
                    SOURCE RELIABILITY PYRAMID

                           Backend
                         (DynamoDB +
                        Backend Logs)
                          | Trust |
                          | Level |
                          | HIGH  |
                     +----+-------+----+
                     |                 |
                  BigQuery           Mixpanel
                (Raw Events)      (Client Events)
                  | Trust |         | Trust |
                  | MEDIUM|         | MEDIUM|
                  +-------+         +-------+
                           \       /
                            \     /
                             \   /
                              \ /
                             GA4
                        (Attribution /
                         Auto-tracked)
                          | Trust |
                          | HIGH  |
                          | (for  |
                          |opens) |
                          +-------+
```

### Decision Framework

When two sources disagree, apply this priority:

1. **Persistent state changes** (signup, follow, user creation) → **DynamoDB wins**
2. **Server-confirmed actions** (play started, stream completed) → **Backend logs win**
3. **Client-intent actions** (like, comment, share) → **Mixpanel wins** (with backend validation)
4. **Attribution & acquisition** (app open, campaign, screen view) → **GA4 wins**
5. **Everything else** → **BigQuery derived (reconciled table)**

---

[Back to Index](SUBMISSION_README.md)
