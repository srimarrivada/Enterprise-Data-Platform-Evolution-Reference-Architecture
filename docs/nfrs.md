# Non-Functional Requirements

These requirements describe how well the platform must work, not what it does. Each one states how it is verified and which decision addresses it. Where a requirement tightens over the roadmap, the target is shown per phase. Targets are design targets, expressed qualitatively; an implementation would set specific values for its own context.

## Requirements

| ID | Category | Requirement | How It's Verified | Addressed By |
|---|---|---|---|---|
| NFR-01 | Freshness | Reporting data current by business start; operational events available within minutes (from Phase 3) | Freshness timestamp on every published table; end-to-end event latency measured from change to profile update | [ADR-0006](adr/0006-lambda-over-kappa.md), [ADR-0008](adr/0008-webhook-dedup-redis.md) |
| NFR-02 | Completeness | No data gaps after failures: every change in the source eventually reaches the platform | Watermark advances only after a successful publish; reconciliation of counts across zones and between batch and stream | [ADR-0001](adr/0001-yaml-driven-extraction.md), [ADR-0009](adr/0009-batch-stream-reconciliation.md) |
| NFR-03 | Reliability | Every load and every stream batch is safe to re-run or replay | Re-run and replay tests (run twice, or reset offsets): stored state unchanged | [Idempotent loads (Hybrid D9)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0009-idempotent-loads.md), [ADR-0007](adr/0007-emp-id-partition-key.md) |
| NFR-04 | Quality | Consumers never see data built on a failed critical check | Critical DQ failure skips downstream builds; published snapshot does not move | [ADR-0004](adr/0004-configurable-dq-framework.md), [ADR-0010](adr/0010-delta-over-hive-parquet.md) |
| NFR-05 | Identity accuracy | Cross-source person links: no false links; most records resolved automatically; the rest reviewed | Precision and recall measured against known matches; review-queue rate gated per source | [ADR-0003](adr/0003-master-person-index.md) |
| NFR-06 | Consistency | Readers see one consistent set of table versions; KPIs have one owning definition | Multi-table snapshot published only after all checks pass; KPI definitions held in the conformed model | [ADR-0010](adr/0010-delta-over-hive-parquet.md), [Kimball model (Hybrid D10)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0010-kimball-dimensional-model.md) |
| NFR-07 | Security | Sensitive data tiered, minimized, masked and audited | Access tests per group (masked vs unmasked, filtered rows); audit log of restricted reads | [ADR-0011](adr/0011-unity-catalog-domain-schemas.md), [Defense in depth (Hybrid D11)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md) |
| NFR-08 | Privacy & retention | Raw, history and event data kept only as long as justified; personal data deletable on request | Retention per zone and store (event TTL, Delta retention); deletion procedure covers every layer | [ADR-0002](adr/0002-four-zone-lake.md), [ADR-0010](adr/0010-delta-over-hive-parquet.md) |
| NFR-09 | Recoverability | Any zone can be rebuilt; a bad table version can be undone | Rebuild from Landing; replay from the event archive; table restore to a previous version | [ADR-0002](adr/0002-four-zone-lake.md), [ADR-0010](adr/0010-delta-over-hive-parquet.md) |
| NFR-10 | Maintainability | New sources, rules and dependencies added through configuration, not code changes | Onboarding a test source needs only a manifest; pipeline dependencies inferred from configs | [ADR-0001](adr/0001-yaml-driven-extraction.md), [ADR-0004](adr/0004-configurable-dq-framework.md) |
| NFR-11 | Operability | Every job and stream monitored, logged and restartable; failures alert | Run log for every task; consumer lag monitored; failed tasks skip only their dependents | [ADR-0013](adr/0013-phase4-orchestrator.md), [Failure Modes](failure-modes.md) |
| NFR-12 | Scalability | Workforce growth, more sources or more consumers don't require redesign | Work per run proportional to change (incremental MERGE); partitions sized for consumer parallelism | [ADR-0010](adr/0010-delta-over-hive-parquet.md), [ADR-0007](adr/0007-emp-id-partition-key.md) |
| NFR-13 | Portability | Each phase reuses the previous phase's components; cloud migration is mainly configuration | Same framework and configs across phases; storage target and broker settings are configuration | [ADR-0001](adr/0001-yaml-driven-extraction.md), [ADR-0012](adr/0012-managed-over-self-managed.md) |
| NFR-14 | Cost | Cost follows usage; no idle always-on capacity beyond what operational streaming needs | Compute only while jobs run; consumption-based services; budget alerts | [ADR-0012](adr/0012-managed-over-self-managed.md), [ADR-0013](adr/0013-phase4-orchestrator.md) |

## How Targets Change by Phase

| ID | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|---|---|---|---|---|
| NFR-01 Freshness | Daily | Daily | Minutes for events; daily truth | Same, one platform |
| NFR-02 Completeness | Watermarks + cross-zone counts | + every source | + batch/stream reconciliation | + snapshot checks |
| NFR-03 Reliability | Re-runnable batch jobs | Same | + replayable stream, idempotent sinks | + incremental MERGE re-runs |
| NFR-04 Quality | DQ gates before consumption | + per-source rules | + stream dead-letter handling | Gates protect the published snapshot |
| NFR-05 Identity | n/a (one source) | Applies | Applies | Applies |
| NFR-06 Consistency | Swap-based publish per table | Same | Batch is the floor for stream | Multi-table published snapshot |
| NFR-07 Security | Tiered zones and access | + restricted profile fields | + no payloads in logs | Masks, row filters, audit in the catalog |
| NFR-09 Recoverability | Rebuild from Landing | Same | + replay from archive and offsets | + table restore, time travel |
| NFR-12 Scalability | Full rebuilds | Full rebuilds | Partitioned consumers | Incremental, proportional to change |
| NFR-14 Cost | Fixed on-prem capacity | Same | + always-on streaming | Consumption-based |

## Related
- [Business Context](business-context.md)
- [Roadmap and Phase Exit Criteria](roadmap.md)
- [Failure Modes](failure-modes.md)
- [Risks and Mitigations](risks.md)
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*