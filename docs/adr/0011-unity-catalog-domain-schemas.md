# ADR-0011: Unity Catalog Domain Schemas

- **Status:** Accepted
- **Phase:** 4

## Context
Domain teams (workforce, compensation, talent acquisition, learning, employee experience) want self-service access without copies of the data, while restricted fields and row-level access stay centrally governed and every read can be audited.

## Decision
- **One catalog with four kinds of schema:**

  | Kind | Examples | Access |
  |---|---|---|
  | Platform-managed | `curated`, `consumption`, `governance`, `ops` | Platform engineers only |
  | Published | `published` | Views pinned to the last checked snapshot; read by data products, not people |
  | Domain data products (read) | `dp_core_workforce`, `dp_comp`, `dp_talent_acquisition`, `dp_learn`, `dp_emp_experience` | `SELECT` for the domain group |
  | Domain workspaces (write) | `ws_comp`, `ws_talent_acquisition` | Domain team creates and modifies its own tables |

- **Data products are views over `published`**, which is pinned to the last checked snapshot, so every product reads one consistent, checked set of versions. Products are views, not copies. → [ADR-0010](0010-delta-over-hive-parquet.md)
- **Masks and row filters are applied inside the product views** through governance functions (e.g., `mask_email`, `mask_compensation`, `bu_filter`). Each function is evaluated for the reader through group membership:
  - email is masked unless the reader is in the restricted group
  - compensation is visible only to the compensation team
  - rows are filtered by business unit
- **Domain groups get `SELECT` on their product schema only.** Views run with their owner's access to the underlying tables, so readers need no grants on base or published tables.
- **Product metadata** (domain, owner, freshness target, PII status) is recorded as tags. Lineage and audit logs come from the catalog.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Column masks and row filters on the base tables | Stronger by default, but platform jobs access Delta tables by path, and path access is blocked on tables with masks or row filters. |
| Copy data per domain | Duplicate pipelines and drift. → [Federation over replication (Hybrid D4)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0004-federation-over-replication.md) |
| One shared schema with table grants | Ownership and boundaries are unclear; grants grow table by table. |

## Consequences
**Positive**
- Self-service without copies.
- Masking and filtering rules change in one place.
- Every product reads a consistent, checked snapshot.
- Who accessed what is auditable, and lineage runs from source to product.

**Negative / Trade-offs**
- Every new product view must apply the governance functions; a view that skips them exposes data. → [R-16](../risks.md#phase-4--cloud-native-governed-lakehouse)
- Row-filter groups are maintained per business unit.

## Guardrails
- Platform and published schemas are granted to platform engineers only.
- Grants are versioned SQL, reviewed like code, never set by hand.
- Every product view is reviewed for governance functions, and access is tested per group (masked vs unmasked values, filtered rows, denied base-table access) before release.
- New columns are classified for sensitivity before they appear in any product.

## Revisit When
- Platform jobs read and write by table name instead of path; then masks and row filters can move onto the base tables, so every access path is covered by default.
- The number of business units grows; then drive row filters from a mapping table instead of one condition per unit.

## Related
- [Phase 4: Governance](../architecture/phase4-lakehouse.md#governance-unity-catalog)
- [ADR-0010: Delta Lake over Hive/Parquet](0010-delta-over-hive-parquet.md)
- [Defense in depth (Hybrid D11)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*