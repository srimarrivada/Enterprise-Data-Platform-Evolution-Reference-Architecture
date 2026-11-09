# Business Context

The roadmap starts from who uses HR data and what they need to know. Each phase is justified by the questions it makes answerable, and each design decision traces back to one or more of the personas and questions below.

## Personas

| # | Persona | Needs | First Served In |
|---|---|---|---|
| P1 | HR leadership | Workforce trends, attrition, hiring against plan | Phase 1 |
| P2 | HR business partners | Team-level headcount and movement; one complete view of an employee | Phase 1, expanded in Phase 2 |
| P3 | Talent acquisition | Funnel conversion, time to hire, requisition health | Phase 2 |
| P4 | Learning & development | Required-training coverage, enrollment and completion | Phase 2 |
| P5 | Employee experience | Engagement results linked to workforce attributes | Phase 2 |
| P6 | Integration operations | New-hire and change events processed within minutes | Phase 3 |
| P7 | Domain analysts | Self-service analysis within their domain, without engineering tickets | Phase 4 |
| P8 | Restricted-data users and auditors | Governed access to compensation and personal data; who accessed what | Phase 1 (tiered), audited in Phase 4 |

## Representative Business Questions

| # | Question | Personas | First Answered In | Enabled By |
|---|---|---|---|---|
| BQ1 | How is headcount trending by business unit, location and job family? | P1, P2 | Phase 1 | [ADR-0002](adr/0002-four-zone-lake.md), [Kimball model (Hybrid D10)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0010-kimball-dimensional-model.md) |
| BQ2 | How many people joined and left each month, and where is attrition concentrated? | P1, P2 | Phase 1 | [Change detection (Hybrid D8)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0008-change-detection-strategy.md) |
| BQ3 | What did the organization look like on a given past date? | P1, P2 | Phase 1 | SCD Type 2 history ([Phase 1](architecture/phase1-batch-lake.md)) |
| BQ4 | Which populations are behind on required training? | P4 | Phase 2 | [ADR-0003](adr/0003-master-person-index.md) |
| BQ5 | How do engagement results relate to attrition and organization? | P5, P1 | Phase 2 | [ADR-0003](adr/0003-master-person-index.md), [ADR-0004](adr/0004-configurable-dq-framework.md) |
| BQ6 | Where do candidates drop out of the hiring funnel, and how long does hiring take? | P3 | Phase 2 | [ADR-0003](adr/0003-master-person-index.md) |
| BQ7 | What is the complete current picture of one employee? | P2 | Phase 2 | [ADR-0005](adr/0005-document-store-360-profile.md) |
| BQ8 | Has today's new hire or change reached downstream systems yet? | P6 | Phase 3 | [ADR-0008](adr/0008-webhook-dedup-redis.md), [ADR-0007](adr/0007-emp-id-partition-key.md) |
| BQ9 | Does the employee profile reflect changes made today, and still agree with the validated daily truth? | P2, P6 | Phase 3 | [ADR-0006](adr/0006-lambda-over-kappa.md), [ADR-0009](adr/0009-batch-stream-reconciliation.md) |
| BQ10 | Can a domain team answer its own questions on governed data without a request to engineering? | P7 | Phase 4 | [ADR-0011](adr/0011-unity-catalog-domain-schemas.md) |
| BQ11 | Who can see compensation and personal data, who accessed it, and where did a number come from? | P8 | Phase 4 | [ADR-0011](adr/0011-unity-catalog-domain-schemas.md), [ADR-0010](adr/0010-delta-over-hive-parquet.md) |

## How Needs Grow Across Phases

| Need | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|---|---|---|---|---|
| Freshness | Daily | Daily | Minutes for operational events; daily truth | Same, on one platform |
| Scope of a person | HCM worker record | One `person_id` across four sources | + real-time profile | + governed domain views |
| Who builds reports | Engineering | Engineering + BI | + operations dashboards | Domain teams, self-service |
| Trust | DQ gates before consumption | + identity review queue | + batch/stream reconciliation | + consistent published snapshots, lineage, audit |

## What This Means for the Architecture

- **History matters from day one:** trends, attrition and point-in-time questions need SCD Type 2 history and an event fact, so Phase 1 builds them before adding sources. → [Phase 1](architecture/phase1-batch-lake.md)
- **Adding sources must not mean new code:** four sources by Phase 2 makes config-driven extraction worthwhile. → [ADR-0001](adr/0001-yaml-driven-extraction.md)
- **Cross-source questions need one identity:** training, engagement and recruiting questions are only answerable once records link to the same person, and a wrong link is worse than none. → [ADR-0003](adr/0003-master-person-index.md)
- **Some questions can't wait a day:** onboarding and access provisioning need minutes, but reporting still needs a validated daily truth, so streaming is added *alongside* batch, not instead of it. → [ADR-0006](adr/0006-lambda-over-kappa.md)
- **Self-service needs governance first:** domain teams can only be given direct access once masking, row filters, consistent snapshots and audit are in place. → [ADR-0011](adr/0011-unity-catalog-domain-schemas.md)
- **Sensitive data has a small, governed audience in every phase.** → [Defense in Depth (Hybrid D11)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0011-defense-in-depth-security.md)

## Related
- [Non-Functional Requirements](nfrs.md)
- [Roadmap and Phase Exit Criteria](roadmap.md)
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*