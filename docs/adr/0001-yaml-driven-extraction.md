# ADR-0001: YAML-Driven Extraction Framework

- **Status:** Accepted
- **Phase:** 1 (reused unchanged in Phases 2–4)

## Context
Phase 1 replaces licensed batch ETL. The HCM system of record exposes SOAP APIs, custom report web services and REST report-as-a-service, and later phases add REST APIs with cursor paging and SFTP file drops. If every source gets its own hand-written script, onboarding is slow, and authentication, paging, retries and watermark handling end up implemented differently each time.

## Decision
Build **one Python extraction framework** in which every source is described by a **YAML manifest**, and the framework does the work.

- **One plugin per interface type:** `soap` (pre-built APIs and custom reports, using templated requests, with WSDL parsing where a WSDL is available), `raas`, `rest` (cursor or single-response paging) and `sftp` (file drops). A new *source* is a new manifest; only a new *interface type* needs code.
- **Common concerns live in the framework:** authentication (WS-Security, OAuth 2.0 client credentials, SSH), paging, client-side rate limiting, retries with backoff that respect the source's retry hints, watermark state, Landing writes and the run log.
- **Manifests are validated** against a strict schema before any run. Unknown fields are rejected.
- **Secrets are referenced, never stored:** manifests contain `${ENV_VAR}` placeholders, resolved at runtime. Pipeline planning works without secrets.
- **Storage is configuration:** the target path uses a lake-root variable, so Phase 4 switches from `hdfs://` to `abfss://` without code changes.
- **Pipelines are generated** from the same manifests, so orchestration can't drift from the source definitions.

```yaml
source: hcm_workers
interface: soap
operation: Get_Workers
endpoint: $${HCM_BASE_URL}/ccx/service/$${HCM_TENANT}/Human_Resources
auth:
  type: ws_security
  username: ${HCM_USERNAME}
  password: ${HCM_PASSWORD}
response_groups: [personal, employment, compensation]
extraction_mode: incremental          # incremental | full | effective_dated
pagination: { page_size: 200 }
rate_limit: { requests_per_second: 5 }
watermark:
  initial: "2000-01-01T00:00:00Z"
  overlap_seconds: 0
target_path: ${LAKE_ROOT}/landing/hcm_workers
```

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Keep the licensed ETL tool | Licensing cost is the main driver for Phase 1, and the logic stays tied to one vendor. |
| One hand-written script per source | Fast for the first source, then slow and inconsistent; common concerns are duplicated. |
| Generic managed ingestion connectors | Strong for common SaaS sources, but often limited for HCM-specific needs such as custom response groups and effective-dated extraction. Revisit as connectors mature. |
| Metadata stored in database tables | Harder to review and version than files in source control. |

## Consequences
**Positive**
- Onboarding a source is a reviewed configuration change, not a development task.
- Every source behaves the same for retries, state, logging and landing.
- Each plugin can be tested once, against a mock source, and reused.
- The same framework runs unchanged in all four phases.

**Negative / Trade-offs**
- The framework becomes an internal product that needs owners, tests and versioning.
- A new interface type still needs code.
- YAML can turn into a programming language if logic creeps into it.

## Guardrails
- **No logic in YAML:** manifests declare *what* to extract, never *how* to transform it.
- Manifest validation runs in CI; an invalid manifest never reaches a run.
- The watermark advances only after the payload is published, so a failed run re-extracts the same window. → [Idempotent loads (Hybrid D9)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0009-idempotent-loads.md)
- Landing writes are append-only and immutable. → [ADR-0002](0002-four-zone-lake.md)
- Unknown SSH host keys are rejected by default.
- Manifest changes are reviewed like code.

## Revisit When
- Managed connectors cover the needed HCM interfaces, including effective-dated extraction.
- Most sources move from pull-based extraction to events (partly addressed in Phase 3).

## Related
- [Phase 1 architecture](../architecture/phase1-batch-lake.md)
- [ADR-0002: Four-zone lake](0002-four-zone-lake.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*