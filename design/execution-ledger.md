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

### PR [clairvoyance#1029](https://github.com/juspay/clairvoyance/pull/1029) — workflows + walker + entry rules *(MERGED 2026-08-31, manas-narra)*

| Item | Delivered |
|---|---|
| W1 | T19 `crm_workflow` (057): draft/publish lifecycle, version audit stamp, live-plan reads per merchant |
| W2 | T20 `crm_workflow_enrollment` (058): wake_at = timer AND lease (no reaper — wake_at-push claim style), attempts++ at claim, errors PARK never exit, `enrollment_key` partial unique merchant-first, `'completed'` exit ratified (finished-without-converting; never rendered as success — late conversions are a reporting join) |
| W3 | Walker: SKIP LOCKED claim, goal re-check at fire time (`customer_has_event`), deterministic uuid5 lead ids, `queue_message` dedupe `run:node` · ADR 0010 call nodes stamp `enrollment_id` through the legacy accessor (059), once-only |
| W4 | Entry processor: per-row inside the row's savepoint, goal-cancel time-aware, `entry.key` wired (plan-level, defaults customer id), keyed-plan-refuses-missing-key |
| W5 | `NODE_TYPES` registry in nodes.py (Swaroop ruling: enforce from day one) — `is_wait()` single source + Literal pin test; `run_facts()` shared filter |
| — | Review: 1 BLOCKER (exit-context wipe) + first-node `wait_event` MAJOR (caught by pressure-testing the registry defer — the proof behind "guarantees land NOW") fixed; 233/233 green. Carried: consumer registry next record PR · repeat-entry vocabulary · keyed-flow trigger bucket (reply-match, consume-reply-on-branch, key-count cap) |

### PR [clairvoyance#1037](https://github.com/juspay/clairvoyance/pull/1037) — WhatsApp send via Meta Cloud API *(MERGED 2026-09-01, rab1prasad)*

| Item | Delivered |
|---|---|
| C1/C2 (tables) | Migration 060: T11 `crm_connector_installation` + T12 `crm_channel_binding` per canon — no vocabulary CHECKs, composite tenant-pinned FK, global `(channel, address)` non-retired unique (the second sealed merchant-first exception), T16 trigger amended for set-once binding_id. Onboarding console/API = #1038 |
| C4 | Real `send()` behind the ADAPTERS registry + MetaWhatsAppAdapter (template-only, explicit error-code classes incl. 400-carried throttles, provider-echo masking) · `CHANNELS` metadata registry (channels.py — rule-11 dictates its home) · route resolution fail-closed at every hop (paused pipe, unhealthy installation, vault via its owner's accessor with raise_errors) · gate suppression slice under its own deadline (B5 decision-log deferral RULED — see B-lane note) |
| — | #1031 tripwire honored structurally: dummy died in the PR that gated every adapter — CI rule 11 confinement + `ADAPTERS ⊆ CHANNELS` pin + lease inequality test (`batch × 2 × timeout ≤ lease`, gate + send each get one). Review: 3 MAJORs (vault-outage terminality, blocked-vs-failed vocabulary, gate probe outside deadline) + 4 MINORs — all fixed with pinning tests |

### PR [clairvoyance#1025](https://github.com/juspay/clairvoyance/pull/1025) — the push door + Shopify extractor *(MERGED 2026-09-01, cmd-err)*

| Item | Delivered |
|---|---|
| A9 | `POST /ingest/events` — the envelope door: EventIn (aware-datetime, extra=forbid, strip-before-min-length), auth as DECLARED dependency (`verify_s2s_caller`: relay wildcard JWT branch-on-claims + per-merchant token byte-compare — revocation stays live), 503-on-store-failure with dedupe-safe retry, 413 size gate as a second declared dependency, receipt `{id, duplicate}` |
| A10 | `extractors/` package (RULED: one source, one file — registry in `__init__` mirrors providers/): flat.py + shopify.py (default_address workhorse, guest-checkout fallbacks, shopify_customer_id early-frame resolution, no defaulted names) · `ingest_event` (raises) / `record_event` (fire-and-forget) fail-posture split |
| — | ADR 0022 executed: `/crm/*` mount → root (`/ingest` + `/customers` + `/workflows`), tags cleaned, producer doc moved out of docs/crm/ with zero internal names. MAJOR fixed in round: extractor handles now flow to run context (`consume_attributed_event(event, customer_id, handles)`) — the parked-Shopify-run divergence closed at the seam. Owed at shadow-live: recorded fixtures replacing synthetic Shopify payloads |

### PR [clairvoyance#1045](https://github.com/juspay/clairvoyance/pull/1045) — ingest guarantees pinned *(MERGED 2026-09-01, test-only)*

Five tests closing #1025's coverage gaps: the default_address-only park regression at the entry layer, handles-beat-payload precedence, 413 behavior + boundary, and the size gate as a declared-dependency structural pin.

### PR [clairvoyance#1046](https://github.com/juspay/clairvoyance/pull/1046) — spine consumer registry *(MERGED 2026-09-02)*

Record hears, never calls: `record/consumers.py` slot (idempotent register, ordered execution) + worker_main registration + **boundary rule 12** (record imports no subscriber, red-tested both ways). Behavior-identical; multi-pod-safe by construction (per-process registry, deterministic at import). A new spine consumer (segments, A13) is now one `register_consumer` line, zero edits in the pass.

## Open — phase 1

| Lane | Items | Notes |
|---|---|---|
| A (Identity & Record) | A4 config resolver · A8 completion (replay(), topic dispatch) · A12 remaining arms (with their lanes) · A13 transactional send consumer (now = one `register_consumer` line + the consumer, once #1046 lands) | A2+A10 SHIPPED (#1020) · **A9 SHIPPED (#1025)** — the door is open; facts flow the moment nautilus#195 relays · **consumer registry SHIPPED (#1046)** |
| B (Permission) | T07/T08 + `record_consent()` · B3 blacklist backfill · B4 `decision_log` · **B5 `may_contact()` gate** (token, tz ladder, quiet hours, caps — ADR 0018 spec done, zero code) · Shopify consent importer | **B5 definition-of-done grew (ruled 1 Sep 2026, #1037 review)**: the C4 gate slice (suppression probe) runs WITHOUT decision_log writes — ratified as part of the sealed B5 deferral because T14's writer is permission's contract. B5 must therefore ship: decision_log rows for allow AND refuse (retroactively covering the slice's verdicts), `decision_id` through `SendToken` → T16 col 18, Redis GETDEL token consumption. The seam + column + token field already wait; a B5 PR landing without them is the trigger sweep's MAJOR |
| C (Connectivity) | C1/C2 onboarding + console (#1038 — now owes the T23 template lookup, merging second) · C6 receipt walker · C7 WABA template registry (T23 sealed) · C8 connectors door | C3+C5 SHIPPED (#1031) · **C4 SHIPPED (#1037)** — WhatsApp sends for real behind the adapter + channel registries, gated by the suppression slice |
| X (external) | **X1 nautilus relay — cmd-err, #195 IN REVIEW** (with the door, #1025): reshape per the two-plane ruling — relay at webhook receipt, letter verbatim, `source=shopify`, shadow-only (cutover branch deleted; returns as per-shop `dispatch_brain` when W-lane lands) · X2 embedded signup — no owner · X3 pilot merchant + WABA — Swaroop | ADR 0022: no never-words on external surfaces — `/crm/*` → `/ingest/events`, `/customers/*` (called out on #1025) |
| W (Outreach) | W6 broadcast tables/scheduler (T17/T18) · W8 broadcast send path (**trigger: fairness lanes + per-table db split land here**) · repeat-entry vocabulary (`on_repeat` + debounce) · keyed-flow bucket (reply-match · consume-reply-on-branch · key-count cap) · W3-cadence ruling before voice takeover | W1–W5 SHIPPED (#1029) — walker + entry rules live behind `dispatch_brain` |
| U (Swaroop + Claude) | U1 loom wiring · U2 customers list (**backend live** — `GET /crm/customers` returns `CrmCustomerSummary` rows; detail GET carries full attributes) · U3 customer 360 (needs A12+B4) · U4 template manager (needs C7) | design complete (ADR 0019) |
| P0 remainder | PgBouncer (**before** A2 multiplies connections) · fail-closed voice DND · P0.4 LIKE-over-JSONB fix · reseller backfill | |

## Open — follow-ups created during the foundation build

- **Live-plan cache** (trigger: event volume makes per-event live_workflows reads
  visible in queue-lag): short-TTL in-process cache of validated definitions keyed
  (workflow_id, version) in the entry processor — plans are authored, not generated
- **Dispatcher fairness lanes** (trigger: W8/broadcast PR): the manifest claim is one
  global queue by design; before the first 10k-recipient broadcast, the claim must
  prefer transactional/utility purpose roots over marketing so a blast can never
  starve COD confirmations — ordering by purpose root + per-merchant round-robin
- **Provider package split** (trigger: the SECOND channel adapter PR; Swaroop ruling
  1 Sep 2026): one flat whatsapp.py is correct for one provider only. The second
  adapter ships `providers/<name>/` packages for BOTH (adapter.py · classify.py ·
  payload.py — spec in modules/04-connectivity.md); registry + rule 11 unchanged.
  Stacking a second flat file = MAJOR
- **One decode engine, two spec sources** (trigger: the catalog/T24 engine landing;
  Swaroop ruling 1 Sep 2026, sealed in design/event-catalog.md §Companion rulings):
  connector mappings become code-DECLARED specs run by the same generic engine as
  registered vendor rows (derive() escape hatch); Shopify's imperative extractor is
  rewritten as the first code-layer spec. Until then: consumers take the extractor's
  handles, never re-parse payloads (the #1025 pass-the-handles seam); extractors
  split one-file-per-source from #1025 onward (extractors/ package, registry in
  __init__ — design/ingest-doors.md folder table)
- **Event Catalog stack** (sealed design/event-catalog.md; RULED 1 Sep — vendor
  schemas registered at enrollment): CATALOG registry + pin tests beside EXTRACTORS ·
  catalog API (merges code + registered layers) · typed where-grammar (validator +
  entry evaluator; canon touch on entry.where; ONE migration maps→lists) ·
  seen-vs-matched counters · **T24 `crm_event_schema` (migration 061)** +
  `POST /ingest/schemas` + unregistered-topic nudge + wizard pre-fill query ·
  "Your events" console wizard (U-lane design, after runs monitor). Blocks the
  schema-driven workflow editor and push-vendor (NammaYatri-type) onboarding.
- **Spine consumer registry — SHIPPED (#1046, 2 Sep)**:
  record/consumers.py slot + worker_main registration + boundary rule 12
  (record imports no subscriber, red-tested). Trigger honesty: this fired on
  #1025 (a record-touching PR) and the review sweep MISSED it — caught in the
  post-merge growth audit; the skill now sweeps per-PR, not per-session
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

Every stage of the loop now exists in code: the door is open (#1025), the spine
drains (#1020), workflows walk (#1029, behind `dispatch_brain`), and WhatsApp
sends for real (#1031 + #1037, gated by the suppression slice). What separates
this from a merchant-visible loop is wiring, not machinery: X1's relay reshape
(facts start flowing), #1038 onboarding (a merchant gets a door + a pipe),
B4/B5 (the full permission verdict + the diary), C6 receipts + C7 templates
(the delivery story closes).

## Suggested next slices

PR-next: #1038 onboarding (owes the T23
template lookup, merging second) · X1 reshape on nautilus#195 (the door is
waiting) · B-pod starts T07/T08 + B4 (B5's diary is the sworn
definition-of-done) · recorded Shopify fixtures at shadow-live · PgBouncer
before the pod count grows again (dispatcher just added one).
