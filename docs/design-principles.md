# Design Principles

The roadmap rests on one idea:

> Evolve the platform in steps that each stand on their own: Reuse what works, change one thing at a time, and never let consumers see data that hasn't been checked.

Each principle links to the decision record (ADR) that applies it. Principles shared with the [Hybrid Reference Architecture](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/design-principles.md) are marked **(shared)**.

| # | Principle | Decision |
|---|---|---|
| [1](#1-reuse-before-replace) | Reuse before replace | [ADR-0001](adr/0001-yaml-driven-extraction.md), [ADR-0012](adr/0012-managed-over-self-managed.md) |
| [2](#2-configuration-over-code) | Configuration over code | [ADR-0001](adr/0001-yaml-driven-extraction.md), [ADR-0004](adr/0004-configurable-dq-framework.md) |
| [3](#3-every-zone-can-be-rebuilt) | Every zone can be rebuilt | [ADR-0002](adr/0002-four-zone-lake.md) |
| [4](#4-history-before-breadth) | History before breadth | [ADR-0002](adr/0002-four-zone-lake.md) |
| [5](#5-one-person-one-identifier) | One person, one identifier | [ADR-0003](adr/0003-master-person-index.md) |
| [6](#6-a-wrong-answer-is-worse-than-no-answer) | A wrong answer is worse than no answer | [ADR-0003](adr/0003-master-person-index.md), [ADR-0004](adr/0004-configurable-dq-framework.md) |
| [7](#7-batch-is-the-truth-streaming-adds-speed) | Batch is the truth, streaming adds speed | [ADR-0006](adr/0006-lambda-over-kappa.md), [ADR-0009](adr/0009-batch-stream-reconciliation.md) |
| [8](#8-dont-depend-on-arrival-order) | Don't depend on arrival order | [ADR-0007](adr/0007-emp-id-partition-key.md) |
| [9](#9-every-load-and-every-replay-is-safe) | Every load and every replay is safe (shared) | [ADR-0008](adr/0008-webhook-dedup-redis.md), [Hybrid D9](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0009-idempotent-loads.md) |
| [10](#10-check-before-consumers-see-it) | Check before consumers see it (shared) | [ADR-0004](adr/0004-configurable-dq-framework.md), [ADR-0010](adr/0010-delta-over-hive-parquet.md) |
| [11](#11-publish-consistent-sets-not-single-tables) | Publish consistent sets, not single tables | [ADR-0010](adr/0010-delta-over-hive-parquet.md) |
| [12](#12-govern-before-self-service) | Govern before self-service | [ADR-0011](adr/0011-unity-catalog-domain-schemas.md) |
| [13](#13-secure-by-design) | Secure by design (shared) | [ADR-0011](adr/0011-unity-catalog-domain-schemas.md), [Hybrid D11](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md) |
| [14](#14-run-in-parallel-before-cutting-over) | Run in parallel before cutting over | [Migration Notes](migration/) |
| [15](#15-operate-what-differentiates-manage-the-rest) | Operate what differentiates, manage the rest | [ADR-0012](adr/0012-managed-over-self-managed.md), [ADR-0013](adr/0013-phase4-orchestrator.md) |
| [16](#16-cost-follows-usage) | Cost follows usage | [ADR-0012](adr/0012-managed-over-self-managed.md) |

---

## 1. Reuse Before Replace
Each phase keeps the previous phase's components unless an ADR records why one is replaced. The extraction framework, manifests, zones, DQ rules and reconciliation logic carry through all four phases. Moving to the cloud changes storage targets and broker settings, not code. → [ADR-0001](adr/0001-yaml-driven-extraction.md), [ADR-0012](adr/0012-managed-over-self-managed.md)

## 2. Configuration Over Code
Adding a source, a quality rule or a dependency means changing configuration that is reviewed like code. Pipelines are generated from the same configs, so the orchestration can't drift from the data contracts. Configuration declares *what*, never *how*: no transformation logic in YAML. → [ADR-0001](adr/0001-yaml-driven-extraction.md), [ADR-0004](adr/0004-configurable-dq-framework.md)

## 3. Every Zone Can Be Rebuilt
Landing keeps every payload exactly as received and is never modified, so Raw, Curated and Consumption can always be rebuilt from it. In Phase 3, the event archive and topic retention give the stream the same property. A bug fix means re-running the pipeline, not asking the source for the data again. → [ADR-0002](adr/0002-four-zone-lake.md)

## 4. History Before Breadth
Phase 1 builds SCD Type 2 history, a conformed date dimension and an event fact before any new source is added. Trend, attrition and point-in-time questions depend on history. History is hard to recover later and easy to extend once its pattern exists. → [ADR-0002](adr/0002-four-zone-lake.md)

## 5. One Person, One Identifier
Cross-source questions are only answerable when every record links to the same person. Every source record resolves to one stable `person_id`, and every link records how it was made (method, score, alternatives). → [ADR-0003](adr/0003-master-person-index.md)

## 6. A Wrong Answer Is Worse Than No Answer
When the platform isn't sure, it says so instead of guessing:
- an ambiguous identity match goes to review
- a failed critical check holds the publish
- a dimension older than its source stops reconciliation

Consumers keep the last good result, and the uncertainty is visible and owned. → [ADR-0003](adr/0003-master-person-index.md), [ADR-0004](adr/0004-configurable-dq-framework.md)

## 7. Batch Is the Truth, Streaming Adds Speed
The validated daily batch stays the source of truth. Events add freshness for operational needs. Reconciliation treats the batch as the floor: it corrects stale or missing stream values and keeps values that are newer than the batch. Streaming is added *alongside* batch, never as a replacement for it. → [ADR-0006](adr/0006-lambda-over-kappa.md), [ADR-0009](adr/0009-batch-stream-reconciliation.md)

## 8. Don't Depend on Arrival Order
Events arrive late, out of order, duplicated, and on different topics. Each field remembers the version that last set it and only accepts newer ones, so any arrival order and any replay give the same result. Partitioning keeps one employee's events together. Correctness doesn't depend on it. → [ADR-0007](adr/0007-emp-id-partition-key.md)

## 9. Every Load and Every Replay Is Safe
- Watermarks, offsets and progress markers move only after success.
- Writes are upserts or MERGEs keyed on stable identifiers.
- Duplicate deliveries are absorbed.

Recovery never needs a judgment call about double counting. → [ADR-0008](adr/0008-webhook-dedup-redis.md), [Hybrid D9](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0009-idempotent-loads.md)

## 10. Check Before Consumers See It
Structural checks run inside each job. Configurable business rules run as quality gates in front of consumption. In Phase 4, changes are checked before commit and the full table after it, with a restore if those checks fail. → [ADR-0004](adr/0004-configurable-dq-framework.md), [ADR-0010](adr/0010-delta-over-hive-parquet.md)

## 11. Publish Consistent Sets, Not Single Tables
Tables that are read together are published together. A pointer records one version per table, and it moves only after every table in the set has passed its checks. Readers never see a new dimension with an old fact. → [ADR-0010](adr/0010-delta-over-hive-parquet.md)

## 12. Govern Before Self-Service
Domain teams get direct access only once these are in place:
- masking, row filters and grants
- consistent snapshots
- lineage and audit

Data products are governed views over the published snapshot, not copies. Self-service comes last in the roadmap because it depends on everything before it. → [ADR-0011](adr/0011-unity-catalog-domain-schemas.md)

## 13. Secure by Design
Sensitive data is tiered from Phase 1 and kept in the most governed place. Restricted fields are separated in profiles, kept out of logs and masked in views. Access is granted by group, through versioned SQL. → [ADR-0011](adr/0011-unity-catalog-domain-schemas.md), [Hybrid D11](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md)

## 14. Run in Parallel Before Cutting Over
- Every migration runs the old and new paths side by side until reconciliation shows they agree.
- The old path is switched off only after the exit criteria are met.
- Until then, rollback is a configuration change. → [Migration Notes](migration/), [Roadmap](roadmap.md)

## 15. Operate What Differentiates, Manage the Rest
Running brokers, databases and schedulers doesn't make an HR platform better. Use managed services where they keep the same interfaces (the Kafka protocol, the MongoDB API, Delta), and keep engineering effort for the data model, identity, quality and governance. → [ADR-0012](adr/0012-managed-over-self-managed.md), [ADR-0013](adr/0013-phase4-orchestrator.md)

## 16. Cost Follows Usage
- Compute runs only while jobs run.
- Processing is incremental, proportional to change.
- Services are consumption-based.
- Streaming is the only capacity that's always on, because operational freshness needs it.

Budgets and alerts are part of the infrastructure code. → [ADR-0012](adr/0012-managed-over-self-managed.md)

---

## Related
- [Business Context](business-context.md)
- [Non-Functional Requirements](nfrs.md)
- [Roadmap and Phase Exit Criteria](roadmap.md)
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*