# ADR-0010: Delta Lake Over Hive/Parquet

- **Status:** Accepted
- **Phase:** 4

## Context
Phases 1–3 rebuild Curated and Consumption tables in full and publish each table by swapping a Parquet folder. That is simple and safe, but every run rewrites every row, there is no commit history or rollback, a quality gate can only protect the next zone rather than the table itself (ADR-0004), and tables read together can be published at different moments.

## Decision
- **Curated and Consumption tables move to Delta Lake:** ACID commits, time travel, and commit history with row metrics. Raw stays Parquet, rebuilt from Landing. → [ADR-0002](0002-four-zone-lake.md)
- **Dimensions: Incremental SCD2 by MERGE**
  - Members with new source versions since the last processed run are found.
  - Their **full** history is recomputed with the same SCD2 rules as Phase 1, which handles back-dated changes and corrections.
  - The result is merged on the surrogate key: update changed rows, insert new ones, delete rows the new history no longer has.
- **Facts: Incremental by dimension comparison.** Members are affected when the dimension's fact-relevant columns differ between the last processed version and the current one. Only their events are recomputed and merged.
- **Aggregates are small, so they're rebuilt** from exactly the dimension and fact versions the run produced, and checked **before** they are written.
- **Audit, commit, verify:**
  - Structural checks run on the change set before commit
  - After commit the full table is checked, and on failure the table is **restored** to its version before the run
- **Progress markers** (last processed run, dimension version) are saved only after success, so a re-run recomputes the same members with the same result.
- **Published snapshot:** After every table in a run passes, one pointer records a version per table. Readers use those versions and always see a consistent, checked set; a failed run never moves the pointer. In Unity Catalog the pointer is exposed as views in a `published` schema, each pinned to its table's published version. → [ADR-0011](0011-unity-catalog-domain-schemas.md)

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Keep Parquet with full rebuilds | Correct, but cost grows with history, and there is no audit trail, rollback or multi-table consistency. |
| Apache Iceberg | Equally strong; branches give true write-audit-publish with full reader isolation. Delta was chosen because the target platform (Databricks, Unity Catalog) is Delta-native. |
| Close-and-insert SCD2 MERGE (close the current row, insert the new one) | Fast, but mishandles back-dated changes and corrections; recomputing affected members' history handles both. |
| Change Data Feed to detect affected fact members | Cheaper at scale, but needs a table property enabled from the start and is harder to reason about after a restore. Version comparison works on any table. Revisit at scale. |

## Consequences
**Positive**
- Work proportional to change.
- Every commit auditable; rollback in one step.
- Readers of the published snapshot always see a consistent, checked set of tables.

**Negative / Trade-offs**
- Readers of the *latest* version can briefly see a bad commit before its restore. Readers of the published snapshot never do, so every consumer reads through it. → [R-15](../risks.md#phase-4--cloud-native-governed-lakehouse)
- Comparing two table versions scans both; this costs more as tables grow.
- `VACUUM` retention limits how far back restore and time travel reach.

## Guardrails
- Delta tables live under their own root and are never read as plain Parquet.
- `VACUUM` retention of at least 7 days, so restore and time travel stay possible.
- The affected-member filter is sized for HR volumes; at much larger scale, merge against a staged table of affected keys instead.
- Data layout is clustered on the member key and effective date (e.g., liquid clustering or Z-ORDER).

## Revisit When
- Tables grow enough that version comparison is slow; then enable Change Data Feed.
- Raw benefits from Delta (e.g., schema evolution, time travel on parsed data).
- Platform jobs move to reading and writing by table name; then masks and row filters can move onto the base tables. → [ADR-0011](0011-unity-catalog-domain-schemas.md)

## Related
- [Phase 4: Incremental Processing](../architecture/phase4-lakehouse.md#incremental-processing)
- [Phase 4: Published Snapshot](../architecture/phase4-lakehouse.md#published-snapshot)
- [Failure modes F11–F12](../failure-modes.md#f11-bad-table-version-committed)
- [ADR-0004: Configurable DQ framework](0004-configurable-dq-framework.md)
- [Change detection (Hybrid D8)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0008-change-detection-strategy.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*