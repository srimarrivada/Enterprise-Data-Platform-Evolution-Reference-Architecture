# Failure Modes and Recovery

How the platform behaves when things go wrong in each phase, and how operators recover. Later phases keep every guarantee of the earlier ones and add their own.

**Design Goal:** A failure never causes silent data loss, a partial publish, double counting, or a consumer seeing unchecked data. Every failure is either retried automatically or leaves the platform in a known good state, with documented recovery steps.

| # | Failure | Phase | Worst Outcome (if not designed for) | Designed Outcome |
|---|---|---|---|---|
| [F1](#f1-extraction-fails-part-way) | Extraction fails part-way | 1 | Data gap or half-written files | Nothing published; same window re-extracted next run |
| [F2](#f2-critical-data-quality-check-fails) | Critical data quality check fails | 1 | Wrong numbers in dashboards | Consumption not rebuilt; previous version stays live |
| [F3](#f3-history-integrity-check-fails) | History integrity check fails | 1 | Broken point-in-time results | Dimension not published; previous version stays live |
| [F4](#f4-identity-resolution-is-uncertain) | Identity resolution is uncertain | 2 | One person's data attached to another | Uncertain matches go to review; publish blocked if review rate spikes |
| [F5](#f5-source-file-is-late-partial-or-duplicated) | Source file late, partial or duplicated | 2 | Lost, half-read or double-loaded file | Young files deferred; duplicates removed; schema problems rejected per source |
| [F6](#f6-webhook-delivered-twice-or-concurrently) | Webhook delivered twice or concurrently | 3 | Duplicate events downstream | Published once; sender told to retry while in flight |
| [F7](#f7-broker-or-dedup-store-unavailable) | Broker or dedup store unavailable | 3 | Events lost after acknowledgment | Not acknowledged; sender retries; nothing lost |
| [F8](#f8-stream-consumer-down-or-replayed) | Stream consumer down or replayed | 3 | Gaps or duplicates in stores | Lag drains on restart; replay leaves state unchanged |
| [F9](#f9-events-late-or-out-of-order) | Events late or out of order | 3 | Older change overwrites newer one | Field versions keep the newest change |
| [F10](#f10-batch-and-stream-disagree) | Batch and stream disagree | 3 | Profiles drift from the truth | Reconciliation corrects stale values; verification fails on gaps |
| [F11](#f11-bad-table-version-committed) | Bad table version committed | 4 | Bad data visible to readers | Table restored; published snapshot unchanged |
| [F12](#f12-one-table-in-a-set-fails) | One table in a set fails | 4 | New dimension read with old fact | Snapshot pointer doesn't move; readers keep the last consistent set |
| [F13](#f13-old-and-new-paths-disagree-during-migration) | Old and new paths disagree during migration | All | Cutover to a wrong platform | Cutover held; old path stays live; rollback is configuration |

---

## Phase 1: Batch Data Lake

### F1: Extraction Fails Part-Way
**Trigger:** A source API error, timeout, authentication failure or rate limit during a multi-page extraction.

**Detection:** The run log records the failed run with its window, pages and error; an alert is raised.

**Automatic Behavior**
- Transient errors (rate limits, timeouts, temporary unavailability) are retried with backoff, respecting the source's retry hints.
- Pages are written to a hidden in-progress folder; nothing is published unless every page succeeds.
- The watermark is **not advanced**, so the next run re-extracts the same window. → [ADR-0001](adr/0001-yaml-driven-extraction.md)
- Downstream jobs for that source are skipped; independent branches still run.

**Recovery Steps**
1. Classify from the run log: source outage, credentials, or a change in the source's response.
2. Fix the cause, or wait for the source to recover.
3. Re-run the pipeline. No manual backfill is needed, because the window starts at the last successful watermark.

**Prevention:** Client-side rate limiting below the source's limit; retries with exponential backoff; credentials referenced from a secret store, never written to manifests or logs.

### F2: Critical Data Quality Check Fails
**Trigger:** A business rule fails: an unexpected value, a reconciliation mismatch between zones, an orphan key, or an unusual change in volume.

**Detection:** The quality suite records each result with its measurement; critical failures alert and fail the suite.

**Automatic Behavior**
- The suite gates the Consumption tables it protects, so they are **not rebuilt**, and dashboards keep the last good version. → [ADR-0004](adr/0004-configurable-dq-framework.md)
- Warnings alert without blocking.
- Volume and distribution checks compare with the last *passing* run, so a bad run never becomes the new baseline.

**Recovery Steps**
1. Review the failing check and its sample values.
2. **Source problem:** raise it with the source owner and re-extract once it's fixed.
3. **Intended change** (e.g., a reorganization changes the business-unit mix): accept the new baseline explicitly, with a recorded reason.
4. Re-run; the gated tables rebuild once the suite passes.

**Prevention:** Every critical check has an owner; thresholds are reviewed when they fire too often or never.

### F3: History Integrity Check Fails
**Trigger:** The rebuilt SCD2 dimension has a member with no current row or two current rows, a gap or overlap in its history, a duplicate surrogate key, or no unknown-member row.

**Detection:** Structural checks run inside the job before publishing.

**Automatic Behavior:** The new dimension is not published; the previous version stays live, and so does everything built on it. → [ADR-0002](adr/0002-four-zone-lake.md)

**Recovery Steps**
1. Inspect the affected members' versions in Raw.
2. Fix the source data or the transformation, then rebuild. Every zone can be rebuilt from Landing.

**Prevention:** History logic tested against back-dated changes, corrections and future-dated changes; rebuilds that are deterministic, so a re-run gives identical results.

---

## Phase 2: Multi-Source Analytics

### F4: Identity Resolution Is Uncertain
**Trigger:** A source record matches several workers almost equally, matches only on name, or carries an ID whose name clearly disagrees.

**Detection:** The crosswalk records the status, method, score and top candidates for every record; review rates are reported per source.

**Automatic Behavior**
- Ambiguous and weak matches go to the **review queue** with their candidates. They are never auto-matched. → [ADR-0003](adr/0003-master-person-index.md)
- Records with no plausible candidate stay unmatched. That's the correct answer for external candidates who were never hired.
- If a source's review rate exceeds its limit, the crosswalk is not published.

**Recovery Steps**
1. Work the review queue: confirm or reject the top candidate.
2. If one source's review rate jumps, look for a format change (e.g., names or dates arriving differently) before tuning thresholds.

**Prevention:** Precision and recall measured against a known-answer set before any threshold change; restricted attributes never used as matching fields.

### F5: Source File Is Late, Partial or Duplicated
**Trigger:** A file drop arrives late, is still being uploaded when the extraction runs, is delivered twice, or arrives with a changed layout.

**Detection:** Arrival checks per schedule; the original file names are recorded with each Landing run; schema validation runs at the Raw zone.

**Automatic Behavior**
- Files younger than a settle time are deferred, and the watermark stops at the same cut-off, so they are picked up next run, not skipped.
- A re-read margin covers clock differences between servers; duplicates are removed by key and version in Raw.
- Rows that don't fit the layout go to a rejects table with a reason. If too many are rejected, the run fails and Raw is unchanged.

**Recovery Steps**
1. **Late file:** nothing to do; the next run picks it up.
2. **Layout change:** update the source's column mapping and rebuild Raw from Landing.

**Prevention:** Settle time or a done-marker agreed with the sender; host keys verified for file servers.

---

## Phase 3: Real-Time Hybrid

### F6: Webhook Delivered Twice or Concurrently
**Trigger:** The source retries a delivery it didn't see acknowledged in time, sometimes while the first attempt is still being processed.

**Detection:** The receiver counts accepted, duplicate and in-flight deliveries.

**Automatic Behavior**
- Each event ID is claimed before publishing and marked done only after the broker confirms the write. → [ADR-0008](adr/0008-webhook-dedup-redis.md)
- A repeat after completion is acknowledged as a duplicate and not published again.
- A repeat while the first is in flight is told to retry shortly.

**Recovery Steps:** None needed. Rising duplicate rates point to slow acknowledgment; check broker latency.

**Prevention:** Acknowledgment time kept well inside the source's timeout; consumers idempotent on event ID as a second layer.

### F7: Broker or Dedup Store Unavailable
**Trigger:** The event broker or the dedup store is down or unreachable.

**Detection:** The receiver returns "temporarily unavailable" with a retry hint; alerts fire on the error rate.

**Automatic Behavior**
- **Broker down:** The publish fails, the claim on the event is released, and the event is not acknowledged. The source's retry succeeds once the broker returns, so **nothing is lost**.
- **Dedup store down:** The receiver fails closed and rejects events for retry rather than accepting them unchecked.

**Recovery Steps**
1. Restore the broker or dedup store.
2. Confirm that the source's retries are being accepted and that the receiver's error counts fall back to zero.

**Prevention:** Broker replication with writes acknowledged by all in-sync replicas (design target); monitoring of the receiver's error rate, not just its uptime.

### F8: Stream Consumer Down or Replayed
**Trigger:** The consumer stops, crashes mid-batch, or offsets are reset to reprocess history.

**Detection:** Consumer lag per partition; the consumer's batch log.

**Automatic Behavior**
- Offsets are committed only **after** the event and profile stores are written, so a crash means the batch is redelivered, never skipped. → [ADR-0007](adr/0007-emp-id-partition-key.md)
- Every store write is idempotent: event rows are keyed on the event ID, and profile fields only change when the incoming version is newer. A replay leaves the stored state unchanged.
- Messages that can't be parsed or validated go to a dead-letter file; the consumer moves past them.

**Recovery Steps**
1. Restart the consumer; lag drains on its own.
2. Review the dead-letter file and fix the producer if events are malformed.
3. To rebuild a store, reset offsets and replay, or rebuild from the event archive.

**Prevention:** Topic retention longer than the longest expected outage; lag alerts well before retention runs out.

### F9: Events Late or Out of Order
**Trigger:** Events for one employee arrive in a different order than they happened (different topics, retries, replays), or a change is future-dated.

**Detection:** Not a failure as such: field versions make order irrelevant.

**Automatic Behavior**
- Each profile field remembers the version that last set it and only accepts newer ones, so a late older change can't overwrite a newer one.
- Future-dated changes are held as pending and applied on their effective date.

**Recovery Steps:** None needed.

**Prevention:** Events without a time zone are rejected, so versions always compare correctly.

### F10: Batch and Stream Disagree
**Trigger:** The stream missed an event, a back-dated correction reached only the batch, or an event was stored but never applied to the profile.

**Detection:** Reconciliation compares each profile field with the batch dimension; verification recomputes "batch plus newer events" from the event store and compares it with every profile.

**Automatic Behavior** → [ADR-0009](adr/0009-batch-stream-reconciliation.md)
- Stream values older than the batch cut-off are corrected to the batch value.
- Values newer than the cut-off are kept.
- Workers or fields the stream never touched are filled from batch.
- A verification mismatch fails the run.
- If the batch dimension is older than its source data, reconciliation **refuses to run** rather than overwrite real stream values.

**Recovery Steps**
1. **Verification mismatch:** find the event that is stored but not applied, then replay it or the affected window.
2. **Refused run:** rebuild the dimension, then re-run reconciliation.
3. **Rising "stream behind batch" rate:** check for lost events at the receiver or consumer.

**Prevention:** Event-store retention longer than the time between batch runs; reconciliation scheduled after every batch.

---

## Phase 4: Cloud-Native Governed Lakehouse

### F11: Bad Table Version Committed
**Trigger:** An incremental MERGE commits a table version that fails its full-table checks.

**Detection:** Checks run on the change set before commit, and on the full table after commit.

**Automatic Behavior** → [ADR-0010](adr/0010-delta-over-hive-parquet.md)
- **Check fails before commit:** nothing is written.
- **Check fails after commit:** the table is **restored** to the version before the run.
- The progress marker is not advanced, so the next run recomputes the same members.
- Readers of the published snapshot never see the bad version. Readers of the latest version might, briefly, before the restore.

**Recovery Steps**
1. Inspect the failed version through the table history (time travel).
2. Fix the cause and re-run; the recomputation is identical.

**Prevention:** Consumers read through the published snapshot, never through the latest version; table retention long enough for restore and time travel.

### F12: One Table in a Set Fails
**Trigger:** The dimension commits successfully, but the fact or the aggregate built on it fails.

**Detection:** Each table's run is logged; the snapshot is published only at the end.

**Automatic Behavior:** The published-snapshot pointer **does not move**. Readers keep the last set of versions that were built and checked together, so a new dimension is never read with an old fact. → [ADR-0010](adr/0010-delta-over-hive-parquet.md)

**Recovery Steps:** Fix the failing table and re-run. The tables that already succeeded are recomputed with no change, and the pointer moves once the whole set passes.

**Prevention:** Domain data products read only from the published views. → [ADR-0011](adr/0011-unity-catalog-domain-schemas.md)

---

## During Migration

### F13: Old and New Paths Disagree During Migration
**Trigger:** During a parallel run, the new phase's output differs from the old phase's: row counts, history, identity links or aggregates.

**Detection:** Reconciliation between the old and new paths, defined in each migration note.

**Automatic Behavior:** The cutover is held. Consumers stay on the old path, which remains fully operational. → [Migration Notes](migration/)

**Recovery Steps**
1. Classify the difference: a defect in the new path, a defect in the old path, or an intended change.
2. Fix the defect, or record the intended change and its approval.
3. Continue the parallel run until the exit criteria are met.
4. If the new path must be abandoned, rollback is a configuration change: consumers never moved.

**Prevention:** Exit criteria agreed before each phase starts; a defined length of parallel run; the old path switched off only after sign-off. → [Roadmap](roadmap.md)

---

## Operating Principles
1. **Never advance state on failure.** Watermarks, offsets, progress markers and the published snapshot move only after success.
2. **Every load and every replay is safe.** Recovery never needs a judgment call about duplicates.
3. **Fail closed for trusted data, keep operations flowing.** Gates hold reporting tables; the stream keeps running and is reconciled later.
4. **Keep the raw data.** Rebuild from Landing or the event archive instead of asking the source again.
5. **Make uncertainty visible.** Review queues, rejects, dead letters and mismatch reports have owners.

These apply [Design Principles](design-principles.md) 3, 6, 9, 10 and 11 to operations.

## Related
- [Non-Functional Requirements](nfrs.md)
- [Risks and Mitigations](risks.md)
- [Roadmap and Phase Exit Criteria](roadmap.md)
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*