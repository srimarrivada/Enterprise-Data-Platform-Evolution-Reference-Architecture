# ADR-0005: Document Store for the 360° Profile

- **Status:** Accepted
- **Phase:** 2 (real-time profile in Phase 3; managed service in Phase 4)

## Context
HR business partners need one page per employee combining employment, organization, manager, learning and engagement data. Assembling it with joins at read time is slow under interactive use, and the set of attributes differs between employees and changes over time.

## Decision
Store **one document per employee** in a document database (e.g., MongoDB; Cosmos DB for MongoDB in Phase 4), read by key.

- **Phase 2:** Document is rebuilt from Curated after each batch, so it always reflects the validated daily model.
- **Phase 3:** Same store holds the **real-time profile**:
  - Updated per event, with each field keeping the version that last set it, so arrival order doesn't matter → [ADR-0007](0007-emp-id-partition-key.md)
  - Future-dated changes held as pending until their effective date
  - Revision check on every write, so concurrent consumers can't overwrite each other's updates
  - Reconciled with the batch layer after each batch, with batch as the floor → [ADR-0009](0009-batch-stream-reconciliation.md)
- **Restricted attributes** (e.g., compensation) are kept in a **separate section** of the document and served only to authorized roles.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Wide relational table | Schema change for every new attribute; mostly empty columns. |
| Query the warehouse per page | Joins at read time; heavy load from interactive use. |
| Key-value cache | No secondary indexes or partial updates. |

## Consequences
**Positive**
- One read per employee page.
- Attributes can be added without schema changes.
- Partial updates per field, which the real-time profile depends on.

**Negative / Trade-offs**
- A second copy of the data, which must be rebuilt (Phase 2) or reconciled (Phase 3) against the model.
- Restricted sections need care in every access path.
- Field versions and pending changes add some size to each document.

## Guardrails
- The profile is never a source of truth: it's derived from the model and the event stream, and can be rebuilt from them.
- Restricted attributes are never used for matching, never written to logs, and never returned to general roles.
- Every write path uses the same field-version and revision rules.

## Revisit When
- Profile reads move to the lakehouse (e.g., a serving layer over Delta tables) and the separate store is no longer needed.
- Write volume outgrows optimistic concurrency on single documents.

## Related
- [Phase 2: 360° Employee Profile](../architecture/phase2-multi-source.md#360-employee-profile)
- [Phase 3: Stream Consumer](../architecture/phase3-lambda.md#stream-consumer)
- [ADR-0012: Managed Services over Self-managed](0012-managed-over-self-managed.md)
- [Defense in Depth (Hybrid D11)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*