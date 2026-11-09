# ADR-0013: Phase 4 Orchestrator Choice

- **Status:** Accepted
- **Phase:** 4

## Context
Phases 1–3 use Airflow, with tasks generated from the configs by an orchestrator-neutral pipeline plan. In Phase 4 the compute runs on Databricks, so running a separate scheduler means operating a second platform.

## Decision
- **Databricks Workflows**, deployed as code (an Asset Bundle), so no separate scheduler has to be run and versioned.
- All tasks in a run share **one job cluster**, which stops when the run ends.
- Tasks run on **dedicated (single-user) compute**, because the processing uses RDD APIs, which serverless and shared compute don't support.
- A failed step's exit code becomes a task failure, so its dependents don't run.
- One run at a time.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Keep Airflow (self-hosted or managed) | Works through the same plan, but adds a second platform to operate. Managed Airflow availability on Azure has changed over time; check it before choosing. |
| Serverless jobs | Cheaper to start, but the RDD-based steps would need rewriting on DataFrame APIs first. |

## Consequences
**Positive**
- One platform for compute and orchestration.
- Deployments are versioned and reproducible.

**Negative / Trade-offs**
- The job definition is written by hand, so it can drift from the pipeline plan.
- Dedicated clusters start slower and cost more than serverless.

## Revisit When
- Generate the job's tasks from the pipeline plan, as the Airflow pipeline already is.
- The RDD-based steps move to DataFrame APIs; then switch to serverless compute.

## Related
- [ADR-0001: YAML-driven extraction](0001-yaml-driven-extraction.md)
- [Phase 1 orchestration](../architecture/phase1-batch-lake.md#orchestration)
- [Phase 4 orchestration](../architecture/phase4-lakehouse.md#orchestration)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*