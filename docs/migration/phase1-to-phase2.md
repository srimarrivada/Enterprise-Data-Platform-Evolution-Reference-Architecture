# Migration: Phase 1 → Phase 2

**Character:** Additive. Phase 2 adds sources, identity resolution and a profile store next to the Phase 1 model. No existing consumer moves, so the main risk is a wrong identity link, not a broken report.

## Starting and Ending State

| | Phase 1 | Phase 2 |
|---|---|---|
| Sources | HCM system of record | + Learning platform (REST), Survey platform (files), Recruiting module |
| Identity | HCM worker ID | One `person_id` across all sources |
| Serving | Dashboards and feeds from Consumption | + 360° profile for HR business partners |
| Quality | Gates in front of Consumption | + Per-source rules, Identity review queue |

## Reused Unchanged
- **Extraction framework:** Each new source is a new manifest; new interface types (cursor-paged REST, file drops) are added as plugins without changing the rest. → [ADR-0001](../adr/0001-yaml-driven-extraction.md)
- **Four zones and their contracts:** Publish mechanism and rebuild-from-Landing. → [ADR-0002](../adr/0002-four-zone-lake.md)
- **SCD2 dimension, date dimension, event fact and headcount aggregate.** Untouched.
- **DQ framework and pipeline generation:** New suites and configs only. → [ADR-0004](../adr/0004-configurable-dq-framework.md)

## Added, and Why

| Addition | Why | Decision |
|---|---|---|
| Learning, survey and recruiting sources | Training, engagement and hiring questions (BQ4–BQ6) | [ADR-0001](../adr/0001-yaml-driven-extraction.md) |
| Master Person Index with review queue | Cross-source questions need one identity; a wrong link is worse than none | [ADR-0003](../adr/0003-master-person-index.md) |
| 360° profile in a document store | One read per employee page (BQ7) | [ADR-0005](../adr/0005-document-store-360-profile.md) |

## Parallel Run and Comparison
1. **Phase 1 outputs stay identical:** Row counts, SCD2 history and headcount are compared before and after the new sources join the pipeline. Any difference is a defect, because nothing in Phase 1 should change.
2. **Identity accuracy is measured before any consumer uses it:** Precision and recall come from a known-answer set, with matches confirmed by hand from a sample of each source. Review rates are measured per source.
3. **New sources run in shadow mode:** They are extracted, modeled and quality-checked for a full cycle before any new dashboard or the profile reads them.

## Cutover
- **New consumers go live by domain:** Learning, then engagement, then recruiting dashboards. After that, the profile store goes live for HR business partners.
- **Rollback:** Switch off a new source's consumers; the source keeps running in shadow mode. Phase 1 consumers are unaffected throughout.

## Exit Criteria
Same as the [Roadmap](../roadmap.md#phase-2--multi-source-analytics-year-2):
- All four sources run through the unchanged framework
- No false links on the known-answer set, and unresolved records reach review
- A critical DQ failure holds the publish
- The profile is rebuilt after each batch

## Main Risks
- False identity links → [R-05](../risks.md#phase-2--multi-source-analytics), [F4](../failure-modes.md#f4-identity-resolution-is-uncertain)
- Review queue backlog → [R-06](../risks.md#phase-2--multi-source-analytics)
- Transient file sources losing data → [R-07](../risks.md#phase-2--multi-source-analytics), [F5](../failure-modes.md#f5-source-file-is-late-partial-or-duplicated)

## Related
- [Phase 1](../architecture/phase1-batch-lake.md)
- [Phase 2](../architecture/phase2-multi-source.md) 
- [Next: Phase 2 → Phase 3](phase2-to-phase3.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*