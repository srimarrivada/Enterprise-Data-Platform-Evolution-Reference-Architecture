# ADR-0003: Master Person Index with 4-Pass Resolution

- **Status:** Accepted
- **Phase:** 2

## Context
Phase 2 adds learning, recruiting and survey sources. Each identifies people differently: some carry the employee ID, some only an email, some only a name and a start date. Analysis across sources needs one person identifier, and a wrong link (one person's training or survey data attached to another) is worse than no link.

## Decision
Resolve every source record against the worker master in four passes, and write a **crosswalk** with the source, record, `person_id`, status, method, score and top candidates:

1. **Exact employee ID**, with a name-conflict guard against ID typos.
2. **Normalized email** (trimmed, case-insensitive) that identifies exactly one worker, with the same guard.
3. **Fuzzy score:** candidates found by last name or hire date; weighted last name, first name (nicknames + similarity), hire date and job family. Auto-match only above a threshold **and** with a clear margin over the runner-up.
4. **Manual review queue** for the remaining plausible matches, with the top candidates attached.

- Missing fields score zero, so a name alone can never auto-match.
- "Unmatched" is a valid result (e.g., external candidates who were never hired).
- `person_id` is derived from the worker ID, so it survives rebuilds.
- Publishing is blocked if any source's review rate exceeds its limit, or a match points to a worker that doesn't exist.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Exact ID only | Leaves sources without IDs unlinked. |
| One combined probabilistic model (e.g., Fellegi–Sunter, as in Splink) | Better at large scale, but heavier to explain, tune and test. The pass structure keeps cheap, certain matches separate from uncertain ones. Revisit at scale. |
| Auto-match the best fuzzy candidate | Turns ambiguity into silent wrong links. |

## Consequences
**Positive**
- Measurable: precision, recall and review rate are tracked per source against a known-answer set.
- Every match is explainable: method, score and alternatives are stored.
- Wrong links are avoided by design (margin guard, review band, conflict checks).

**Negative / Trade-offs**
- The review queue needs an owner and a turnaround target.
- Thresholds and weights need tuning per source.
- Candidate lookup by last name or hire date misses people whose last name *and* hire date both differ.

## Guardrails
- No threshold or weight change without re-measuring precision and recall on the known-answer set.
- Reviewer decisions are stored as overrides and take precedence over later automatic results.
- Restricted attributes are never used as matching fields.

## Revisit When
- Volumes outgrow holding the worker master in memory; move to joins on blocking keys.
- A source's review rate stays high; add matching fields or a probabilistic model.

## Related
- [Phase 2 architecture](../architecture/phase2-multi-source.md)
- [Risks: false identity links (R-05)](../risks.md#phase-2--multi-source-analytics)
- [Fact and dimension patterns (Hybrid D13)](https://github.com/srimarrivada/Hybrid-HR-Data-Platform-Reference-Architecture/blob/main/docs/adr/0013-fact-design-patterns.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*