# Arre Voice - Data Engineer Assignment Requirements


## Candidate Mission
Join as the Data Engineer responsible for solving Arre Voice's existing March reporting mismatch. Submit an actionable solution that the team can use to **debug, reconcile, and fix** the reporting layer. This is not a theory exercise.

---

## 1. Business Context
Arre Voice is a D2C social audio platform where users listen to audio posts, engage with creators, participate in Voicepools, and move through a product journey from acquisition to retention.

Backend and frontend teams are already capturing events. The problem is that **GA4, Mixpanel, BigQuery, DynamoDB, backend logs, frontend analytics, and the data lake** are showing different numbers. The company needs a trusted reporting layer that product, business, and leadership teams can rely on.

---

## 2. Current Systems

| Source | Current Purpose |
|--------|----------------|
| GA4 | App opens, screen views, frontend analytics events |
| Mixpanel | Funnels, retention, product behavior, user journey analytics |
| BigQuery / GBQ | Internal event storage and reporting |
| DynamoDB | Backend-confirmed user, content, and event records |
| Backend Logs | Server-side confirmations, failures, retries, stream/session logs |
| Frontend Analytics | Client-side user actions and product interaction events |
| Data Lake | Raw and historical product lifecycle data |

---

## 3. March Data Problem

### Raw Event Counts (March 1-31)

| Source | March 1-31 Event Count |
|--------|------------------------|
| GA4 | 10,229,448 |
| Mixpanel | 5,644,723 |
| BigQuery / GBQ | 5,000,000 |
| DynamoDB | 85,000 |

### Known Facts & Impact on Analysis

| Known Fact | Impact on Analysis |
|------------|-------------------|
| GA4 and Mixpanel include combined events across Pods and Voicepools | Content-type split is currently incomplete. Candidate must design how to split going forward. |
| Mixpanel started tracking only from March 11 | March 1-10 cannot be compared directly with Mixpanel. |
| Mixpanel is available only for app version 8.2.x | App-version filtering is mandatory before fair comparison. |
| Fair GA4 vs Mixpanel filter is March 11-31 and app version 8.2.4 | Only this filtered comparison is meaningful for GA4 vs Mixpanel. |

### Filtered Reality
After applying fair comparison filters:
- **GA4 = 4,234,174**
- **Mixpanel = 5,644,723**
- Mixpanel is higher by **1,410,549 events** (~33.3%)

---

## 4. Core Objective (Seven Questions to Answer)

The candidate must submit an actionable solution that answers:

1. Why are the numbers different?
2. Which numbers are comparable and which are not?
3. Which source should be trusted for each metric?
4. What SQL or pseudo-SQL should be used to validate the mismatch?
5. What should be the final BigQuery reporting table?
6. What data quality checks should run daily?
7. Can leadership trust March numbers or not?

---

## 5. Required Solution Pack (Deliverables)

The candidate must submit all sections below. Each section must be practical, specific, and directly tied to the March problem:

- **A. First-Day Investigation Plan** - 10-step debugging plan
- **B. Comparable vs Non-Comparable Analysis** - Classify each comparison
- **C. Metric-Level Source of Truth** - Define primary/secondary source per metric
- **D. GA4 vs Mixpanel Mismatch Diagnosis** - Top causes, verification, fix
- **E. Final BigQuery Reporting Table** - Schema and reporting logic
- **F. SQL / Pseudo-SQL Tasks** - Minimum 4 of 7 queries
- **G. Daily Data Quality Checks** - Logic, severity, owner, action
- **H. Incident Simulation: March 18 Spike** - Real debugging response and RCA
- **I. Final Leadership Recommendation** - Note under 200 words

---

## 6. Evaluation Criteria

| Evaluation Area | Weight |
|-----------------|--------|
| Practical debugging ability | 15% |
| Ability to identify valid vs invalid comparisons | 15% |
| Source-of-truth judgment | 15% |
| SQL / pseudo-SQL quality | 20% |
| Reconciliation and reporting table design | 15% |
| Data quality and monitoring thinking | 10% |
| Leadership communication clarity | 10% |
