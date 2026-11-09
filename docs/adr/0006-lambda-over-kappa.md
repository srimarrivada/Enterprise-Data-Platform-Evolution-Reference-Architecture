# ADR-0006: Lambda Over Kappa

- **Status:** Accepted
- **Phase:** 3 (kept in Phase 4 on one platform)

## Context
HR reporting needs a complete, validated daily truth: corrections, back-dated changes and the full set of quality rules. Operations need changes within minutes: onboarding, access provisioning, profile updates. The source offers both full extracts and business process events, and events can be missed, duplicated, delayed or later corrected.

## Decision
Run both layers, with clear roles:

- **Batch Layer:** Source of truth. It extracts everything, applies every quality rule and builds the validated daily model.
- **Speed Layer:** Adds freshness. It applies events within minutes to the profile and event stores.
- **Reconciliation:** After each batch, merges the two with **batch as the floor**. Stream values older than the batch cut-off are corrected; newer ones are kept. → [ADR-0009](0009-batch-stream-reconciliation.md)

Phase 4 keeps the split, but both layers share one platform, one storage format and one catalog.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Kappa (stream only, replay to rebuild) | Depends on the event stream being complete and on retention long enough to rebuild all state. The source's full extracts would go unused, and back-dated corrections that arrive only in extracts would be missed. |
| Batch only | Too slow for operational use cases. |
| Batch and stream side by side, without reconciliation | Each layer drifts: the stream keeps stale values after corrections, and a nightly rebuild overwrites fresher stream values. |

## Consequences
**Positive**
- Fresh where it's needed, correct everywhere.
- Each layer can fail without losing the other: if the stream stops, the daily truth still arrives; if a batch fails, the stream keeps running.
- Lost or misapplied events are found and corrected by reconciliation.

**Negative / Trade-offs**
- Two processing paths and a reconciliation step to build, run and monitor. → [R-12](../risks.md#phase-3--real-time-hybrid)
- Some logic exists in both layers (e.g., how a change maps to profile fields) and must stay consistent.

## Guardrails
- The batch layer is never changed to serve the speed layer's needs; it stays the source of truth.
- Consumers that need validated numbers read batch outputs only; the stream serves operational use cases.
- Reconciliation runs after every batch, and its correction rate is monitored. → [R-13](../risks.md#phase-3--real-time-hybrid)

## Revisit When
- The source guarantees complete, ordered, replayable events, including corrections. Then a Kappa design becomes viable.
- Corrections stop arriving outside the event stream, so the batch adds little the stream doesn't already have.

## Related
- [Phase 3 architecture](../architecture/phase3-lambda.md#why-lambda-over-kappa)
- [ADR-0007: emp_id partition key](0007-emp-id-partition-key.md)
- [ADR-0009: Batch/stream reconciliation](0009-batch-stream-reconciliation.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*