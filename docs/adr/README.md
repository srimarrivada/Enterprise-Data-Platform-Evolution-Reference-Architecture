# Architecture Decision Records

Each record captures one significant design decision: its context, the options considered, the choice made, its consequences, and the signals that would cause it to be revisited. Records follow a lightweight ADR format and describe a **generalized reference architecture**, not any specific organization's system.

Decisions shared with the [Hybrid Reference Architecture](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/README.md) (Kimball modeling, idempotent loads, content-based change detection, defense-in-depth security) link there rather than repeating them.

| ADR | Decision | Phase | Status |
|---|---|---|---|
| [0001](0001-yaml-driven-extraction.md) | YAML-driven extraction framework | 1 | Accepted |
| [0002](0002-four-zone-lake.md) | Four-zone lake | 1 | Accepted |
| [0003](0003-master-person-index.md) | Master Person Index with 4-pass resolution | 2 | Accepted |
| [0004](0004-configurable-dq-framework.md) | Configurable DQ framework with severity routing | 2 (applied from Phase 1) | Accepted |
| [0005](0005-document-store-360-profile.md) | Document store for the 360° profile | 2 | Accepted |
| [0006](0006-lambda-over-kappa.md) | Lambda over Kappa | 3 | Accepted |
| [0007](0007-emp-id-partition-key.md) | `emp_id` partition key with an order-independent consumer | 3 | Accepted |
| [0008](0008-webhook-dedup-redis.md) | Two-phase webhook dedup with Redis | 3 | Accepted |
| [0009](0009-batch-stream-reconciliation.md) | Batch/stream reconciliation | 3 | Accepted |
| [0010](0010-delta-over-hive-parquet.md) | Delta Lake over Hive/Parquet | 4 | Accepted |
| [0011](0011-unity-catalog-domain-schemas.md) | Unity Catalog domain schemas | 4 | Accepted |
| [0012](0012-managed-over-self-managed.md) | Managed services over self-managed | 4 | Accepted |
| [0013](0013-phase4-orchestrator.md) | Phase 4 orchestrator choice | 4 | Accepted |

## Status Values
- **Proposed:** Under discussion, not yet applied.
- **Accepted:** Part of the architecture.
- **Superseded:** Replaced by a later ADR. The old record stays for history and links to its replacement.

## Adding a Decision
1. Copy [0000-template.md](0000-template.md) and take the next number.
2. Fill in Context, Decision, Alternatives Considered, Consequences, Guardrails and Revisit When.
3. Add a row to this table.
4. If it replaces an existing decision, set the old one's status to *Superseded by [ADR-NNNN](NNNN-title.md)* and update its row here.

## Related
- [Design Principles](../design-principles.md)
- [Roadmap and Phase Exit Criteria](../roadmap.md)
- [Failure Modes](../failure-modes.md)
- [Risks](../risks.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*