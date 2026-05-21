# Section G: Daily Data Quality Checks

## Objective
Create a daily checklist with logic, severity, owner, and action for each check.

---

## Daily Checklist

| Check ID | Check Name | Logic / Query | Severity | Owner | Action if Failed |
|----------|-----------|---------------|----------|-------|------------------|
| **DQ-01** | **Event volume drop >30%** | `SELECT event_date, COUNT(*) AS cnt FROM bq_raw_events WHERE event_date = CURRENT_DATE()-1 GROUP BY event_date HAVING cnt < (SELECT AVG(cnt)*0.7 FROM daily_volumes WHERE event_date >= CURRENT_DATE()-8)` | **P0-Critical** | Data Engineering | Page on-call. Trigger pipeline health check. Investigate upstream source (GA4 export, Mixpanel API, Kafka). |
| **DQ-02** | **Mixpanel > GA4 by >20%** | Compare yesterday's filtered counts. If `(mp_count - ga_count) / ga_count > 0.20`, alert. | **P1-High** | Data Engineering | Run duplicate detection query (Section F, Task 2). Check for SDK double-firing or new auto-tracked events. |
| **DQ-03** | **Duplicate rate >5%** | `SELECT COUNTIF(is_duplicate)/COUNT(*) AS dup_rate FROM reconciled WHERE event_date = CURRENT_DATE()-1` | **P1-High** | Data Engineering | Investigate dedup window logic. Check for network retry storms. Adjust deduplication threshold if needed. |
| **DQ-04** | **Backend-confirmed plays missing** | `SELECT COUNT(*) FROM reconciled WHERE event_name LIKE '%play%' AND is_backend_confirmed = FALSE AND event_date = CURRENT_DATE()-1` | **P1-High** | Backend Engineering | Frontend events without backend confirmation indicate ghost plays or stream API failure. Check CDN/stream health. |
| **DQ-05** | **App version missing >1%** | `SELECT COUNTIF(app_version IS NULL)/COUNT(*) AS missing_pct FROM bq_raw_events WHERE event_date = CURRENT_DATE()-1` | **P2-Medium** | Mobile Engineering | Missing app version breaks fair comparison. Likely SDK initialization race condition. Alert mobile team. |
| **DQ-06** | **Staging/dev data in production** | `SELECT COUNT(*) FROM bq_raw_events WHERE environment != 'production' AND event_date = CURRENT_DATE()-1` | **P0-Critical** | Data Engineering | Immediately quarantine non-production events. Audit pipeline routing. Block staging Kafka topics from production BQ. |
| **DQ-07** | **Events delayed >24 hours** | `SELECT COUNT(*) FROM bq_raw_events WHERE TIMESTAMP_DIFF(ingestion_timestamp, event_timestamp, HOUR) > 24 AND event_date = CURRENT_DATE()-1` | **P2-Medium** | Data Engineering | Investigate ingestion lag. Check GA4 export schedule, Mixpanel API rate limits, or BQ load jobs. |
| **DQ-08** | **Unknown event names** | `SELECT event_name, COUNT(*) FROM bq_raw_events WHERE event_date = CURRENT_DATE()-1 AND event_name NOT IN (SELECT event_name FROM event_allowlist) GROUP BY event_name` | **P2-Medium** | Product Analytics | New untracked events indicate SDK changes or feature launches. Update event taxonomy and allowlist. |
| **DQ-09** | **Pod/Voicepool content_type missing** | `SELECT COUNTIF(content_type IS NULL)/COUNT(*) AS null_pct FROM bq_raw_events WHERE event_name LIKE '%play%' AND event_date = CURRENT_DATE()-1` | **P2-Medium** | Data Engineering | NULL content_type breaks content split reporting. Infer from screen_view context or enforce SDK tagging. |
| **DQ-10** | **Timezone/date boundary mismatch** | Compare GA4 vs BQ daily counts grouped by UTC date vs property timezone date. Delta >2% triggers. | **P2-Medium** | Data Engineering | Reconcile timezone configurations. Standardize all reporting to UTC date boundaries. |
| **DQ-11** | **Backend retries increased** | `SELECT SUM(retry_count) AS total_retries FROM backend_logs WHERE log_date = CURRENT_DATE()-1` | **P2-Medium** | Backend Engineering | Retry storms indicate downstream pressure (DB, cache, third-party API). Investigate before cascade failure. |
| **DQ-12** | **Identity stitching failure** | `SELECT COUNTIF(user_id IS NULL AND anonymous_id IS NULL)/COUNT(*) AS orphan_pct FROM reconciled WHERE event_date = CURRENT_DATE()-1` | **P1-High** | Data Engineering | Orphan events cannot be attributed to users. Check identity resolution service health and alias mapping table. |
| **DQ-13** | **Mixpanel duplicate rate by event >10%** (Post-Mar 18 addition) | `SELECT event_name, COUNTIF(is_duplicate)/COUNT(*) AS dup_rate FROM reconciled WHERE event_date = CURRENT_DATE()-1 AND source LIKE '%Mixpanel%' GROUP BY event_name HAVING dup_rate > 0.10` | **P0-Critical** | Data Engineering | Auto-quarantine affected date partition in BQ. Trigger incident response. Investigate CDN/SDK layer. |

---

## Severity Definitions

| Severity | Response Time | Escalation Channel | Action Required |
|----------|---------------|--------------------|-----------------|
| **P0-Critical** | < 15 minutes | PagerDuty + Slack #data-alerts | Stop-the-line. Pipeline quarantine. Leadership notification. |
| **P1-High** | < 2 hours | Slack #data-alerts + email | Same-day fix. Ticket created. Next-day retro if needed. |
| **P2-Medium** | < 24 hours | Slack #data-notices | Weekly batch fix. Added to sprint backlog. |

---

## Monitoring Dashboard

```
+----------------------------------------------------------+
|  ARRE VOICE - DAILY DATA QUALITY DASHBOARD               |
|  Date: 2026-03-19                                        |
+----------------------------------------------------------+
|                                                          |
|  [PASS]  DQ-01  Volume Drop              OK              |
|  [FAIL]  DQ-02  MP > GA4 by 22%          ALERT          |
|  [WARN]  DQ-03  Dup Rate: 4.8%           WARNING        |
|  [PASS]  DQ-04  Backend Plays            OK              |
|  [PASS]  DQ-05  App Version              OK              |
|  [PASS]  DQ-06  No Staging Leak          OK              |
|  [PASS]  DQ-07  Delay <24h               OK              |
|  [WARN]  DQ-08  3 Unknown Events         WARNING        |
|  [PASS]  DQ-09  Content Type             OK              |
|  [PASS]  DQ-10  Timezone Aligned         OK              |
|  [PASS]  DQ-11  Retries Normal           OK              |
|  [PASS]  DQ-12  Stitching OK             OK              |
|  [PASS]  DQ-13  Post-Mar18 Dup Check     OK              |
|                                                          |
+----------------------------------------------------------+
|  YESTERDAY SNAPSHOT                                      |
|  +------------------+  +------------------+              |
|  | GA4:    142,301  |  | Backend Plays:   |              |
|  | Mixpanel:173,607 |  |   118,450        |              |
|  | Diff:   +22%     |  | Ghost Plays: 204 |              |
|  +------------------+  +------------------+              |
+----------------------------------------------------------+
```

---

## Automation Setup

### Airflow DAG Structure
```
data_quality_checks_dag
  |-- task: dq_volume_check      (runs 03:00 UTC)
  |-- task: dq_mp_vs_ga4         (runs 03:05 UTC)
  |-- task: dq_duplicate_rate    (runs 03:10 UTC)
  |-- task: dq_backend_missing   (runs 03:15 UTC)
  |-- task: dq_staging_leak      (runs 03:20 UTC)
  |-- task: dq_identity_orphans  (runs 03:25 UTC)
  |-- task: dq_summary_notify    (runs 03:30 UTC)
```

### Slack Alert Template (P0)
```
:red_circle: P0 DATA QUALITY ALERT
Check: {check_name} ({check_id})
Severity: P0-Critical
Date: {event_date}
Value: {measured_value}
Threshold: {threshold}
Owner: {owner}
Action: {recommended_action}
Runbook: https://wiki.arrevoice.io/runbooks/{check_id}
```

---

## Weekly DQ Review Meeting Agenda

1. Review all P0/P1 alerts from past 7 days
2. Validate fix effectiveness (re-run check on corrected data)
3. Tune thresholds if false-positive rate > 10%
4. Add new checks for emerging patterns
5. Update runbooks based on lessons learned

---

[Back to Index](SUBMISSION_README.md)
