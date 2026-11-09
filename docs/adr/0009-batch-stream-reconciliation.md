# ADR-0009: Batch/Stream Reconciliation

- **Status:** Accepted
- **Phase:** 3 (runs after each lakehouse run in Phase 4)

## Context
The batch layer is complete and validated but runs nightly. The speed layer is fresh but can miss events, receive them late, or hold values that a back-dated correction later changed. Without reconciliation, either the nightly batch overwrites fresher stream values, or stream values drift from the truth.

## Decision
- **Cut-off:** End of the extraction window behind the batch dimension. Batch is the truth for everything recorded before it; events recorded after it are newer.
- **Repair by floor:** Every profile field is raised to at least its batch value, stamped with a version that sorts just before any event at the cut-off. The consumer's field-version rule then keeps newer stream values and ignores late events older than the cut-off. No event replay is needed. → [ADR-0007](0007-emp-id-partition-key.md)
- **Outcomes:**
  - Stale or missing stream values are corrected
  - Newer stream values are kept
  - Workers and fields the stream never touched are filled from batch
  - New hires after the cut-off are left alone and reported
- **Verify:** Recompute "batch value plus events after the cut-off" from the event store and compare it with each profile. Any mismatch fails the run.
- **Refuse inconsistent inputs:** If the dimension was built before its latest source data, the cut-off would claim changes the batch doesn't contain, so the run stops.
- **Report-only by default; repair is an explicit step.** Every run produces counts per outcome and samples.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Rebuild profiles from batch nightly, then replay newer events | Correct, but the profile store is briefly wrong during the rebuild and every profile is rewritten. The floor touches only fields that need it. |
| Stream only (Kappa) | No validated daily truth; corrections and lost events are never fixed. → [ADR-0006](0006-lambda-over-kappa.md) |
| Batch overwrites stream nightly | Loses changes made after the cut-off until the next batch. |

## Consequences
**Positive**
- Every worker gets a complete profile.
- Stream freshness is kept.
- Corrections and lost events are fixed, and events stored but never applied are found.
- Repair is idempotent and safe to run while the consumer is running (revision check on profiles).

**Negative / Trade-offs**
- A future-dated change recorded before the cut-off becomes visible through the next batch, not through stream promotion, so it can lag until then.
- Restricted fields (compensation) aren't in the batch dimension and aren't reconciled here.

## Guardrails
- A "stream behind batch" rate above its threshold raises a warning (it points to lost events upstream); any verification mismatch fails the run. → [R-13](../risks.md#phase-3--real-time-hybrid)
- Report samples carry identifiers and field names only, never restricted values.
- Event-store retention (e.g., 90 days) must exceed the time between batch runs.

## Revisit When
- Profiles move into the lakehouse (e.g., a Delta profile table); then reconciliation becomes a MERGE from the published dimension.
- Restricted attributes get a batch source; then they can be reconciled too.

## Related
- [Phase 3: Reconciliation](../architecture/phase3-lambda.md#reconciliation)
- [Failure mode F10: batch and stream disagree](../failure-modes.md#f10-batch-and-stream-disagree)
- [ADR-0006: Lambda over Kappa](0006-lambda-over-kappa.md)
- [ADR-0007: `emp_id` partition key with an order-independent consumer](0007-emp-id-partition-key.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*