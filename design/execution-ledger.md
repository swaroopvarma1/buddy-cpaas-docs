# Execution ledger — built vs pending

The running account of what is MERGED versus what is OPEN, updated as PRs
land. The [task map](task-map.md) is the plan; this is the score. Update
protocol: when a PR merges, move its items here with the PR number and
date; when scope changes, the task map changes first.

## Merged

### CPaaS foundation — MERGED 2026-08-23 via [#1010](https://github.com/juspay/clairvoyance/pull/1010) (`9669eab`)

*Authored and reviewed as [#1016](https://github.com/juspay/clairvoyance/pull/1016); manas-narra force-pushed #1016's final tree byte-identically onto #1010's branch and merged under that number; #1016 closed. Review history (CodeRabbit round, tenancy thread, skill review) lives on #1016.*

| Item | Delivered |
|---|---|
| P0.5 | Migration CI: numbering + immutability guards; 026/034 renumbered with tracker reconciliation |
| A1 | `app/crm` scaffold to the sealed skeleton (contracts · api · schemas · logic · db/ door) |
| A5 | `/crm` router mount + admin auth + s2s verifier (dormant until A9) |
| A6 | `crm_customer` (049) + `resolve()` — INCLUDING staple-on-collision + ADR 0021 evidence ladder (pulled forward) |
| A7 | `assert_facts()` + assertion history + winners + inferred 0.5 cap (pulled forward) |
| A8 ½ | `crm_event_raw` (051) + `record_event()` with dedupe + ADR 0020 stamp passthrough |
| A14 | Voice event mirroring: lead.pushed · call.attempted · call.completed · call.inbound. Customer traffic only — `is_non_customer_lead` excludes \*_TEST, playground and DAILY_STREAM (transport-only service, client-driven payloads → numbers aren't trusted identities). HOLD_TRANSFER deliberately mirrors (reversed 23 Aug on engineer evidence: live configs dial real customers, e.g. the ride booker; a future staff-consult config gets a per-config exclusion, not a mode ban). Stamp pass-through (23 Aug): call.inbound mirrors from the created tap sequenced after the stamp (born attributed); call.attempted/completed carry `lead.customer_id` (column surfaced on the model); lead.pushed born NULL for the A10 consumer — resolution stays singular, mirrors never resolve |
| A15 | Voice identity stamping at the lead-creation choke point (hook registry — data layer imports nothing) |
| B2 | `platform_identity` (048) + suppression contracts, liveness-aware, trigger-derived |
| — | `check_crm_boundaries.py` (10 CI rules) · atomic grammar · building-modules.md · CLAUDE.md |
| — | Review-round hardening: full-scope recompute trigger · composite tenant-pinned merge FK + no-self-merge · event_raw immutability trigger · whitespace-handle fix (zero-handle law, regression-tested) · PII-free logs · inbound-mirror persistence guard · list endpoint ships `CrmCustomerSummary` (attributes jsonb never fetched for lists) · s2s verifier unit-tested · 71 pure tests total |

### PR [clairvoyance#1014](https://github.com/juspay/clairvoyance/pull/1014) — journey view, call arm *(MERGED 2026-08-25, `737be0e`)*

| Item | Delivered |
|---|---|
| A12 ½ | `crm_journey_event` view (052) — call arm only, reads LCT in place, `WHERE customer_id IS NOT NULL`, direction lowercased; 12-column canon contract |
| A12 ½ | `GET /crm/customers/{id}/journey` — keyset cursor `(started_at, id)`, record-owned (`timeline.py` logic seam); view registered in TABLE_OWNERS |

Review: skill round bounced module placement (journey→record), accessor-in-contracts, OFFSET pagination — all fixed by author. Remaining arms (message · consent · commerce `event`) land with their lanes via CREATE OR REPLACE VIEW.

### PR [clairvoyance#1020](https://github.com/juspay/clairvoyance/pull/1020) — event worker + drain-loop scaffold *(MERGED 2026-08-28)*

| Item | Delivered |
|---|---|
| A2 | Consumer runtime: `shared/worker.py` drain-loop scaffold (sleep-only-when-empty, jittered doubling backoff→5s cap, per-row isolation, interruptible waits, idle heartbeat) · `CRM_ROLE` role flags in the app lifespan (`worker_main.py` at crm root, closed ROLES dict, pool-floor ≥2 startup assert) · api-pod loops (dispatcher/scheduler/analysis) gated off worker pods |
| A10 | The processor: extractor REGISTRY (`EXTRACTORS` + `Extracted`, payload→canon fact translation, default flat shape) → `resolve()` (pass-through honored, evidence=observed always) → `assert_facts()` → per-row entry-rules slot (no-op until outreach) → one closing stamp; quarantine sets processed_at (three-state model); queue-lag logging |
| A7+ | `assert_facts` drift-only append (`_is_drift`: value/evidence/source vs latest claim) — history records evidence, not traffic |
| — | `savepoint()` enters the atomic grammar (shared/db + all doors) · accessor `conn`/logic `txn` naming codified · mirrors carry customer_name on all voice topics · review round: 1 BLOCKER (evidence inflation) + 6 MAJORs fixed; recorded-delay test pattern (zero wall-clock flake) is the new house standard |

Note: ships an intentional release rider in `numbers/rbac.py` (merchant role added to number search/buy gate — Swaroop-confirmed). Follow-up owed: add `transaction` to checker rule 5's HANDLE_CALL regex so building-modules.md's "CI catches nesting" claim is true.

### PR [clairvoyance#1031](https://github.com/juspay/clairvoyance/pull/1031) — crm_message manifest + dispatcher *(MERGED 2026-08-30, rab1prasad)*

| Item | Delivered |
|---|---|
| C3 | `crm_message` manifest per amended canon T16 (24 cols: `sending` status, `claimed_at` lease, `next_attempt_at`, total dedupe unique, single-statement claim) — migration 056 with departures recorded as the T16 amendment trail |
| C5 | Dispatcher: lease-style claim (sweep stale → claim batch), `plan_for_outcome` pure retry policy, never-raises `_dispatch_one` (send error → retryable requeue; reclaimed row → outcome discarded), ±20% jitter with floor ≥1 · dummy `send()` seam |
| — | Gate deferred to B5 by explicit scope ruling (send() is a dummy reaching nothing — zero exposure); **tripwire owed**: a test pinning send() as dummy that must die for a real adapter to land. Carried: `source_kind` dictionary with first producer · `send(send_token, message_id)` at B5 · adapter timeout < lease with first real adapter |

## Open — phase 1

| Lane | Items | Notes |
|---|---|---|
| A (Identity & Record) | A4 config resolver · **A9 ingest front door** (+A8 completion: replay(), topic dispatch) · A12 remaining arms (with their lanes) · A13 transactional send consumer | A2+A10 SHIPPED (#1020) — deploy crm-event-worker pod to drain the voice backlog; **A9 is now the whole read-path critical chain** |
| B (Permission) | T07/T08 + `record_consent()` · B3 blacklist backfill · B4 `decision_log` · **B5 `may_contact()` gate** (token, tz ladder, quiet hours, caps — ADR 0018 spec done, zero code) · Shopify consent importer | |
| C (Connectivity) | C1 installations · C2 bindings · **C4 `send()` + WhatsApp adapter** (+ gate tripwire from #1031) · C6 receipt walker · C7 WABA template registry (T23 sealed) · C8 connectors door | C3+C5 SHIPPED (#1031) |
| X (external) | **X1 nautilus relay — cmd-err, #195 IN REVIEW** (with the door, #1025): reshape per the two-plane ruling — relay at webhook receipt, letter verbatim, `source=shopify`, shadow-only (cutover branch deleted; returns as per-shop `dispatch_brain` when W-lane lands) · X2 embedded signup — no owner · X3 pilot merchant + WABA — Swaroop | ADR 0022: no never-words on external surfaces — `/crm/*` → `/ingest/events`, `/customers/*` (called out on #1025) |
| U (Swaroop + Claude) | U1 loom wiring · U2 customers list (**backend live** — `GET /crm/customers` returns `CrmCustomerSummary` rows; detail GET carries full attributes) · U3 customer 360 (needs A12+B4) · U4 template manager (needs C7) | design complete (ADR 0019) |
| P0 remainder | PgBouncer (**before** A2 multiplies connections) · fail-closed voice DND · P0.4 LIKE-over-JSONB fix · reseller backfill | |

## Open — follow-ups created during the foundation build

- **Repeat-entry debounce + refresh** (sealed position, modules/05 §Repeat entries,
  31 Aug): `on_repeat` + `debounce_minutes` entry vocabulary + one idempotent
  entry-processor UPDATE — named follow-up AFTER #1029 merges, not part of its fix round

- DB-integration test harness (CI Postgres) for resolve()/suppression race tests
- `crm_event_raw` partitioning when volume demands (documented in migration 051)
- Corpus migration into `clairvoyance/docs/crm/` + review skill into repo `.claude/` once phase-1 code stabilizes
- Rule-5 regex follow-up (#1020): add `transaction` to HANDLE_CALL so nesting is CI-caught
- **A15b — SCHEDULED (decided 23 Aug): historical LCT stamp, backfill night after
  release.** Runbook:
  1. Prereq: release deployed, migrations 048–051 applied. Release does NOT wait
     for the backlog — leads finishing pre-deploy are history lost forever; leads
     mid-flight at deploy are safe (fail-open, deduped, partial-but-truthful).
  2. Scope: **status='FINISHED' rows only** (never mid-flight — their updated_at
     feeds reaper timing), customer_id IS NULL, payload phone present. Dry-run a
     100-row sample first; watch the unparseable-phone rate.
  3. The sweep: batches ~500 with sleeps, off-peak; per row →
     `resolve(merchant, {phone}, evidence="observed", source="lct-backfill")` →
     optional `assert_facts(name, "observed")` from payload customer_name (decided:
     include it — nameless lists are useless) → stamp via
     update_lead_customer_id_query. Idempotent + kill-safe (customer_id IS NULL
     guard); resumable keyset on created_at.
  4. **HARD RULE: customers only via resolve() — never INSERT INTO crm_customer**
     (raw SQL skips E.164 normalization → CHECK violations, skips the platform
     registry, violates the boundary law the CI guard enforces).
  5. Populates exactly: crm_customer (one per distinct merchant+phone),
     platform_identity (one per distinct phone, registry only — zero suppression
     state), and the LCT stamp. NO events, NO consent, NO suppressions.
  6. Verify: stamped-by-status counts, customer/registry counts, leftover NULLs =
     unparseable phones (truthful). Straggler pass ~1 week later, same script.
  Known cosmetics (accepted): first_seen_at = backfill night (affects only future
  staple survivor ties); flat last_seen ordering for the historical block in U2.

## State in one sentence

The write path exists end-to-end for voice, the first read exists (call-arm
journey #1014), and the spine now DRAINS (#1020: event worker + processor —
deploy the crm-event-worker pod and the accumulated voice backlog attributes
itself). Nothing sends yet (no gate, no manifest, no adapter) — critical path:
A9 ∥ C1–C6 + B4/B5 → A13, with X1 as the ownerless external blocker.

## Suggested next slices

PR-next: A9 + A8 completion (the front door opens — external sources land) ·
remaining journey arms ride their lanes · C-pod starts C1–C3 in parallel · B-pod
starts T07/T08 + B4.
