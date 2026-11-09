# Migration Notes

What carries forward between phases, what changes, and how each cutover is made safely. Every migration follows the same rules (see [Roadmap](../roadmap.md#rules-that-apply-across-phases)):
- Old and new paths run in parallel
- Outputs are reconciled
- Consumers move only after the exit criteria are met
- Rollback stays a configuration change until sign-off

| Migration | Character | Consumers that move | Hardest part |
|---|---|---|---|
| [Phase 1 → Phase 2](phase1-to-phase2.md) | Additive | None (new consumers only) | Proving identity links are correct |
| [Phase 2 → Phase 3](phase2-to-phase3.md) | Additive, plus one switch | Operational integrations: nightly feed → events | Proving no event is lost or misapplied |
| [Phase 3 → Phase 4](phase3-to-phase4.md) | Re-platform | All, by domain | Proving the cloud platform produces the same results |

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*
