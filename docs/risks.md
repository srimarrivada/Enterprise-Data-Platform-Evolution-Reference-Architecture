# Architecture Risks and Mitigations

The main risks of this roadmap: how likely and serious each one is, how it would be noticed early, and how it is mitigated. Risks are grouped by the phase where they first appear; later phases inherit earlier risks unless a mitigation removes them. Ratings are qualitative (H / M / L) and would be re-assessed for any real implementation.

## Top Risks

| ID | Risk | Why it ranks high |
|---|---|---|
| [R-05](#phase-2--multi-source-analytics) | False identity links | Silent and damaging: one person's data shown as another's |
| [R-09](#phase-3--real-time-hybrid) | Event ordering assumed but not guaranteed | Easy to miss in design; corrupts current state without errors |
| [R-16](#phase-4--cloud-native-governed-lakehouse) | Governance gap on a new data product | Self-service multiplies the number of access paths |
| [R-20](#migration-risks) | Cutover before the new path is proven | The most common way modernization programs lose trust |

---

## Phase 1: Batch Data Lake

| ID | Risk | Likelihood | Impact | Early warning | Mitigation | See also |
|---|---|---|---|---|---|---|
| R-01 | **Source interface changes:** fields renamed, added or removed; API version deprecated | M | H | Rising reject rate; schema validation failures at Raw | Schema expectations per source; contract tests against a mock source; pinned API version; rebuild Raw from Landing after a mapping fix | [ADR-0001](adr/0001-yaml-driven-extraction.md), [F1](failure-modes.md#f1-extraction-fails-part-way) |
| R-02 | **Changes missed by incremental extraction:** source timestamps unreliable or late commits | L | H | Periodic full-extract comparison shows records missing from Raw | Re-read overlap on every window; at-least-once extraction with dedup in Raw; scheduled full extract to reconcile | [ADR-0001](adr/0001-yaml-driven-extraction.md) |
| R-03 | **Unbounded growth** of Landing and history | H | M | Storage and runtime trending up | Retention per zone; archiving policy from day one; incremental processing in Phase 4 | [ADR-0002](adr/0002-four-zone-lake.md), [NFR-08](nfrs.md) |
| R-04 | **Limited history before the first extraction:** early periods show attributes the platform first saw, not the ones in effect then | H | M | Trend lines that shift when old data is reloaded | One-time effective-dated historical extract; limitation documented on affected reports | [ADR-0002](adr/0002-four-zone-lake.md) |

## Phase 2: Multi-Source Analytics

| ID | Risk | Likelihood | Impact | Early warning | Mitigation | See also |
|---|---|---|---|---|---|---|
| R-05 | **False identity links:** a record attached to the wrong person | L | H | Precision drops on the known-answer test set; complaints about wrong profiles | Auto-match only with a clear margin over the runner-up; review band for uncertain scores; name-conflict checks on ID and email matches | [ADR-0003](adr/0003-master-person-index.md), [F4](failure-modes.md#f4-identity-resolution-is-uncertain) |
| R-06 | **Review queue backlog:** unresolved records accumulate | M | M | Queue age and size trending up | Named owner and turnaround target; publish blocked when a source's review rate exceeds its limit | [ADR-0003](adr/0003-master-person-index.md) |
| R-07 | **Transient sources lose data** before capture (overwritten exports, removed files) | M | H | Missing or late file arrivals | Capture on arrival; settle time and arrival alerts; Landing kept immutable | [F5](failure-modes.md#f5-source-file-is-late-partial-or-duplicated), [Bronze capture (Hybrid D12)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0012-bronze-captures-transient-sources.md) |
| R-08 | **Alert fatigue** from data quality warnings | M | M | Alerts closed without action; rising warning volume | Severity routing; warnings that don't block; thresholds reviewed when they fire too often or never | [ADR-0004](adr/0004-configurable-dq-framework.md) |

## Phase 3: Real-Time Hybrid

| ID | Risk | Likelihood | Impact | Early warning | Mitigation | See also |
|---|---|---|---|---|---|---|
| R-09 | **Event ordering assumed but not guaranteed:** per-type topics don't order one employee's events across types | M | H | Profiles showing an older value after a newer change | Order-independent consumer: field-level versions, newest change wins; single-topic option recorded | [ADR-0007](adr/0007-emp-id-partition-key.md), [F9](failure-modes.md#f9-events-late-or-out-of-order) |
| R-10 | **Events lost** between the source and the broker | L | H | Reconciliation's "stream behind batch" rate rising | Publish confirmed before acknowledging; claims released on failure so retries succeed; daily reconciliation as the backstop | [ADR-0008](adr/0008-webhook-dedup-redis.md), [F7](failure-modes.md#f7-broker-or-dedup-store-unavailable) |
| R-11 | **Outage longer than topic retention:** consumer can't catch up | L | H | Consumer lag approaching the retention window | Lag alerts well before retention; event archive for rebuilds; daily batch as the floor | [F8](failure-modes.md#f8-stream-consumer-down-or-replayed) |
| R-12 | **Two layers to run:** batch and streaming double the operational surface | H | M | Incidents or releases bottlenecked on a few people | Shared framework and standards; runbooks; Phase 4 moves both layers onto one platform | [ADR-0006](adr/0006-lambda-over-kappa.md) |
| R-13 | **Reconciliation masks real source problems:** stale values silently corrected every night | M | M | Steady or rising rate of corrections for the same fields | Correction rate reported and warned on; verification fails on gaps; runs refused when inputs are inconsistent | [ADR-0009](adr/0009-batch-stream-reconciliation.md), [F10](failure-modes.md#f10-batch-and-stream-disagree) |
| R-14 | **Sensitive payloads exposed** in logs, archives or dead-letter files | L | H | Payload content found in log samples | Logs carry identifiers only; restricted fields kept in a separate profile section; archive and dead-letter access limited to platform engineers | [Defense in depth (Hybrid D11)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md) |

## Phase 4: Cloud-Native Governed Lakehouse

| ID | Risk | Likelihood | Impact | Early warning | Mitigation | See also |
|---|---|---|---|---|---|---|
| R-15 | **Readers bypass the published snapshot** and read latest table versions | M | M | Queries against base tables from non-platform users | Data products built only on published views; base schemas granted to platform engineers only | [ADR-0011](adr/0011-unity-catalog-domain-schemas.md), [F12](failure-modes.md#f12-one-table-in-a-set-fails) |
| R-16 | **Governance gap on a new data product:** a view created without masks or row filters | M | H | Access tests show unmasked values for unauthorized groups | Governance functions required in every product view (code review); access tests per group before release; grants as versioned SQL | [ADR-0011](adr/0011-unity-catalog-domain-schemas.md) |
| R-17 | **Cloud cost overrun:** clusters left running, oversized capacity | M | M | Budget alerts; spend outside job windows | Job clusters that stop after each run; consumption-based services; budget alerts in the infrastructure code; teardown for non-production | [ADR-0012](adr/0012-managed-over-self-managed.md), [NFR-14](nfrs.md) |
| R-18 | **Managed-service differences:** features of the Kafka-compatible broker or document store behave differently from the self-managed versions | M | M | Client errors after cutover; throughput limits reached | Feature checks during the parallel run; duplicates handled downstream regardless of producer settings | [ADR-0012](adr/0012-managed-over-self-managed.md) |
| R-19 | **Lock-in** to one cloud or vendor | M | M | New requirements that only one vendor's service can meet | Open formats and protocols (Delta, Kafka protocol, MongoDB API); infrastructure as code; framework independent of the orchestrator | [ADR-0012](adr/0012-managed-over-self-managed.md), [ADR-0013](adr/0013-phase4-orchestrator.md) |
| R-20 | **Processing tied to one compute mode:** code that runs only on dedicated clusters | M | L | Rising cluster start times and cost | Recorded as a known constraint; DataFrame-only processing as the path to serverless compute | [ADR-0013](adr/0013-phase4-orchestrator.md) |

## Migration Risks

| ID | Risk | Likelihood | Impact | Early warning | Mitigation | See also |
|---|---|---|---|---|---|---|
| R-21 | **Cutover before the new path is proven** | M | H | Parallel run shortened; open reconciliation differences at sign-off | Exit criteria agreed before each phase starts; minimum parallel-run length; old path switched off only after sign-off | [F13](failure-modes.md#f13-old-and-new-paths-disagree-during-migration), [Roadmap](roadmap.md) |
| R-22 | **Consumers on two platforms at once:** some reports on the old path, some on the new, showing different numbers | M | M | Same KPI with different values in two places | One owning platform per KPI at any time; consumers moved by domain, not report by report | [Migration Notes](migration/) |
| R-23 | **Rollback no longer possible:** the old path decommissioned too early | L | H | Old infrastructure scheduled for removal before exit criteria are met | Decommissioning is a separate, approved step after the exit criteria | [Roadmap](roadmap.md) |

## Program Risks

| ID | Risk | Likelihood | Impact | Early warning | Mitigation | See also |
|---|---|---|---|---|---|---|
| R-24 | **Skills spread across many technologies** (Spark, streaming, document and event stores, cloud governance) | H | M | Knowledge held by one or two people per component | Shared framework and patterns across phases; pairing and runbooks; managed services reduce what must be operated | [Design Principle 15](design-principles.md#15-operate-what-differentiates-manage-the-rest) |
| R-25 | **Phases overlap:** the next phase starts before the current one exits | M | M | Exit criteria still open when new work begins | Roadmap rule: no phase starts before the previous one exits | [Roadmap](roadmap.md) |
| R-26 | **Design and system drift apart** | M | M | ADRs that no longer match how the platform works | Changes that alter a decision update or supersede its ADR in the same change | [ADR Index](adr/README.md) |

---

## How Risks Are Managed
- Each risk has an **owner** and is reviewed at every phase gate and at regular architecture reviews.
- **Early-warning signals** are part of standard monitoring, not separate checks.
- A risk that occurs becomes an incident; its root cause updates this register and, where needed, an ADR.
- Risks that no longer apply after a phase (e.g., R-12 after Phase 4 unifies the layers) are closed with the reason recorded.

## Related
- [Failure Modes and Recovery](failure-modes.md)
- [Non-Functional Requirements](nfrs.md)
- [Roadmap and Phase Exit Criteria](roadmap.md)
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*