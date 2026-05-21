# Section H: Incident Simulation: March 18 Spike

## Incident Overview

| Field | Detail |
|-------|--------|
| **Incident Date** | March 18, 2026 |
| **Detected At** | March 19, 2026 03:05 UTC (via DQ-02 alert) |
| **Duration** | ~3 hours (09:00 - 12:00 UTC) |
| **Impacted Source** | Mixpanel |
| **Symptom** | Mixpanel events suddenly increased by 70% overnight. GA4 and DynamoDB remained flat. |

---

## Timeline: Hour-by-Hour Investigation

### Hour 0 — Alert Triggered (03:05 UTC, Mar 19)

Mixpanel daily volume alert fires:

| Date | Mixpanel Events | Baseline Avg (Mar 11-17) | Variance |
|------|-----------------|--------------------------|----------|
| Mar 18 | 280,000 | 165,000 | **+70%** |

Cross-checks:
- GA4 Mar 18 count = 155,000 (flat, within normal variance)
- DynamoDB confirmed events Mar 18 = 5,200 (flat)
- BigQuery raw events Mar 18 = 278,000 (close to Mixpanel, suggesting ingestion pipeline is healthy)

**Initial Hypothesis**: Issue is isolated to Mixpanel client-side SDK, not upstream pipeline.

---

### Hour 1 — Narrow Down Scope (04:00 UTC)

**Check 1**: Run daily comparison (Section F, Task 1) for Mar 18 only.

| Date | GA4 Events | Mixpanel Events | Difference | % Diff |
|------|------------|-----------------|------------|--------|
| Mar 18 | 155,000 | 280,000 | +125,000 | **+80.6%** |

**Check 2**: Split Mixpanel Mar 18 by event_name.

| Event Name | Mar 18 Count | Baseline Avg | Variance |
|------------|-------------|-------------|----------|
| Pod Play | 140,000 | 78,000 | **+79.5%** |
| Voicepool Play | 42,000 | 38,000 | +10.5% |
| App Open | 48,000 | 45,000 | +6.7% |
| Like | 28,000 | 27,000 | +3.7% |
| Other | 22,000 | 18,000 | +22.2% |

**Finding**: The spike is isolated to **"Pod Play" events in Mixpanel** (+79.5%).

**Narrowing**: Not a general volume issue. Specific to one event type.

---

### Hour 2 — Drill into Pod Play Duplicates (05:00 UTC)

**Check 3**: Run duplicate detection (Section F, Task 2) for Mar 18, filtered to `event_name = 'Pod Play'`.

```sql
WITH with_lag AS (
  SELECT
    event_timestamp,
    LAG(event_timestamp) OVER (
      PARTITION BY distinct_id, event_name, content_id
      ORDER BY event_timestamp
    ) AS prev_ts
  FROM mixpanel_events
  WHERE event_date = '2026-03-18'
    AND app_version = '8.2.4'
    AND event_name = 'Pod Play'
)
SELECT
  COUNTIF(TIMESTAMP_DIFF(event_timestamp, prev_ts, SECOND) <= 30) AS duplicate_events,
  COUNT(*) AS total_events,
  ROUND(COUNTIF(TIMESTAMP_DIFF(event_timestamp, prev_ts, SECOND) <= 30) * 100.0 / COUNT(*), 2) AS duplicate_pct
FROM with_lag;
```

**Result**:

| duplicate_events | total_events | duplicate_pct |
|-----------------|-------------|---------------|
| 68,000 | 140,000 | **48.5%** |

**Baseline**: Mar 17 duplicate rate for Pod Play = 3.2%.

**Conclusion**: Massive duplicate firing started on Mar 18. 48.5% of Pod Play events are duplicates.

---

### Hour 3 — Cross-Reference with Releases (06:00 UTC)

**Check 4**: Query `backend_logs` for app_version distribution on Mar 18 vs Mar 17.

| Date | v8.2.4 | v8.2.3 | Older |
|------|--------|--------|-------|
| Mar 17 | 98.1% | 1.5% | 0.4% |
| Mar 18 | 97.9% | 1.7% | 0.4% |

**Finding**: No new app version released on Mar 17 or 18. Version mix is stable.

**Check 5**: Check Mixpanel project settings change log.

**Finding**: No configuration changes in Mixpanel on Mar 17-18. No new auto-tracking rules enabled.

**Elimination**: Not an app release issue. Not a Mixpanel configuration change.

---

### Hour 4 — Network & SDK Layer (07:00 UTC)

**Check 6**: Examine `backend_logs.retry_count` for Mar 18.

| Date | Avg Retry Count | Max Retry Count | Events with retry > 0 |
|------|----------------|----------------|-----------------------|
| Mar 17 | 0.1 | 2 | 3.2% |
| Mar 18 | **1.4** | **5** | **34.7%** |

The retry count average jumped from 0.1 to 1.4 per event on Mar 18 between 09:00-12:00 UTC.

**Hypothesis**: A CDN or API degradation caused frontend play events to timeout. The Mixpanel SDK retry mechanism fired the same event multiple times when the initial ACK was lost.

**Check 7**: Infrastructure monitoring.

**Found**: CDN edge node `edge-apac-03` had 30-second latency spikes from 09:00-12:00 UTC on Mar 18.

| Time (UTC) | edge-apac-03 p99 Latency | Threshold |
|------------|-------------------------|-----------|
| 08:00-09:00 | 120ms | 500ms |
| **09:00-10:00** | **4,200ms** | **500ms** |
| **10:00-11:00** | **8,500ms** | **500ms** |
| **11:00-12:00** | **6,800ms** | **500ms** |
| 12:00-13:00 | 180ms | 500ms |

---

### Hour 5 — Root Cause Confirmed (08:00 UTC)

## Root Cause Analysis (RCA)

| RCA Field | Detail |
|-----------|--------|
| **Incident date** | March 18, 2026 |
| **Impacted source** | Mixpanel (frontend analytics) |
| **Impacted event(s)** | Pod Play (primary), minor impact on Voicepool Play |
| **Root cause** | CDN edge node degradation (`edge-apac-03`) caused 30s API latency spikes. Mixpanel SDK retry logic fired duplicate events without server-side deduplication. GA4 uses a batched, offline-queue approach that handles retries more gracefully. DynamoDB only stores confirmed backend events unaffected by frontend network issues. |
| **Timeline of failure** | 09:00 UTC - CDN latency spikes begin; 09:15 UTC - Mixpanel SDK starts retrying timed-out events; 12:00 UTC - CDN recovers; Mar 19 03:05 UTC - DQ alert triggers |
| **Business impact** | Mixpanel DAU and play count dashboards became unreliable for Mar 18. Any leadership report using Mixpanel for play metrics is overstated by ~70%. |
| **Data correction required** | Deduplicate Mar 18 Mixpanel events using 30-second window per `distinct_id + event_name + content_id`. Exclude duplicates from downstream reporting. Backfill `fact_reconciled_events` for Mar 18 with `is_duplicate = TRUE`. |
| **Product/engineering fix** | 1. Mobile team to implement client-side request-idempotency key in Mixpanel SDK.<br>2. Backend to reject duplicate events with same idempotency key within 5 minutes.<br>3. DevOps to add CDN latency alert threshold at 5s p99. |
| **Prevention check added** | **DQ-13**: "Mixpanel duplicate rate by event type > 10%" — P0 alert with auto-quarantine of affected date partition in BQ. |

---

## Post-Incident Metrics

### Corrected Mar 18 Numbers

| Metric | Raw Mixpanel | After Deduplication | After Full Reconciliation |
|--------|-------------|--------------------|--------------------------|
| Pod Play | 140,000 | 72,000 (-48.5%) | 71,800 (-0.3% backend mismatch) |
| Total Events | 280,000 | ~212,000 | ~210,500 |
| Variance vs GA4 | +80.6% | +36.8% | +35.8% |

After deduplication, the Mar 18 variance drops from +80.6% to a more reasonable +35.8% (within the expected 33.3% baseline gap for the month).

---

## Lessons Learned

1. **Client-side SDK retries are dangerous without idempotency.** Mixpanel's default retry behavior amplified a transient network issue into a 70% data spike.
2. **Multi-source validation caught this early.** Because GA4 and DynamoDB stayed flat, the anomaly was immediately scoped to Mixpanel.
3. **Daily DQ checks are the safety net.** DQ-02 (MP > GA4 by >20%) triggered within hours of data landing, minimizing the blast radius.
4. **48h reporting lock would have prevented bad decisions.** If leadership had accessed unverified Mar 18 numbers on Mar 19 morning, product decisions would have been based on inflated play metrics.

---

[Back to Index](SUBMISSION_README.md)
