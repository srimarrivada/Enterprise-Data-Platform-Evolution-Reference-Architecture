# Phase 2: Multi-Source Analytics

**Year 2 - On-premises. Batch, with identity resolution**

## Goal

Grow from one source to four, link records across sources to **one person**, give HR business partners a **360° employee profile**, and extend quality rules to the new sources. All of this runs on the unchanged Phase 1 framework, zones and pipeline.

![Phase 2 Architecture](../diagrams/svg/02-phase2-multi-source.drawio.svg)

---

## Sources

| Source | Interface | Identifies People By | Extraction |
|---|---|---|---|
| HCM system of record | SOAP, custom reports, RaaS | Worker ID (master) | Unchanged from Phase 1 |
| Learning platform | REST, OAuth 2.0, JSON, cursor paging | Learner ID; often the employee ID, sometimes only name and email | Full extract (no change filter) |
| Survey platform | SFTP file drops (CSV), SSH key or password, verified host keys | Respondent email only | Incremental by file modification time |
| Recruiting module | RaaS + custom reports | Candidate ID; hired worker ID when linked, otherwise name, email and start date | Incremental |

Every new source is a new manifest. The two new interface types (cursor-paged REST and file drops) are added as framework plugins, and nothing else in the framework changes. → [ADR-0001](../adr/0001-yaml-driven-extraction.md)

### File Drops
- **Settle Time:** Files younger than a configured age may still be uploading, so they are deferred. The watermark stops at the same cut-off, so a deferred file is picked up next run instead of being skipped.
- **Clock Differences:** A re-read margin covers differences between the file server's clock and ours. The Raw zone removes the resulting duplicates.
- **Lineage:** Original file names are recorded with each Landing run.
- **CSV Handling:** Byte-order marks, quoted line breaks and blank cells are handled. A row with more values than the header is rejected with a reason.

---

## Master Person Index (MPI)

The same person has different identifiers in each source. The MPI resolves every source record to one stable **`person_id`** in four passes. Each pass handles only the records the previous passes left unresolved. → [ADR-0003](../adr/0003-master-person-index.md)

| Pass | Match Rule | Guard | Expected Share (design assumption) |
|---|---|---|---|
| 1 | Exact employee ID | Last name must not clearly disagree (catches ID typos) | ~95% |
| 2 | Normalized email (trimmed, case-insensitive) matching exactly one worker | Same name-conflict guard | ~3% |
| 3 | Fuzzy score: last name, first name (nicknames + similarity), hire date, job family | Auto-match only above a threshold **and** with a clear margin over the runner-up | ~1.5% |
| 4 | Manual review queue, with the top candidates and their scores | A person decides | ~0.5% |

- **Missing fields score zero:** They are not reweighted, so a name alone can never auto-match: two people can share a name.
- **Unmatched is a valid answer:** External candidates who were never hired correctly stay unmatched.
- **Explainable:** Every crosswalk row records its status, method, score and alternatives.
- **Stable:** `person_id` is derived from the worker ID, so it survives rebuilds.
- **Publish gate:** If any source's review rate exceeds its limit, or a match points to a worker that doesn't exist, the crosswalk is not published.
- **Source-specific matching:** Recruiting uses the candidate's start date in place of a hire date, which links hired candidates whose record never stored the worker ID. Survey respondents carry only an email, so pass 2 resolves them.

Accuracy is measured against a known-answer set before any threshold change: precision (no false links), recall (automatic resolution rate) and review rate per source.

---

## Data Model

The Phase 1 model grows, with every new dimension and fact conformed on `person_id`.

- **New Dimensions:** Course, skill, survey question, requisition, source channel
- **New Facts:** Training completion, certification, engagement score, requisition pipeline, candidate application, offer, workforce monthly snapshot
- **New Consumption Aggregates:** Training compliance, attrition trends, recruiting pipeline summary, engagement by organization

---

## 360° Employee Profile

One document per employee in a **document store** (e.g., MongoDB), covering employment, organization, manager, learning and engagement. → [ADR-0005](../adr/0005-document-store-360-profile.md)

- Rebuilt from Curated after each batch
- Serves the **HR business partner portal**: one read per employee page
- **Restricted attributes** (e.g., compensation) are kept in a separate section and served only to authorized roles → [Hybrid D11](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md)

In Phase 3, the same store also holds the real-time profile, reconciled with the batch layer.

---

## Data Quality

The Phase 1 quality gates continue unchanged, and each new source adds its own suite. → [ADR-0004](../adr/0004-configurable-dq-framework.md)

| Check type | Fails when |
|---|---|
| Not null | The share of nulls in a column exceeds its limit |
| Unique | A key combination appears more than once |
| Referential integrity | A value has no match in the referenced table |
| Accepted values | A value isn't in the allowed list |
| Value range | A value is below the minimum or above the maximum (e.g., survey scores outside the scale) |
| SCD2 consistency | Wrong number of current rows, gaps or overlaps in history |
| Reconciliation | Two counts (same or different tables, or zones) differ beyond a tolerance |
| Row-count change | Volume moved more than allowed from the last passing run |
| Distribution | Category mix drifted from the last passing run |

- **Critical** failures skip the gated tables; **warnings** alert without blocking.
- Results are recorded with their measurements for trend reporting.
- The MPI review gate adds an identity-specific control on top of these suites.

---

## Serving

- **Profile store:** 360° profile for the HR business partner portal
- **BI** (e.g., Tableau) over Hive JDBC: Headcount, attrition, learning compliance, recruiting and engagement dashboards
- **Onboarding integration:** New-hire records found in the batch are pushed to an IT service management tool (e.g., ServiceNow) for provisioning and access setup

---

## Orchestration

The generated pipeline from Phase 1 picks up the new configs automatically:
- Extraction and Raw tasks for the three new sources
- The MPI task, after the worker dimension and the new Raw tables
- A DQ suite per new source
- The profile rebuild and onboarding push, after the publish

No dependencies are wired by hand. → [ADR-0001](../adr/0001-yaml-driven-extraction.md)

---

## Design Targets

These are design parameters, not measured results:

| Parameter | Target |
|---|---|
| Sources | 4 batch (HCM, learning, survey, recruiting) |
| Data model | 14 dimensions (SCD2) · 15 facts |
| MPI | No false links; ≥ 99% resolved automatically (passes 1–3); review rate per source within its limit |
| Profile read | Under 2 seconds per employee page |
| Profile store | 3-node replica set |
| BI | 15+ dashboards |

---

## Failure Handling

| Failure | Designed outcome |
|---|---|
| Identity match is uncertain | Goes to review; publish blocked if the review rate spikes → [F4](../failure-modes.md#f4-identity-resolution-is-uncertain) |
| File late, partial or duplicated | Young files deferred; duplicates removed; bad rows rejected → [F5](../failure-modes.md#f5-source-file-is-late-partial-or-duplicated) |
| Critical DQ check fails on a new source | Gated tables not rebuilt; previous version stays live → [F2](../failure-modes.md#f2-critical-data-quality-check-fails) |

---

## Exit Criteria

Phase 2 is complete when:
- All four sources run through the unchanged framework
- Identity resolution produces no false links on a known-answer test set, and unresolved records reach the review queue
- A critical DQ failure holds the publish and raises an alert
- The profile is rebuilt after each batch and served to the HR business partner portal

→ [Roadmap](../roadmap.md#phase-2--multi-source-analytics-year-2)  
→ Previous: [Phase 1 → Phase 2](../migration/phase1-to-phase2.md)  
→ Next: [Phase 2 → Phase 3](../migration/phase2-to-phase3.md)

---

## Related Documents

- [Phase 1: Batch Data Lake](phase1-batch-lake.md)
- [Phase 3: Real-Time Hybrid](phase3-lambda.md)
- [Risks: Phase 2](../risks.md#phase-2--multi-source-analytics)
- [ADR Index](../adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*
