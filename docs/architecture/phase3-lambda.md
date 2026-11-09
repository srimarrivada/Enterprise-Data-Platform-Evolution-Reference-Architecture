# Phase 3: Real-Time Hybrid (Lambda)

**Year 3 - Hybrid On-premises + Cloud. Batch + Streaming**

## Goal

Make operational HR changes (hires, terminations, job, compensation and organization changes) available **within minutes** for onboarding, access provisioning and profile updates, while the **validated daily batch stays the source of truth**. Reconciliation keeps the two layers consistent.

![Phase 3 Architecture](../diagrams/svg/03-phase3-lambda.drawio.svg)

---

## Why Lambda Over Kappa

HR reporting needs an **authoritative, fully validated daily snapshot**: corrections, back-dated changes and the full set of quality rules. Events add speed, but they can be missed, duplicated, delayed or later corrected, so they can't be the only source. → [ADR-0006](../adr/0006-lambda-over-kappa.md)

| | Batch Layer (on-premises) | Speed Layer |
|---|---|---|
| Role | Complete, validated truth | Low-latency updates |
| Sources | The four Phase 2 sources | HCM business process events |
| Latency | T+1 | Seconds to minutes |
| Storage | Lake zones (unchanged from Phase 2) | Event store · profile store · event archive |
| Correctness | DQ gates, SCD2 checks | Idempotent writes, field versions, reconciliation |

---

## Event Ingestion

**Source:** HCM system pushes business process notifications as HTTPS POSTs with JSON payloads and a bearer token. It retries any delivery it doesn't see acknowledged in time.

**Webhook receiver** → [ADR-0008](../adr/0008-webhook-dedup-redis.md)
1. Authenticate the request (constant-time token compare; request size limited).
2. Validate the event: known type, IDs, effective date, and a timestamp with a time zone. Unknown extra fields are passed through.
3. **Claim** the event ID in the dedup store (Redis) with a short-lived pending marker:
   - already **done** → acknowledge as a duplicate, publish nothing
   - **in flight** (another request holds the claim) → tell the sender to retry shortly
   - dedup store **unavailable** → reject for retry (fail closed)
4. **Publish** to the event's topic, keyed by `emp_id`, and wait for the broker to confirm the write (all in-sync replicas, idempotent producer).
5. On confirmation, mark the event **done** for the dedup window and acknowledge. If the publish **fails**, **release** the claim and reject for retry, so the retry isn't mistaken for a duplicate.

A single "set if not exists" before publishing would lose events: if the publish failed, the sender's retry would look like a duplicate. Claim → publish → complete (or release) is what prevents that.

Logs carry identifiers only, never payloads, because compensation changes pass through this endpoint.

---

## Event Topics

- **One topic per event type:** `hcm.hire`, `hcm.term`, `hcm.job_change`, `hcm.comp_change`, `hcm.org_change`. Consumers can subscribe only to what they need (onboarding reads only hires).
- **Partition key = `emp_id`:** one employee's events of the **same type** stay in order and go to the same consumer.
- **Retention** longer than the longest expected consumer outage, so the stream can be replayed.
- Topics are created explicitly with their partition count and retention. Automatic topic creation is off.

### What `emp_id` Partitioning Does and Doesn't Guarantee

The broker orders messages only **within a partition of one topic**. With one topic per event type, a hire and a later job change for the same employee are in different topics, and **nothing orders them relative to each other**. Retries and replays can reorder events too.

So correctness **doesn't rely on arrival order**: the consumer is order-independent (below). A single topic for all worker events, keyed by `emp_id`, would give true per-employee ordering. It's recorded as the alternative for any consumer that needs strict sequence. → [ADR-0007](../adr/0007-emp-id-partition-key.md)

---

## Stream Consumer

A micro-batch consumer reads the topics and maintains three stores. Each batch is:
1. **Validate** each message; anything unparseable or invalid goes to a dead-letter file, and the consumer moves past it.
2. **Archive** the batch's events.
3. **Write** each event to the event store, keyed on `emp_id` + event ID.
4. **Apply** each event to the employee's profile.
5. **Commit offsets** only after both stores are written.

### Order Independence: Field Versions
- Every profile field remembers the **version** that last set it: when the change was recorded, plus the event ID to break ties.
- An update applies only if its version is newer, so the most recently recorded change wins.
- Any arrival order and any replay give the **same profile**.
- **Future-dated** changes are held as pending and applied on their effective date, so the profile never shows a change that hasn't taken effect.
- **Restricted fields** (compensation) are kept in a separate section of the profile. → [Hybrid D11](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md)

### Delivery Guarantee
The event store and profile store can't share a transaction, so delivery is **at-least-once**: offsets are committed after the stores are written, and a crash means the batch is redelivered, never skipped. The result is still correct, because **every write is idempotent**:
- An event row written again overwrites itself.
- A replayed event changes no profile field, because its version isn't newer.

### Concurrent Updates
- Events for one employee arrive on several topics, so different consumers in the group can update the same profile at the same time.
- Each profile carries a revision. A write succeeds only if the revision hasn't changed since it was read; otherwise the consumer re-reads and retries.
- No update is lost.

At HR event volumes a small micro-batch consumer is simpler to run and test than a streaming engine. When the stream needs to be joined with lake tables, or volumes grow, the same batch logic runs inside Spark Structured Streaming (`foreachBatch`). That's the path taken in Phase 4.

---

## Speed-Layer Stores

| Store | Purpose | Design |
|---|---|---|
| **Event store** (e.g., Cassandra) | Recent events per employee for the operations dashboard and for reconciliation | Partition `emp_id`; clustering on event time (newest first) and event ID; TTL from when the event happened (e.g., 90 days) |
| **Profile store** (e.g., MongoDB) | Current real-time profile per employee | One document per employee; field versions; pending future-dated changes; revision for concurrency |
| **Event archive** | Full event history for replay and audit | Append-only, partitioned by event date; deduplicated by event ID when it moves to Delta in Phase 4 |

The private link between on-premises and cloud (e.g., ExpressRoute) carries the event flow when the speed layer runs in the cloud.

---

## Reconciliation

The batch layer has an **extraction cut-off**: the end of the window its latest extraction covered. The batch is the truth for everything recorded before the cut-off, and stream events recorded after it are newer. → [ADR-0009](../adr/0009-batch-stream-reconciliation.md)

**Repair by floor:** After each batch, every profile field is raised to at least its batch value, stamped with a version that sorts just before any event at the cut-off. The field-version rule then does the rest:

| Situation | Result |
|---|---|
| Stream value recorded before the cut-off, disagrees with batch (missed event, back-dated correction) | **Corrected** to the batch value |
| Stream value recorded after the cut-off | **Kept**: it's fresher than the batch |
| Worker or field the stream never touched | **Filled** from batch, so every worker has a complete profile |
| Late event recorded before the cut-off, arriving after repair | **Ignored**: it can't undo the batch value |
| New hire after the cut-off, not in batch yet | Left alone and reported as new since the batch |

**Verify:** For every worker, "batch value plus events after the cut-off" is recomputed independently from the event store and compared with the profile. A mismatch (for example, an event stored but never applied) **fails the run**.

**Guard:** If the batch dimension was built before its latest source data, the cut-off would claim changes the batch doesn't contain, so reconciliation **refuses to run**.

The report gives counts per outcome and samples with identifiers and field names only, never values from restricted fields. A rising "stream behind batch" rate points to lost events upstream.

---

## Serving

| Store | Serves |
|---|---|
| Profile store (batch + real-time, reconciled) | HR business partner portal, IT service management onboarding |
| Event store (recent events) | Real-time operations dashboard |
| Lake zones + BI | Historical analytics (batch, daily, unchanged) |

---

## Orchestration

- The batch pipeline is unchanged from Phase 2.
- **Reconciliation** runs after each batch publish.
- The **receiver and consumer run continuously** and are monitored, not scheduled: consumer lag, error and duplicate rates, and the dead-letter volume.

---

## Design Targets

These are design parameters, not measured results:

| Parameter | Target |
|---|---|
| Broker | 3 brokers · replication factor 3 · writes acknowledged by in-sync replicas (minimum 2) · 5 topics · 7-day retention |
| Partitions | 3 per topic (caps consumer parallelism at 3; enough for HR event volume) |
| Webhook acknowledgment | Under 3 seconds |
| Dedup window | 5 minutes (pending claim shorter, longer than the publish timeout) |
| Event store | 3 nodes · replication factor 3 · 90-day TTL |
| Profile store | Replica set with majority writes |
| End-to-end latency | Source change to profile update within minutes |

---

## Failure Handling

| Failure | Designed outcome |
|---|---|
| Webhook delivered twice or concurrently | Published once; sender told to retry while in flight → [F6](../failure-modes.md#f6-webhook-delivered-twice-or-concurrently) |
| Broker or dedup store unavailable | Not acknowledged; sender retries; nothing lost → [F7](../failure-modes.md#f7-broker-or-dedup-store-unavailable) |
| Consumer down or replayed | Lag drains on restart; replay leaves state unchanged → [F8](../failure-modes.md#f8-stream-consumer-down-or-replayed) |
| Events late or out of order | Field versions keep the newest change → [F9](../failure-modes.md#f9-events-late-or-out-of-order) |
| Batch and stream disagree | Stale values corrected; verification fails on gaps → [F10](../failure-modes.md#f10-batch-and-stream-disagree) |

---

## Exit Criteria

Phase 3 is complete when:
- End-to-end event latency, from the change to the profile update, is measured and within target
- Duplicate delivery, consumer restart, offset replay and late events all leave stored state correct
- Reconciliation verifies every profile against batch plus newer events, with no mismatches
- Batch layer is unchanged and remains the source of truth

→ [Roadmap](../roadmap.md#phase-3--real-time-hybrid-year-3)  
→ Previous: [Phase 2 → Phase 3](../migration/phase2-to-phase3.md)  
→ Next: [Phase 3 → Phase 4](../migration/phase3-to-phase4.md)

---

## Related Documents

- [Phase 2: Multi-Source Analytics](phase2-multi-source.md)
- [Phase 4: Cloud-Native Lakehouse](phase4-lakehouse.md)
- [Risks: Phase 3](../risks.md#phase-3--real-time-hybrid)
- [ADR Index](../adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*