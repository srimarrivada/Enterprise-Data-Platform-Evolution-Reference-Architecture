# ADR-0012: Managed Services Over Self-Managed

- **Status:** Accepted (Cassandra replacement: Proposed)
- **Phase:** 4

## Context
Phase 3 runs Kafka, Redis, Cassandra, MongoDB and HDFS-style storage itself. Patching, scaling, replication and backups for these systems are operational work that doesn't make an HR platform any better. Moving to the cloud in Phase 4 is the point where each one can be replaced, as long as the replacement keeps the existing client code.

## Decision
Replace self-managed services with managed services that speak the same wire protocol, so the change is configuration, not code.

**Decided**

| Phase 3 | Phase 4 | Change to the code |
|---|---|---|
| Kafka | **Event Hubs**, Kafka endpoint (Standard tier or higher) | Connection and SASL_SSL settings only |
| MongoDB | **Cosmos DB for MongoDB** (serverless) | Connection string only |
| Redis | **Managed Redis** (small cache tier) | Connection settings only (TLS, access key) |
| HDFS / local disk | **ADLS Gen2** + Unity Catalog volumes | Storage paths (configuration) |

**Planned**

| Phase 3 | Target | Status |
|---|---|---|
| Cassandra (90-day event store) | Delta event table, MERGE on `event_id`, queried by SQL | Proposed. Cassandra stays self-managed until the Delta table passes the stream demo with the same results. |

The Delta table fits because the event store is read for analytics and replay, not for single-key lookups at low latency. Those lookups are served by the profile store and the cache.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Keep everything self-managed on VMs | Operational burden with no benefit at this scale. |
| Event Hubs native SDK | Would replace working Kafka clients; the Kafka endpoint keeps them. |
| Hosted Kafka from a third-party vendor | Full Kafka compatibility, but a second vendor, separate billing and a separate identity setup. |
| Managed Cassandra (managed instance) | High fixed cost for a 90-day event store. |
| Cosmos DB for Apache Cassandra | Keeps the CQL code, but the data stays in a separate store, outside Unity Catalog lineage and SQL access. The Delta table puts events next to the facts they reconcile with. |

## Consequences
**Positive**
- No brokers, database servers or disks to patch, scale or back up.
- Pay for capacity provisioned, not servers; scale by changing tiers instead of adding nodes.
- Client code is unchanged; Phase 3 and Phase 4 run the same consumers.

**Negative / Trade-offs**
- Event Hubs implements the Kafka protocol but not every Kafka feature, and what's available depends on the tier (for example retention, log compaction and transactions). The design depends on none of these: delivery is at-least-once, and duplicates are removed by idempotent sinks.
- Serverless Cosmos DB has throughput and storage limits. It suits demo and moderate load, not sustained high throughput.
- Cloud lock-in, reduced but not removed by standard wire protocols (Kafka, MongoDB, Redis).
- Cassandra keeps its operational cost until the Delta event table replaces it.

## Guardrails
- Credentials come from managed identity or a secret store, never from code or committed config.
- Duplicates are always handled downstream, never by relying on producer-side guarantees.
- Every managed service is defined in infrastructure code with its tier, so limits are visible in review.

## Revisit When
- Sustained throughput exceeds serverless Cosmos DB limits; then move to provisioned or autoscale throughput.
- A required Kafka feature isn't available on the current Event Hubs tier; then upgrade the tier or move to hosted Kafka.
- The Delta event table matches Cassandra in the stream demo; then retire Cassandra and mark this ADR fully Accepted.

## Related
- [Phase 4: Cloud Lakehouse](../architecture/phase4-lakehouse.md)
- [ADR-0008: Webhook Dedup with Redis](0008-webhook-dedup-redis.md)
- [ADR-0010: Delta Lake over Hive/Parquet](0010-delta-over-hive-parquet.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*