# ADR-0008: Two-Phase Webhook Dedup with Redis

- **Status:** Accepted
- **Phase:** 3 (unchanged on the managed broker in Phase 4)

## Context
The HCM system pushes business process events as webhooks and retries any delivery it doesn't see acknowledged in time. The same event therefore arrives more than once, sometimes concurrently. The receiver must acknowledge fast, never lose an event, and avoid publishing duplicates.

## Decision
- **Two-phase dedup keyed on `event_id`:**
  1. **Claim:** set a short-lived *pending* marker only if none exists (`SET NX` with a TTL).
  2. **Publish** to the event's topic, keyed by `emp_id`, and wait for the broker to confirm the write.
  3. **Complete:** mark the key *done* for the dedup window, then acknowledge.
  4. **Release on failure:** if publishing fails, delete the claim and reject for retry, so the sender's retry is processed instead of being taken for a duplicate.
- **Responses tell the sender what to do:**
  - `202` accepted
  - `200` duplicate (already done; nothing published)
  - `409` + `Retry-After` while another request holds the claim
  - `503` + `Retry-After` when the broker or the dedup store is unavailable (fail closed)
- **Publish before acknowledging**, with writes confirmed by all in-sync replicas, a bounded wait, and an idempotent producer where the broker supports it, so an accepted event is durable.
- The pending TTL must exceed the publish timeout, so a claim can't expire in the middle of a publish.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Single `SET NX` before publishing | A failed publish makes the retry look like a duplicate, and the event is lost. |
| No receiver-side dedup; rely on downstream idempotency | Works, but duplicates consume topic capacity and complicate monitoring. Downstream idempotency is kept anyway, as a second layer. |
| Acknowledge immediately, publish asynchronously | Faster, but a crash after acknowledging loses the event unless a local durable queue is added. |
| Broker exactly-once transactions | Cover producer-to-topic only; the sender still retries at the HTTP layer. |

## Consequences
**Positive**
- No lost events: an event is acknowledged only after the broker confirms it.
- Duplicates are suppressed within the dedup window.
- Clear retry signals to the sender.

**Negative / Trade-offs**
- Delivery is **at-least-once**. An event can still be published twice if the dedup store fails after a confirmed publish, or if a retry arrives after the dedup window. Stream consumers are idempotent on `event_id`, so the result is unchanged. → [ADR-0007](0007-emp-id-partition-key.md)
- The dedup store becomes a dependency. When it's down, the receiver rejects events for retry instead of accepting them unchecked.

## Guardrails
- Logs carry identifiers only, never payloads (payloads can contain compensation).
- The bearer token is compared in constant time; request size is limited.
- The dedup store is shared by every receiver instance; a per-instance store would let duplicates through.
- The acknowledgment time stays well inside the sender's timeout, so retries stay rare.

## Revisit When
- The source supports signed payloads (e.g., HMAC); verify signatures in addition to the token.
- Volume needs horizontal scaling: the dedup store is already shared, so add receiver instances behind a load balancer.

## Related
- [Phase 3: Event Ingestion](../architecture/phase3-lambda.md#event-ingestion)
- [Failure modes F6–F7](../failure-modes.md#f6-webhook-delivered-twice-or-concurrently)
- [Risks: events lost between source and broker (R-10)](../risks.md#phase-3--real-time-hybrid)
- [ADR-0007: `emp_id` partition key with an order-independent consumer](0007-emp-id-partition-key.md)
- [Idempotent loads (Hybrid D9)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0009-idempotent-loads.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*