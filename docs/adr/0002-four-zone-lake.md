# ADR-0002: Four-Zone Lake

- **Status:** Accepted
- **Phase:** 1 (carried into Phase 4 on Delta Lake)

## Context
Source payloads arrive as nested XML, JSON and CSV. Parsing rules, business logic and models will change over time, and failures will happen. The platform must be able to **rebuild any layer from the original data**, keep source-aligned data separate from business-modeled data, and give consumers a stable layer that isn't affected by upstream changes.

## Decision
Organize the lake into **four zones**, each with an explicit contract:

| Zone | Contains | Format | Mutability | Gate before it's visible | Read by |
|---|---|---|---|---|---|
| **Landing** | Payload exactly as received | Raw XML / JSON / CSV | Immutable, append-only; one folder per run | Run complete (success marker); arrival checks for files | Platform jobs only |
| **Raw** | Source-aligned, typed, flattened data; no business logic | Parquet | Rebuilt from Landing each run | Schema matches the source's expectations; reject rate within limit | Platform jobs only |
| **Curated** | Conformed Kimball model with SCD Type 2 | Parquet + Hive (Delta in Phase 4) | Changed only through SCD logic (MERGE in Phase 4) | Structural checks (history integrity, keys); in Phase 4, checks before and after commit, with restore | Platform jobs; in Phase 4, data products through published views |
| **Consumption** | Purpose-built aggregates | Parquet (Delta in Phase 4) | Rebuilt each run | Reconciliation with Curated; DQ suites | Dashboards, APIs, feeds (in Phase 4, through published views) |

- Paths follow one convention: `<lake_root>/<zone>/<source_or_subject>/<partition>`. Landing is partitioned by source and extraction date.
- A new version becomes visible only when its gate passes; otherwise the previous version stays live. Phases 1–3 publish by swapping a fully written folder into place; Phase 4 publishes one version per table through a published snapshot. → [ADR-0010](0010-delta-over-hive-parquet.md), [Blocking vs alerting DQ (Hybrid D5)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0005-blocking-vs-alerting-dq.md)

**Mapping to Medallion terms:** Landing + Raw ≈ Bronze; Curated ≈ Silver + modeled Gold; Consumption ≈ Gold aggregates. The extra split between Landing and Raw keeps the untouched payload separate from the first typed version.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Three-zone Medallion (Bronze / Silver / Gold) | Works well for tabular sources, but with nested XML a parsing bug would corrupt the only raw copy. Keeping Landing separate preserves the original bytes. |
| Two zones (raw + modeled) | Mixes source-aligned and business logic; any model change forces re-parsing. |
| Load directly into a warehouse | No replay of original payloads; parsing and modeling are tied together. |

## Consequences
**Positive**
- Any zone can be rebuilt from the one before it, and everything can be rebuilt from Landing.
- Parsing, modeling and aggregation change independently.
- Consumers are protected from upstream changes.
- Clear ownership and quality gates at each boundary.

**Negative / Trade-offs**
- Data is stored several times, so storage cost is higher.
- More jobs to orchestrate and monitor.
- Retention has to be managed per zone.

## Guardrails
- **Landing is never modified or deleted** outside the retention policy.
- **Consumers never read Landing or Raw.**
- Every zone has an owner, a gate and a retention period.
- Zone paths come from configuration, so the storage layer can change in Phase 4. → [ADR-0001](0001-yaml-driven-extraction.md)
- Landing may contain personal data as received, so access is limited to platform engineers. → [Bronze capture (Hybrid D12)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0012-bronze-captures-transient-sources.md)

## Revisit When
- On Delta Lake (Phase 4), Raw could also move to Delta, or Landing and Raw could merge into one Bronze layer, if raw payload replay is covered by retained source files and Delta time travel.
- Streaming events (Phase 3) land in an append-only event archive partitioned by event date, outside the four zones; in Phase 4 the archive moves to a Delta table deduplicated by event ID.

## Related
- [Phase 1 architecture](../architecture/phase1-batch-lake.md)
- [End-to-End Data Flow](../architecture/end-to-end-data-flow.md)
- [ADR-0001: YAML-driven extraction](0001-yaml-driven-extraction.md)
- [ADR-0010: Delta Lake over Hive/Parquet](0010-delta-over-hive-parquet.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*