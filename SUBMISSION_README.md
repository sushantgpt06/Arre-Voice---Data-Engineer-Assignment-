# Arre Voice - Data Engineer Assignment Solution
## Production Case: March Data Accuracy, Reconciliation & Trusted Reporting

## Mission

Deliver an actionable solution pack to debug, reconcile, and fix Arre Voice's March reporting mismatch across GA4, Mixpanel, BigQuery, DynamoDB, backend logs, frontend analytics, and the data lake.

---

## Executive Summary

Arre Voice's March reporting shows a **33.3% gap** between GA4 (4.23M) and Mixpanel (5.64M) even after applying fair filters (app v8.2.4, Mar 11-31). This submission provides a complete, production-ready solution covering investigation, diagnosis, reconciliation table design, BigQuery SQL validation, data quality monitoring, incident RCA, and leadership guidance.

### Key Findings
- **Primary cause of gap**: Mixpanel duplicate firing (8-12%), different event taxonomies (10-15%), and identity stitching issues (5-8%)
- **Trustworthy source per metric**: Backend logs/DynamoDB for plays and signups; GA4 for app opens and attribution; Mixpanel for engagement intent (with dedup)
- **March 18 incident**: CDN latency spike caused Mixpanel SDK duplicate firing; requires data correction and idempotency key implementation
- **Leadership guidance**: March numbers are partially trustworthy only after applying mandatory filters (v8.2.4, Mar 11-31, deduplication, production environment)

---

## Table of Contents

| Section | File | Description |
|---------|------|-------------|
| A | [A_Investigation_Plan.md](A_Investigation_Plan.md) | 10-step first-day debugging plan |
| B | [B_Comparable_Analysis.md](B_Comparable_Analysis.md) | Valid vs invalid source comparisons |
| C | [C_Source_of_Truth.md](C_Source_of_Truth.md) | Metric-level source-of-truth matrix |
| D | [D_Mismatch_Diagnosis.md](D_Mismatch_Diagnosis.md) | GA4 vs Mixpanel root cause analysis |
| E | [E_BigQuery_Table_Design.md](E_BigQuery_Table_Design.md) | Final reconciled events table schema & DDL |
| F | [F_SQL_Tasks.md](F_SQL_Tasks.md) | 7 BigQuery queries with sample data & outputs |
| G | [G_Data_Quality_Checks.md](G_Data_Quality_Checks.md) | 12 daily DQ checks with severity & ownership |
| H | [H_Incident_RCA.md](H_Incident_RCA.md) | March 18 Mixpanel spike incident RCA |
| I | [I_Leadership_Recommendation.md](I_Leadership_Recommendation.md) | Leadership note on March number trustworthiness |

---

## Quick Reference: March Data Problem

### Raw Event Counts (March 1-31)

| Source | Event Count |
|--------|-------------|
| GA4 | 10,229,448 |
| Mixpanel | 5,644,723 |
| BigQuery / GBQ | 5,000,000 |
| DynamoDB | 85,000 |

### Filtered Comparison (Fair Filters Applied)

| Source | Mar 11-31, v8.2.4 | Variance |
|--------|-------------------|----------|
| GA4 | 4,234,174 | Baseline |
| Mixpanel | 5,644,723 | +33.3% higher |
| Gap | +1,410,549 events | To be diagnosed |

### Seven Core Questions Answered Across Sections

| Question | Answered In |
|----------|-------------|
| Why are the numbers different? | Section D |
| Which numbers are comparable? | Section B |
| Which source to trust per metric? | Section C |
| What SQL validates the mismatch? | Section F |
| What is the final BigQuery table? | Section E |
| What data quality checks run daily? | Section G |
| Can leadership trust March numbers? | Section I |

---

*All files are self-contained and cross-reference each other where applicable. Section F contains the shared sample datasets used for SQL demonstrations.*
