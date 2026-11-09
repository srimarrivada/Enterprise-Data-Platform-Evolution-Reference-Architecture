# ADR-0007: `emp_id` Partition Key with an Order-Independent Consumer

- **Status:** Accepted
- **Phase:** 3 (unchanged on the managed broker in Phase 4)

## Context
HR events for one employee depend on each other (hire, then job change, then termination). Consumers must end with the correct current state even when events arrive late, are redelivered, or are replayed. The design routes each event type to its own topic (`hcm.hire`, `hcm.job_change`, …) so consumers can subscribe only to the types they need.

## Decision
- **Partition by `emp_id`** in every topic, so one employee's events of the same type stay in order and land on the same consumer.
- **Make the consumer order-independent.** The broker orders messages only *within* a partition of *one* topic, so per-type topics do **not** order a hire relative to a later job change. Each profile field stores the version that last set it (when the change was recorded, then `event_id` to break ties), and an update applies only if its version is newer. Any arrival order and any replay give the same result.
- **Protect concurrent updates.** Because one employee's events arrive on several topics, different consumers in the group can update the same profile at the same time. Each profile carries a revision; a write succeeds only if the revision is unchanged since it was read, otherwise the consumer re-reads and retries.
- **Commit offsets after the stores are written** (at-least-once). Every write is idempotent on `event_id`, so redelivery changes nothing.
- **Future-dated events** are held as pending and applied on their effective date.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| One topic (e.g., `hcm.worker_events`) keyed by `emp_id` | Gives true per-employee ordering and is the stronger default for new designs. Per-type topics were kept because consumers subscribe by type (e.g., onboarding reads only hires), and the consumer doesn't need ordering. Revisit if a consumer needs strict sequence. |
| Rely on per-type topics for ordering | Incorrect: there is no ordering across topics. |
| Random or round-robin partitioning | Spreads one employee's events across partitions and consumers; locality is lost and concurrent updates become the norm. |

## Consequences
**Positive**
- Correct state under reordering, redelivery and replay.
- Partitions can be added later (changing which partition each `emp_id` maps to) without breaking correctness.
- No update is lost when two consumers touch the same profile.

**Negative / Trade-offs**
- "Most recently recorded change wins" is a current-state rule. It doesn't reconstruct effective-dated history; the batch layer and reconciliation do. → [ADR-0009](0009-batch-stream-reconciliation.md)
- Field versions add some size to each profile.
- Contention on one profile causes retries; sustained contention would fail the batch, which is then redelivered.

## Guardrails
- Events without a time zone on the recorded timestamp are rejected, so versions always compare correctly.
- Timestamps are compared at one common precision in every store, so an event has the same version wherever it's read.
- Consumers are idempotent on `event_id`.
- Partition skew is monitored; with many employees and few events each, it is unlikely.

## Revisit When
- A consumer needs strict cross-type sequence; then move to a single topic.
- Profile contention becomes frequent; then route all of an employee's events to one consumer (single topic) to remove it.

## Related
- [Phase 3: What `emp_id` Partitioning Does and Doesn't Guarantee](../architecture/phase3-lambda.md#what-emp_id-partitioning-does-and-doesnt-guarantee)
- [Risks: ordering assumed but not guaranteed (R-09)](../risks.md#phase-3--real-time-hybrid)
- [ADR-0008: Two-phase webhook dedup](0008-webhook-dedup-redis.md)
- [ADR-0009: Batch/stream reconciliation](0009-batch-stream-reconciliation.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*