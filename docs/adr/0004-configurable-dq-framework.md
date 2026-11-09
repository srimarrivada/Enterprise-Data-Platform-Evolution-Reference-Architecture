# ADR-0004: Configurable DQ Framework with Severity Routing

- **Status:** Accepted
- **Phase:** 2 (applied from the Phase 1 tables onward)

## Context
Jobs already enforce structural invariants (one current row per member, no orphan keys). Business rules are different: data owners know which statuses are valid, which counts must reconcile, and how much volume can change overnight. Those rules change more often than code, and a broken rule must stop bad numbers reaching dashboards without stopping everything else.

## Decision
- **Suites in YAML**, one per table, with nine check types: not-null, unique, referential integrity, accepted values, value range, SCD2 consistency, reconciliation (within or across tables and zones), row-count change and distribution drift.
- **Severity routing:** a `critical` failure fails the suite, so the downstream tables it gates are not rebuilt and their previous versions stay live. A `warning` sends an alert and the pipeline continues.
- **Gates are declared by the suite** (the outputs it protects), and the generated pipeline adds the dependency automatically. → [ADR-0001](0001-yaml-driven-extraction.md)
- **History-based checks** (volume, distribution) compare with the last *passing* run, so one bad run never becomes the new baseline. An intended change (e.g., a reorganization) is accepted explicitly by the data owner, with a recorded reason.
- Every result is recorded with its measurement, for trend reporting.
- Phase 2 adds a suite per new source. Identity resolution adds its own gate (review-rate limit) on top. → [ADR-0003](0003-master-person-index.md)

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Rules as code inside each job | Owners can't change rules without a release; rules mix with transformation logic. |
| Great Expectations, Soda, or platform expectations (e.g., Delta constraints, pipeline expectations) | Strong options with richer reporting. A small framework keeps the contract visible and dependency-free; the suites map almost one-to-one if a tool is adopted later. |
| Block on every failure | Distribution drift and similar signals are noisy; blocking on them causes alert fatigue. → [Hybrid R-10](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/risks.md) |

## Consequences
**Positive**
- Rules are reviewable configuration, owned by the people who know the data.
- Consumers never see a Consumption table built on data that failed a critical rule.
- Every result is measured and kept, so thresholds can be tuned from evidence.

**Negative / Trade-offs**
- In Phases 1–3, a Curated table is already published when its suite runs, so the gate protects Consumption, not Curated. Structural checks inside each job protect Curated.
- Thresholds need tuning.
- History-based checks need an explicit baseline acceptance after intended changes.

## Guardrails
- Suite changes are reviewed like code (filter predicates run as SQL against the data).
- Every critical check has an owner; every warning has a threshold review date.
- A suite that can't run (missing table, unknown column) is an **error**, not a pass.

## Revisit When
- **Phase 4:** Tables are checked before and after commit, and readers use a published snapshot (ADR-0010). Suites can then gate the snapshot pointer itself, so Curated is protected for readers too.
- A DQ tool is adopted platform-wide.

## Related
- [Phase 1: Data Quality Gates](../architecture/phase1-batch-lake.md#data-quality-gates)
- [Phase 2: Data Quality](../architecture/phase2-multi-source.md#data-quality)
- [ADR-0010: Delta Lake over Hive/Parquet](0010-delta-over-hive-parquet.md)
- [Blocking vs Alerting DQ (Hybrid D5)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0005-blocking-vs-alerting-dq.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*