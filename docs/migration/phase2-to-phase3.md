# Migration: Phase 2 → Phase 3

**Character:** Additive, plus one switch. The speed layer is built next to the batch layer, which stays the source of truth. The only consumers that move are operational integrations, such as onboarding and access provisioning, which switch from the nightly feed to events.

## Starting and Ending State

| | Phase 2 | Phase 3 |
|---|---|---|
| Freshness | Daily | Minutes for operational events; daily truth unchanged |
| Ingestion | Pull (APIs, files) | + Push: business process webhooks |
| Profile | Rebuilt after each batch | Updated per event; reconciled with batch after each batch |
| New stores | Document store | + Event store, event archive |
| Connectivity | On-premises | + Private link to cloud services |

## Reused Unchanged
- **Entire batch layer:** Extraction, zones, model, DQ, identity resolution. It remains the source of truth. → [ADR-0006](../adr/0006-lambda-over-kappa.md)
- **Profile store and its document structure:** Restricted fields stay in their separate section. → [ADR-0005](../adr/0005-document-store-360-profile.md)
- **Batch dashboards and feeds:** Untouched.

## Added, and Why

| Addition | Why | Decision |
|---|---|---|
| Webhook receiver with two-phase dedup | Accept pushed events fast without losing or duplicating them | [ADR-0008](../adr/0008-webhook-dedup-redis.md) |
| Event topics partitioned by `emp_id`; order-independent consumer | Correct state under redelivery, replay and reordering | [ADR-0007](../adr/0007-emp-id-partition-key.md) |
| Event store and event archive | Operations view of recent events; replay and audit | [ADR-0007](../adr/0007-emp-id-partition-key.md) |
| Batch/stream reconciliation | Batch stays the floor; stream values newer than the batch are kept | [ADR-0009](../adr/0009-batch-stream-reconciliation.md) |

## Parallel Run and Comparison
1. **Shadow mode first:** Events flow end to end into the stores, but no consumer reads them. The batch-built profile stays the one served.
2. **Every batch cycle is reconciled:**
   - The real-time profile is compared with the batch dimension
   - Verification recomputes "batch plus newer events" from the event store
   - Mismatches and the "stream behind batch" rate are tracked
3. **Failure drills before cutover:**
   - Duplicate delivery
   - Broker and dedup store outages
   - Consumer restart
   - Offset replay
   - Late and out-of-order events

   Each must leave stored state correct. → [F6–F10](../failure-modes.md#phase-3--real-time-hybrid)
4. **Latency is measured** from the source change to the profile update, against the freshness target. → [NFR-01](../nfrs.md)

## Cutover
- **The profile store switches** from batch-rebuilt to event-updated plus reconciled. HR business partners see no change apart from fresher data.
- **Operational integrations move one at a time** from the nightly feed to events. The nightly feed keeps running in parallel for at least one full cycle per integration.
- **Rollback:** Point the integration back to the nightly feed, and switch the profile back to batch rebuild. Both are configuration changes, and the batch path never stopped.

## Exit Criteria
Same as the [Roadmap](../roadmap.md#phase-3--real-time-hybrid-year-3):
- End-to-end latency measured and within target
- Duplicate, restart, replay and late-event drills leave state correct
- Reconciliation verifies every profile with no mismatches
- The batch layer is unchanged

## Main Risks
- Ordering assumed but not guaranteed → [R-09](../risks.md#phase-3--real-time-hybrid)
- Events lost between source and broker → [R-10](../risks.md#phase-3--real-time-hybrid)
- Reconciliation masking source problems → [R-13](../risks.md#phase-3--real-time-hybrid)
- Two layers to operate → [R-12](../risks.md#phase-3--real-time-hybrid)

## Related
- [Phase 2](../architecture/phase2-multi-source.md) 
- [Phase 3](../architecture/phase3-lambda.md) 
- [Previous](phase1-to-phase2.md) 
- [Next: Phase 3 → Phase 4](phase3-to-phase4.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*
