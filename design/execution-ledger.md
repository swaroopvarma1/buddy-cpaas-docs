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

### PRs [#1054](https://github.com/juspay/clairvoyance/pull/1054) · [#1055](https://github.com/juspay/clairvoyance/pull/1055) — channel-neutral route · connectivity structure *(MERGED 2 Sep 2026, `7058087` · `719d88f`)*

| Item | Delivered |
|---|---|
| #1054 (Claude, on Swaroop's direction) | `Channel.registers_templates` gates the T23 lookup per channel; `SendRoute.template` carries the approved ROW; queue.py's phone-channel tuple → `CHANNELS.gate_handle_kind` |
| #1055 (Rahul) — brief items 1–2 | `schemas/` package by table family (`connector` · `message` · `template`; `__init__` exports nothing; 0 package-level imports left) · `status.py` for template/installation/binding words + `test_vocabulary.py` (AST walk of `query = …`) |
| #1055 — NOT delivered at the time (DELIVERED by #1080, merged 4 Sep 2026) | item 3 `merchant_scope` dependency + router-level exception translator + `TenantScoped` + route-walk test (tenancy is still 10 hand-written `assert_merchant_access` calls) · item 4 shared test doubles (`_graph` ×2, `_mocked`, `_FakeInstallationAccessor` ×2 remain) · item 5 docs. Gaps found at audit 3 Sep: crm_message words live in `dispatch.py` with `'sending'/'queued'/'dead'` literals still in `db/queries/message.py` and the vocabulary test only sees `status.py` words (blind spot); one behaviour change rode a "pure structure" PR (`OnboardResult.address` Optional + a new onboarding refusal — defensible, now recorded) |

### The workflow rollout — phases 00–18 (call half) *(MERGED 2–3 Sep 2026, #1056–#1077, Swaroop + Claude — 17 PRs in 13 hours)*

The ordered queue lives in the repo: `docs/crm/workflow-rollout/` (README · PIPELINE · phase files · `context/reading-notes.md` = intent · `context/nits.md` · `99-backlog.md`). Merged phase by phase (the table in modules/05-outreach §"The workflow rollout as built" carries each phase's vocabulary):

| Item | Delivered |
|---|---|
| Correctness (00–04) | repeat entries with P9/P10 guards (#1058, superseding manas's #1041 — closed 2 Sep 2026) · B1/B3/B4 (#1059) · keyed admission B2 (#1060) · walker CAS on the lease P1 (#1061) · event attempts + quarantine P2, migration **062** (#1062) |
| Cart flow (06–09) | goal tiers + key + `converted_elsewhere`, migration **063** (#1063) · plan templates + runbooks in CI (#1064) · publish template check G12 (#1065) · run summary + customer journey routes G9 (#1066) |
| Version pinning (10–14) | **ADR 0023** (#1067, mirrored to decisions/0023) · T25 `crm_workflow_version`, migration **064** (#1068) · walker reads the pin (#1069) · consumer's two reads (#1070) · migrate-forward + versions list + template-retirement guard under an advisory lock, retention dropped (#1071) |
| Long boards (15–18) | `$topic`, doors, reply clearing (#1072) · facts on resume, parked runs move, `restart_on_repeat` (#1074) · `stages` ladder, loan-dropoff = one pinned board (#1075) · call outcomes into runs, `match`, `else` (#1076) · phase 19 deferred (#1077) |
| Review posture | none of the 17 passed the review skill before merge (single-author, single-session, CI green: 679 tests, boundaries/black/isort/pyrefly clean). Post-merge audit 3 Sep 2026: every law holds; findings are trails and owed hygiene — see §follow-ups |

## Open — phase 1

| Lane | Items | Notes |
|---|---|---|
| A (Identity & Record) | A4 config resolver · A8 completion (replay(), topic dispatch) · A12 remaining arms (with their lanes) · A13 transactional send consumer (now = one `register_consumer` line + the consumer, once #1046 lands) | A2+A10 SHIPPED (#1020) · **A9 SHIPPED (#1025)** — the door is open; facts flow the moment nautilus#195 relays · **consumer registry SHIPPED (#1046)** |
| B (Permission) | T07/T08 + `record_consent()` · B3 blacklist backfill · B4 `decision_log` · **B5 `may_contact()` gate** (token, tz ladder, quiet hours, caps — ADR 0018 spec done, zero code) · Shopify consent importer | **B5 definition-of-done grew (ruled 1 Sep 2026, #1037 review)**: the C4 gate slice (suppression probe) runs WITHOUT decision_log writes — ratified as part of the sealed B5 deferral because T14's writer is permission's contract. B5 must therefore ship: decision_log rows for allow AND refuse (retroactively covering the slice's verdicts), `decision_id` through `SendToken` → T16 col 18, Redis GETDEL token consumption. The seam + column + token field already wait; a B5 PR landing without them is the trigger sweep's MAJOR |
| C (Connectivity) | **#1040 (Rabi, WhatsApp webhooks) — RESHAPED 3 Sep 2026 06:54Z (head `1aee446a`) and RE-VERIFIED: every item from both review comments delivered — record's `INGRESS` slot + `GET·POST /ingest/webhooks/{provider}` in `record/api.py` (404 unknown bay, 403, 400, 413 stream cap on `MAX_LETTER_BYTES`, `ingest_event` + 503 on store failure), `connectivity/ingress.py` the one rule-11 root (generic owner→merchant over `ProviderLetter {owner_kind, owner_id, source, topic, external_id, payload, occurred_at, schema_version}`; Meta named once in `META_INGRESS`), `providers/meta/inbound.py` (three public verbs, private walk, `object`→source map, template/account letters under `TOPIC_TEMPLATE_*`/`TOPIC_ACCOUNT` with composed external_ids, payloads narrowed to one item, `schema_version` = the Graph version), `installation_for_inbound_query` (revoked excluded), `ConnectorOnboarder.resubscribe` beside `revoke` + `onboarding.resubscribe` mirroring `disconnect` + `_resubscribe_in_txn` re-stamping status/health via `update_installation_health`, `PROVIDER_ROOTS` = send/connectors/ingress only, `SubscriptionResult` in `schemas/connector.py`, topics in `topics.py`, tests split (`test_ingress_door` / `test_meta_inbound` / `test_connectivity_ingress` / `test_ingress_integration`), body rewritten. 709 tests, checker/black/isort clean. **Verdict APPROVE — awaiting Swaroop's go to post and merge.** #1052 (extractor) is stacked on this head. Earlier round — REVIEWED 2 Sep 2026, REQUEST CHANGES (single comment on the PR)**: the first cut built the Meta door in connectivity at `/connectors/webhooks/whatsapp` with a per-connector `subscribe.py` and rule 11 widened — the body was right (raw-bytes HMAC, constant-time compare, batch split, tenancy from the receiving number, 580 tests), the PLACEMENT was the drift; ruled: record's `INGRESS` is a slot filled from `app/crm/api.py`, the Meta entry is built beside its other faces (`providers/meta/inbound.py` + root `connectivity/ingress.py`), key `meta`, GET handshake on the same bay; four MAJORs ride along (template/account letters dropped at the door — the only status path; store failure answered 200 via `record_event` → `ingest_event` + 503; `resubscribe` on the onboarder port via CONNECTORS, no new roots; resubscribe must re-stamp health since usable = `{healthy}`). **Ownership ruled: #1040 IS the Meta bay (Rabi); Rahul's PR C shrinks to the `template.status` consumer.** #1052 (extractor) rebases after · **PR B = #1050 — MERGED 2 Sep 2026 (`197ccf4`, Rahul)** after two rounds (9 findings + the structure map; all delivered incl. `accounts.py`, `ProviderError` base, `topics.py`, CAS on in-place edit) · **channel-neutral route fix PR (Claude, 2 Sep, on Swaroop's direction)**: `Channel.registers_templates` gates the T23 lookup per channel (email would otherwise have been refused before its adapter), `SendRoute.template` carries the approved ROW instead of `template_language`, queue.py's parallel phone-channel tuple replaced by the registry's handle kind — WhatsApp behaviour unchanged, one observable difference (unregistered channel refused at proposal, not at the gate) · **structure PR next** (schemas/ package, status vocabularies, merchant_scope dependency, ConnectorSpec.key done, shared test doubles) — [PR B review history: T23 registry as `crm_channel_template` (five-column natural key incl. provider_account_ref — canon trail owed), `TemplateProvider` face, rejected→edit path, claim release, hsm_id, **the T23 send-time lookup in send.py DELIVERED** (`template_not_approved`, language from the registry), ZERO sync code; 484 tests. Asks before merge: CAS guard on the in-place edit (unused `get_template_for_transition`), declared `TemplateProviderError` (no bug text in 400s), `TOPIC_*` constants out of the provider package; scalability: `ConnectorSpec.key`, provider request models into `providers/<name>/`; structure map sealed on the PR; **round 2 (review 5089608057)** added three more for this PR — `resolve_template` moves into templates.py as the registry's read (before the vault decrypt), `accounts.py` (healthy door + bundle, one usable-states set; three callers today), one `ProviderError` base with faces translating GraphError — and the brief for a **structure PR right after B** (pure moves, ~20 min, #1045/#1046-style): `schemas/` package by table family, one home per table for status vocabularies + transition sets, `merchant_scope` dependency + router-level exception translator, `ConnectorSpec.key` lookup collapse, shared test doubles in conftest. **C1/C2/C8 SHIPPED — PR A = #1049 — MERGED 2 Sep 2026 (`f181e79`, Rahul)** after three review rounds (9 findings + 2 renames, all delivered: `conn` handle naming restored, `peek_binding_by_address`): the ratified shape delivered in full — providers/whatsapp/ faces, providers/meta/graph.py, CONNECTORS + pins, generic onboarding, db/ subfolders, face-precise rule 11 with red tests, /connectors routes, tenancy check in crm/auth.py, all four onboarding defects fixed and tested; 442 tests green. Asks: app secret off the query string (POST the OAuth exchange), `channel: Optional` for pipe-less connectors (Shopify next), disabled/retired pre-checks before the one-shot code is spent, route-level tests, PR body states the ADR 0007 departure. Supersedes #1038's onboarding half; #1038's template half becomes PR B. (#1038 round-3 history [review 5082190361]: owes the T23 send-time lookup, the `CONNECTORS` registry, the provider package split + `providers/meta/graph.py`, db/ subfolders, webhooks-primary with on-demand sync; plus 7 correctness MAJORs — token_expires_at never written, delete-by-name nukes every language, healer blind to deletion, rejected templates a dead end, claim never released on failure, healthy written over a failed subscription; both calls DECIDED 2 Sep — table `crm_channel_template`, auth = early X2 accepted; three-PR landing plan posted) · C6 receipt walker · C7 WABA template registry (T23 sealed) · C8 connectors door | C3+C5 SHIPPED (#1031) · **C4 SHIPPED (#1037)** — WhatsApp sends for real behind the adapter + channel registries, gated by the suppression slice |
| X (external) | **loom#320 (ADR 0022 root endpoints in the console — Claude subagent, 2 Sep) OPEN, Swaroop to merge**: `/crm/customers` → `/customers`, dev proxy bypass keyed on `Accept: application/json` (loom's own `/customers` marketing pages would otherwise be hijacked), 33 tests; heads-up: branch `assist-onboarding-in-console` carries 24 more `/crm` paths · **X1 nautilus relay — cmd-err, #195 IN REVIEW** (with the door, #1025): reshape per the two-plane ruling — relay at webhook receipt, letter verbatim, `source=shopify`, shadow-only (cutover branch deleted; returns as per-shop `dispatch_brain` when W-lane lands) · **X2 embedded signup — Rahul, ruled 2 Sep 2026: merchant-facing behind RBAC + tenancy check, an accepted early departure from ADR 0007 (admins pass, so our team still drives the pilot)** · X3 pilot merchant + WABA — Swaroop | ADR 0022: no never-words on external surfaces — `/crm/*` → `/ingest/events`, `/customers/*` (called out on #1025) |
| W (Outreach) | W6 broadcast tables/scheduler (T17/T18) · W8 broadcast send path (**trigger: fairness lanes land here**) · **rollout phase 18 message half** (delivery receipts + STOP → suppression; needs #1040 + #1052) · **phase 19 gate wiring** (needs #1021's `may_contact`; the phase file's "interim connectivity-side frequency cap" must NOT be built — caps are permission's, ADR 0018) · key-count cap · W3-cadence ruling before voice takeover · **outreach `db/` subfolders (trigger FIRED at #1068 — queries.py 814 lines, three tables — owed)** | W1–W5 SHIPPED (#1029) · **rollout phases 00–18 (call half) SHIPPED 2–3 Sep 2026 (#1058–#1077)**: repeat entries, keyed admission, CAS walker, goal tiers, plan templates, publish template check, run summary + journey, **version pinning (ADR 0023, T25)**, migrate-forward, retire guard, doors, `$topic`/`else`/`match`, facts on resume, `stages` ladder — the cart board and the loan board both run end to end |
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
- **Provider package split — trigger FIRED by #1038** (ruled 1 Sep, amended 2 Sep
  2026: the trigger is the second adapter OR the first non-send face of a provider;
  #1038 brought onboarding + templates + a Graph client as flat root files). #1038
  ships `providers/whatsapp/` (adapter · classify · payload · onboard · templates),
  `providers/meta/graph.py`, the two ports in `providers/base.py`, root
  `connectors.py` (`CONNECTORS`), and face-precise rule 11 — spec in
  modules/04-connectivity.md. Stacking vendor code at module root = MAJOR
- **db/ subfolders** (ruled 2 Sep 2026; #1038 first): at scale a module's db/ becomes
  `queries/<table>.py` · `accessors/<table>.py` · `decoders/<table>.py` (modules/00
  §1 amended); CI rule 2 admits the folder. Other modules convert at their next db/
  touch past ~2 tables or ~500 lines — outreach (394 lines, two tables) is next
- **#1038 lands as THREE PRs (ruled 2 Sep 2026, plan posted on the PR)**: **A**
  connectors — pure-move commit (providers/whatsapp/ + db/ subfolders) then the ports,
  `providers/meta/graph.py`, `connectors.py`, generic onboarding, `/connectors/*`
  routes, face-precise rule 11 + rule 2, the onboarding defects (needs the auth-phase
  answer); **B** templates — migration 061, `TemplateProvider` face, lifecycle fixes,
  **the T23 send-time lookup in send.py** (the "merges second" obligation), and
  **ZERO sync code** — the timer, the route, the seed, the sync/resume/list queries
  and the Graph list call all deleted, not fixed (amended 2 Sep: "we are merging,
  not releasing", so no bridge is needed); **C** webhooks — SPLIT 2 Sep 2026: the Meta
  ingress bay is Rabi's #1040 (reshaped per the review: `INGRESS` slot in record
  filled from `app/crm/api.py`, entry beside the Meta faces, hub.challenge on the
  bay's GET, X-Hub-Signature-256, merchant from the receiving number for
  messages/statuses and from the WABA for `template.status` / `template.category`
  / `template.quality`); Rahul's PR C is connectivity's consumer only (one
  `register_consumer` line;
  monotonic on out-of-order; carries the crashed-submit resume by natural key;
  deletion arrives as `PENDING_DELETION`/`DELETED`). B branches from A as a draft
  targeting A; one commit per PR at merge. Between B and C a submitted template
  cannot reach `approved` — correct while nothing is live. Named follow-up, not
  code: "import a WABA's existing templates" as an explicit one-shot action if a
  pilot merchant arrives with approved templates
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
  seen-vs-matched counters · **T24 `crm_event_schema` (migration 068 — 062–064 taken by the workflow rollout, 065 by buddy's #1022, 066 by buddy's #1073, 067 by #1079's T25 delete guard; #1047 renumbers to 068/069 at rebase)** +
  `POST /ingest/schemas` + unregistered-topic nudge + wizard pre-fill query ·
  "Your events" console wizard (U-lane design, after runs monitor). Blocks the
  schema-driven workflow editor and push-vendor (NammaYatri-type) onboarding.
- **Spine consumer registry — SHIPPED (#1046, 2 Sep)**:
  record/consumers.py slot + worker_main registration + boundary rule 12
  (record imports no subscriber, red-tested). Trigger honesty: this fired on
  #1025 (a record-touching PR) and the review sweep MISSED it — caught in the
  post-merge growth audit; the skill now sweeps per-PR, not per-session
- **Repeat-entry debounce + refresh — SHIPPED (#1058, 2 Sep 2026)** with the P9
  (founding event never a repeat) and P10 (`GREATEST` debounce) guards; manas's #1041
  closed 2 Sep 2026 as superseded
- **Migration numbers moved (3 Sep 2026)**: 062 attempts · 063 exit reasons · 064
  `crm_workflow_version` · 065 buddy (#1022) · 066 buddy (#1073 widget config, merged 3 Sep 12:24Z — took the number
  #1079 had used; #1079 renumbered to **067** and rebased the same day) · **067 = #1079 (T25 DELETE guard)** ·
  **#1047 (T24 + where→conditions) — rebased onto the db/ split, carries 068/069; REVIEWED 4 Sep 2026
  (REQUEST CHANGES, one comment: 069 misses list-shaped multi-door entries · `is`-family numeric
  coercion widens matches · `catalog_fields` unions by topic across sources while the door carries
  topic only — **RULING OWED: `entry.source` on the door (canon T19)** · shipped cart plans/runbook
  carry no send `variables` map and test_plan_templates passes no catalog; + 7 MINOR, 7 NIT)**.
  **Manas answered the same day (head `c110346c`): every finding addressed** — 069 rewrites both
  entry shapes (I ran the file twice against a real Postgres: object, three-door list, typed,
  no-where — correct and idempotent), the `is` family is like-with-like strict, `AmbiguousTopic`
  becomes a validation problem naming both sources, cart sends map `{"1": "customer_name"}` and
  `test_plan_templates` validates the boards against the code catalog + the loan registration,
  nudges marked known after the commit, the s2s schema door declares its verifier and the
  door-walk covers ingest + catalog routers, `catalog_laws.py` (plans.py 613 → 468), `OPS_BY_TYPE`
  spelled from predicate's families (parity pinned), ETag before the GROUP BY, re-registration
  refuses dropped paths. Re-verified by me: 856 tests, every gate, one commit, clean merge.
  Left: the PR DESCRIPTION still says migrations 061/062 and "#1041" (the commit message is
  right). **Follow-up comment posted 4 Sep (issuecomment-5540030083) with three pre-merge asks**:
  (1) `CatalogEntry.about` carried by `DecodeSpec` and passed through `engine.extract` — without it
  the second code spec (WhatsApp, #1052's rebase) cannot say its template/account letters are the
  MERCHANT's; (2) `SPEC_MODULES` registry line in `extractors/__init__.py` so `catalog.py` never
  names a source; (3) description/commit text (068/069; #1058 not #1041); (4) product-sweep
  finding, posted separately (issuecomment-5540081736): a registration under a CODE-declared
  (source, topic) is accepted and its fields leak into the validator's gather while merge and
  decode ignore the row — refuse it in `validate_registration`. Sweep verdict: product intent
  delivered clause for clause; nothing over-built; three later surfaces named (conformance
  counters, deprecated-field warning to the author + flows-list badge, the no-identity plain
  message); corpus owes the yes-no/date-time rows in the where-grammar op table. **All four asks landed at `c6c649b8` (about word through
  DecodeSpec → extract; SPEC_MODULES assembly; text; shadow-registration refusal — each with a
  test). Final skill sweep incl. db/accessor discipline: accessor shapes by signature (one
  in-atom `conn: DbTxn` for the nudge, standalone `crm_connection()` for the rest), handle names
  clean (no `txn` in db/, no `conn` in logic), every query `$n`-bound and merchant-scoped, SQL
  only in record's db/, contracts from logic. 859 tests, every gate, one commit, clean merge.
  VERDICT: APPROVE on the green build. Three NITs at the author's discretion for the next
  touch: T24 queries use `SELECT *`/`RETURNING *` where every other module spells a column
  list; the decoder re-implements `shared/decode.jsonb_value`; the samples accessor shapes
  rows (json.loads) that a decoder should.** Next free after #1047 = 070. Stale numbers on open PRs: **#1021 carries 055/056** (long taken —
  renumber after #1047 lands, 070+, and rebase; it also exposes `record_consent`/`log_decision` only, no
  `may_contact`, so B5 and phase 19 wait on it) · **#1053 carries 061** (taken by
  crm_channel_template — renumber)
- **`crm_workflow_version` DELETE guard** (ADR 0023 §5 amended): versions are never
  deleted by decision, but 064 has only the UPDATE trigger — a DELETE-refusing trigger
  is a one-file migration (next free number); with one DB role, invariants live in tables
- **Cart template `on_publish`** (repo `docs/crm/plans/cart-recovery.json` +
  `cart-recovery-fallback.json`): carries no word and so PINS; the notes' intent (§16.1) is
  `migrate` — runs are a day long and a template fix should reach every waiting run.
  One-line docs PR before the pilot publishes it
- **MERGED 3–4 Sep 2026 (Claude, on Swaroop's direction; merged before any other PR — #1079 = release `751f29ac`, #1080 = `031341e4`)**:
  **#1079** `fix(crm)` = the audit's gap closures in one PR — `Extracted.about`
  (merchant-level letters processed with a NULL customer, consumers still hear them —
  the template-status consumer's precondition), `ProviderLetter` names source /
  channel / connector_key apart + `OWNER_ENDPOINT`, migration **067** (T25 DELETE
  guard — renumbered from 066 when buddy's #1073 merged first), cart plans `on_publish: migrate`, phase-19 interim cap struck; and
  **#1080 = structure PR 2** (ONE commit — CI's one-commit rule — in two parts: connectivity's `merchant_scope` + `TranslatingRoute`
  + `TenantScoped` + route-walk test, MESSAGE_* words in `status.py` with the SQL bound
  as `$n` and the vocabulary test's fixed four-family list, `TemplateVerdict`,
  `template_reads.py` + `retire_guard.py` out of `templates.py` (648 → 538), shared
  test doubles; then outreach `db/` → `queries/ accessors/ decoders/ × {workflow,
  enrollment, version}` + `queries/tables.py`, a pure AST-driven move). Merged in that order;
  #1080 was rebased over #1079 first (one conflict in `outreach/entry.py`, both sides kept).
  Behaviour verified before the merge, not asserted: an old-vs-new router matrix (11 routes
  × 9 outcomes × 5 merchant placements — every valid request identical in code and contract
  args; drift only on malformed/foreign requests: missing merchant 422 → 400, foreign
  merchant in a POST's query → 403, query-merchant on a POST now accepted) and a
  parameter-substitution proof that the `$n`-bound message SQL renders identically to the
  old literals. Now #1047 / #1052 / #1021 rebase (#1047 renumbers to 068/069).
- **MERGED 4 Sep 2026 (release `cde86230`) — #1082 (Claude, on Swaroop's direction)**: `template_status` asks the
  question of the ACCOUNT the route will send from. CodeRabbit's review of #1080 caught the
  read accepting an approval on ANY of the merchant's accounts (in since #1065; #1080 moved
  it verbatim) and proposed "refuse unless every account is clean" — the wrong shape, because
  sends never fan out: the route is the primary active binding (partial-unique) → its
  installation → one account. The fix mirrors the send door — primary pipe → installation →
  the name's rows on that account → exactly-one rule; reasons name the sending account; no
  primary pipe is its own reason and the registry is not read. No new SQL. Trigger written:
  a send node naming `Message.binding_id` (unset today) moves the read to that binding.
- **Structure PR 2 (connectivity + outreach hygiene) — RAISED, see above; the list it closes**: #1055's undelivered items
  3–5 (`merchant_scope` dependency + router-level translator + `TenantScoped` +
  route-walk test · `tests/crm/conftest.py`/`doubles.py` · docs) · crm_message words into
  `status.py` (out of `dispatch.py`), the five SQL literals in `db/queries/message.py`
  bound as `$n`, and `test_vocabulary.py` checking a fixed all-tables word list ·
  `template_status` verdict-shaped (`{publishable, reason}`) so outreach stops comparing a
  literal across the seam · `connectivity/templates.py` 648 lines → split the retire
  guard/registration out · **outreach `db/` subfolders** (queries.py 814 / three tables —
  the ruled trigger fired at #1068) · `lock_template_exclusive_query` in the `query = …`
  shape the vocabulary test walks
- **Rollout phase 18 message half + phase 19** (repo `docs/crm/workflow-rollout/`): receipts
  (`message.status` letters → `wait_event` on the send's `message_<node>` id) and STOP →
  `record_consent`/suppression via the consumer — after #1040 (reshaped per its review)
  and #1052 (rebased) merge; phase 19 = `may_contact()` in dispatch `_gate` after #1021
  lands it — **the phase file's fallback of a connectivity-side frequency count is a
  second gate and must not be built** (ADR 0018: caps live in permission)
- **Two documentation homes (decide)**: the repo now carries `docs/crm/adr/0023`,
  `migrations.md`, `plans/`, `runbooks/`, `workflow-rollout/`. This corpus remains the
  design truth (the review skill reads it); ADR 0023 is mirrored here. Standing
  follow-up "corpus migration into clairvoyance/docs/crm/" is the moment to pick ONE home
  for ADRs — until then, an ADR written in the repo is mirrored into decisions/ the day it
  merges

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

The loop is built end to end: the door is open (#1025), the spine drains and
quarantines poison (#1020, #1062), workflows walk pinned versions with doors, tiers,
ladders and outcome branches (#1029, rollout 00–18), and WhatsApp sends for real
behind the adapter and the T23 registry (#1031, #1037, #1049, #1050). What separates
this from a merchant-visible loop is now THREE external PRs, not machinery: #1040
(the Meta ingress bay, reshaped) + #1052 (the extractor) close the delivery/reply
feedback loop and the rollout's message half; #1021 (consent ledger + `may_contact`)
unlocks B5 and phase 19; X1's relay makes facts flow. Structure PR 2 (#1080, merged 4 Sep)
has paid the hygiene the rollout deferred; #1079 closed the audit's spine and seam gaps.

## Suggested next slices

PR-next: **#1085 (Rabi — #1052 re-raised on the merged catalog) — REVIEWED 4 Sep (REQUEST
CHANGES, issuecomment-5542196555), RE-VERIFIED 7 Sep at head `14034a60`: APPROVE; MERGED 7 Sep
06:39Z as `ff0aaf81` (the approval was never posted — Swaroop merged on his own read).** All four findings fixed and proven by running the
code, not by reading the reply: all SIX WhatsApp topics now carry a code spec (the four merchant
ones `about="merchant"`, no identity fields, a recorded fixture each incl. the ban shape where
`waba_ban_state` rides as a one-element list) — I enumerated the door's own `_TOPIC_FOR_FIELD`
map and asked the catalog for every topic it can file: none uncovered, so #1084's consumer will
hear every template letter; `crm_message.reason` keeps the provider CODE and the word moved to
the READ side as `reason_label` on connectivity's contracts (dispatch names it in a COMMENT
only — no import, no call; the write test pins `"190"` on the row) — **this settles the T16
col 13 ruling in the recommended direction, no canon amendment needed**; receipts are
`about="merchant"` with no identity field, so `resolve()` no longer runs three times per send —
**this settles the receipts-attribution ruling**, matching the spine note's "processed but not
about a person". Gates on the head: 895 tests, boundaries, migration numbering, black, isort,
pyrefly 0, one commit, clean merge with release, CI green. **One gap, MINOR (coverage, not a
defect): the seam test Rabi's reply describes — one Meta envelope through the door's real
`letters()` walk, each filed letter decoded by its own spec — is NOT in the branch.** The
merchant-topic test hardcodes the four topic strings instead of deriving them from the door's
constants, so a seventh topic filed later would quarantine again with nothing failing. Ask for
it in the follow-up. **#1084 (Rahul, PR C: template webhooks → the registry) — MERGED 7 Sep 09:13Z as `a55a9f53`
(final head `fbbdd4af`: the guard-anchor line landed and was re-proven on the merged release by
injecting the reverse import — caught; 964 tests). History of the round: REBASED onto `ff0aaf81`
and FULLY REVIEWED 7 Sep at head `73713c4e` (one commit, clean merge, CI green; locally: black · isort ·
pyrefly 0 · boundaries · 69 migrations · 959 tests): APPROVE WITH ONE MAJOR TO LAND FIRST —
POSTED 7 Sep on Swaroop's go (issuecomment-5567441321), plus his STRUCTURE RULING posted on his
behalf (issuecomment-5567441554): the three root template files + `retire_guard.py` become the
`connectivity/templates/` package (lifecycle · reads · events · retire_guard), preferred as the last
change on this PR — the logic-side twin of the db/ subfolder rule, now in modules/00 §1.** **RE-VERIFIED
7 Sep at head `a625b042`** (Rahul amended silently, no reply — one commit, merge-base = release, CI
green, locally 964 tests · pyrefly 0 · boundaries · black · isort): the skew landed as
`CRM_TEMPLATE_EVENT_SKEW_SECONDS` (10) in static config, `_not_older_than` takes a skew param and
stays one clause for all three columns, the crash dial moved beside it as
`CRM_TEMPLATE_CLAIM_CRASHED_AFTER_SECONDS` pinned `>= 10 × the Graph default`, the pass test now
registers the REAL consumer beside the spy, the known-limit line is in the docstring, three
follow-ups are in `99-backlog.md`, and the package landed as `connectivity/templates/` (plural —
as-built name, corpus follows it): `lifecycle.py` · `reads.py` · `events.py` · `retire_guard.py`,
`__init__` a docstring that exports nothing, all importers moved, renames at 99–100 % similarity.
**Proven by execution against the real table** (clairvoyance_chameleon, rolled back): a letter 1.1 s
behind our own stamp APPLIES, a replay is a no-op success, 60 s behind is refused, the edge is
exactly the skew after truncation, a clockless letter applies and leaves the stamp, the tombstone
refuses. **One regression the move introduced, one line to fix**:
`test_connectivity_imports_no_outreach` walks `Path(templates.__file__).parent` — before the move
that was the whole connectivity module, after it only the four template files, so the ONLY guard
against the connectivity→outreach cycle (checker rule 4 permits `.contracts` imports, so nothing
else catches it) silently shrank; an injected `import app.crm.outreach.contracts` in `send.py`
is caught on release and passes at head. Fix: anchor the walk on `app.crm.connectivity`'s own
`__file__`. Doc nit: the N7 backlog line still says `connectivity/template_events.py`. **Posted 7 Sep as
issuecomment-5567846074 (Swaroop's go): mergeable once that line lands.** Twelve files, 1,693 lines: `template_events.py` (the consumer,
4-arg signature, topic-filtered on `TEMPLATE_TOPICS`, dispatches `connector_for_source` →
`spec.templates.normalize_event` → neutral `ProviderTemplateState`), `ConnectorSpec.source` +
`connector_for_source`, three guarded CAS applies (status/category/quality, each on its own
stamped column, `_not_retired` tombstone on all three, NULL-clock and NULL-column branches,
second-truncation, `COALESCE($n, column)` never `now()`), the crashed-submit resume (STATUS
letters only; account-free probe first; `CLAIM_CRASHED_AFTER_SECONDS = 300` age gate keeps a
healthy submit out; account DERIVED from the merchant's installations on that connector counted
UNFILTERED — exactly one = known, else decline), and `record_in_place_edit` widened to
`status IN (destination, expected)` so the consumer winning the pending race no longer strands
an edit. The extractor and the encryption change are GONE (extractor superseded by #1085's
spec; encryption not re-raised anywhere yet). Rabi's 4 Sep findings: B1/B2/B3/M1/M2/M3/m1/m2/m3
all closed in the diff and pinned by tests; M4 (heartbeat on the hot path) closed by REMOVAL
of the ingress heartbeat; M5/M6 closed by #1085; B4/M7 moot (encryption left the PR). **Meta
coupling: none in the generic layer** — the consumer, `templates.py`, `db/`, `status.py`,
`topics.py` name no provider; an SMS-DLT provider is `providers/<name>/` (templates face with
`normalize_event`, inbound face), one `CONNECTORS` line (`source`, `channel="sms"`), one
`CHANNELS` line, one record spec + `SPEC_MODULES` line, one INGRESS registration — zero edits
to this PR's files. **The one MAJOR (new, not in Rabi's list): our OWN transitions stamp
`status_updated_at = now()`** (`record_submission`, `record_in_place_edit`) **on the same column
the provider-letter guard orders by** — Meta stamps `entry.time` in whole seconds at the
DECISION, our commit lands ~100–500 ms later, and whenever that crosses a second boundary the
guard `date_trunc('second', column) <= occurred_at` REFUSES the provider's own decision letter.
Exposure: an instant approval (the PR's own docstring: "Meta can approve an AUTHENTICATION
template in seconds") after `edit()` — the WhatsApp face hardcodes the edit response as
`pending`, so the refused APPROVED letter is the only source of truth, and the row sits
'pending' forever with no sync to heal it (the PR removed every heal by design). Cheapest
correct fix: a skew tolerance on the guard (`<= $param + make_interval(secs => SKEW)`, SKEW
≈ 10 s in config, one clause for all three columns), which absorbs the RTT and still refuses a
minutes-late redelivery; the principled fix is a provider-clock column (migration 070, one
column per guarded topic). MINOR: `CLAIM_CRASHED_AFTER_SECONDS` is a tuning dial in db/queries
(config + an inequality pin against the Graph timeout; the test pins `>= 60` only); hygiene —
`db/queries/template.py` 353 → 686 lines, per-table so no ruled sub-split, the webhook-path
section is the seam, OWED at the next builder; pre-existing and now WIDENED — a 'deleted' row
(retire, or Meta's own DELETED letter) blocks re-creating that name+language forever
(`create_draft` refuses a non-draft on the natural key, `edit` refuses 'deleted'): follow-up,
partial unique excluding 'deleted' or create reopens the tombstone. Corpus after merge:
modules/04 §Templates trail — the resume is age-gated and account-DERIVED (declines for a
two-account merchant); N7 landed; the consumer is live
**#1057 (Sharifajahan — the `action` square + the Shopify connector's action face via nautilus;
Manas reviewed 4 Sep + 7 Sep, both REQUEST CHANGES) — FULL REVIEW 7 Sep at head `a4b38e6c`
(one commit, merges clean on `3d57931a`; gates on the merged tree: 993 tests · pyrefly 0 ·
boundaries · black · isort): REQUEST CHANGES — POSTED 7 Sep on Swaroop's go as issuecomment-5569557158.** **RE-VERIFIED 7 Sep at head `1e98661a`** (silent amend, one commit,
clean on release `0b5d4ab5`; gates on the merged tree: 1004 tests · pyrefly 0 · boundaries · black ·
isort · 71 migrations): all three MAJORs and every MINOR are in the DIFF — `outreach/nodes/` package
(`__init__` assembles `NODE_TYPES` from `wait · wait_event · send · call · action`, `spec.py`,
`context.py`; the old 596-line file deleted); `order_id` is an arg on all three Shopify models, the
face reads no context, `execute` passes `{run_id, node_id}` only, placeholders judged against the
send-side allow-list at publish (`catalog_laws` + `placeholder_names`); `needs_installation` gone
from port, root and faces — `installation is None` refuses, `_door` reads `external_account_id`
only; `test_config_bounds.py` pins action-timeout < walker-lease AND send-timeout < dispatch-stale;
`NoTemplates` deleted, `ConnectorSpec.templates` Optional with guards in lifecycle + events; empty
signing secret refuses locally; boundary transport tests parametrized; `.env.example` documented;
no test anchors a walk on `nodes.__file__`. **Verdict: MERGEABLE codewise — MERGED by Swaroop 7 Sep 11:37Z as `f98b7ae1` (squash).** The one MINOR left was raised as its own PR on his ask — **#1101 (`refactor/crm-nodes-init-registry-only`, one commit `5c76eb06` on `f98b7ae1`)**: `nodes/__init__` exports `NODE_TYPES` · `NodeSpec` · `is_wait` only, pinned by a test proven red with an injected re-export; `context.is_bookkeeping(key)` is the one definition entry.py and run_facts share; ten importers moved to full paths; docs say so; 1005 tests. Was:
`nodes/__init__.py` re-exports 21 names (three of them private `_BOOKKEEPING_*`/`_REQUEST_ID_KEYS`)
so the twelve importers stay unchanged — the sanctioned assembly `__init__` exports the REGISTRY
(`NODE_TYPES`, `NodeSpec`, `is_wait`, `NodeParked`), everything else by full path (rules/01: an
`__init__` exports nothing; the 132-line re-export-hub scar). A sed, before or right after merge.
NIT: the PR body is STILL CodeRabbit's summary of a different PR (fourth ask). nautilus follow-up
(400/404/422 on lookup failures) still owed on the other repo. Release moved under it again:
Bhumika's `071_credentials_merchant_id.sql` took 071 → next free migration = **072**.** Right shape in
connectivity (root → registry → face → confined transport; args as contract; two failure
classes; sign-the-bytes; usable-door policy bound as a parameter; NULL credential as the
migration switch). Three MAJORs: (1) the Shopify face reads OUTREACH's run context
(`_order_id(context["facts"])` with a three-key heuristic) while its own port docstring says
args are `{order_id, tags}` and the root's docstring says context "never reaches the provider
as data" — a seam spill; `order_id` is an ARG the plan names as `{id}`, the node resolves it;
(2) `needs_installation = False` is a bypass flag on the one door the verb rests on, kept after
the PR built the door itself (Manas's MAJOR, still open); (3) `outreach/nodes.py` 431 → 596
lines with the fifth word — Swaroop's ruling: the `nodes/` package, one file per word,
registry assembled in `__init__`. MINORs: no inequality pin `CRM_ACTION_TIMEOUT_SECONDS <
CRM_WALKER_LEASE_SECONDS`; placeholders not judged against the fact allow-list at publish;
`perform`'s normalised facts are computed and DISCARDED by `execute_action`; `NoTemplates` is a
59-line refusal stub because `ConnectorSpec.templates` is not Optional; an empty signing secret
omits the header instead of refusing locally; nautilus#201 (merged 10:05Z) answers 401 on a bad
signature and made notes idempotent but still throws → 500 on permanent lookup failures
(unknown shop, bad order id), so those spend three walker attempts before parking — a nautilus
follow-up; `onboarding.py` 502 → 516 (already over the line before this PR; hygiene owed, not this PR's). Corpus now carries both rulings (modules/04
fourth verb, modules/05 `action` word + `nodes/` package, how-it-scales, ADR 0009 trail, T11
col 6 trail); #1078 A/03 (`http` node with an author URL) is superseded and Swaroop's docs PR
should say so. Manas's blocker (the door writing a decoded `reply`) closed by REMOVAL — the
meta bay is no longer in the diff (#1085 derives `reply`).

**#1108 (Rahul — the `list` catalog type + `item_format`; Shopify 11 → 42 declared fields, four
cart phrasings, `fulfillment_state`, `orders/updated`; context ceiling to live config) — Manas
REQUEST CHANGES 8 Sep (G4 unruled; dark door) → APPROVE 9 Sep at `d9abf502`. MY REVIEW 9 Sep at
the same head (one commit, clean on `ce556ce9`; gates on the merged tree: 1044 tests · pyrefly 0 ·
boundaries · 72 migrations · black · isort): APPROVE WITH NITS — mergeable; POSTED 9 Sep on Swaroop's go as issuecomment-5599259612.** **RE-VERIFIED 9 Sep at head `437e3ae5`** (silent amend, one commit, clean on release
`46e7e17b`; merged tree: 1050 tests · pyrefly 0 · boundaries · 72 migrations · black · isort): both MINORs
closed IN THE DIFF and proven by running the getter — a stored 100 is CLAMPED to 256 with a warning,
a stored 0 the same, 500 consecutive reads cost 0 Redis GETs (60 s process-local window, cold-cache
fixture in `test_context_ceiling.py`; the floor is DUPLICATED in `app/core` because core may not
import crm, and a test pins the two equal); the `order_currency` docstring now argues for the key the
code prefers. Unasked scope trim: the `items_full` phrasing is gone (three cart phrasings; `line_total`
still attached per line for vendor formats). Nits left as nits: G4 backlog row, PR body,
`orders/updated` dependency line. **MERGEABLE.** Release moved under it: Batman's assist refactors +
`072_crm_customer_attributes_gin.sql` → next free migration = **073**.
Laws hold: decode at processing time over the verbatim letter; `list` has zero ops (pinned to the
predicate families) so the matcher never sees an array; `item_format` stored only where it means
something (old T24 rows byte-identical); config through the resolver; registration refuses the
four new mistakes. Two MINORs, both on the one dial this PR made live: (1)
`CRM_CONTEXT_VALUE_MAX_CHARS` is read from Redis per consumed letter (1–2 GETs each; `get_config`
has no in-process cache) — the FIRST live read on the spine's per-row path; fine at 1/s, 2–4k
GETs/s at stage 2 for a value that changes once a year → the `CRM_SCHEMA_CACHE_SECONDS` shape (a
process-local TTL) or read once per pass; (2) the getter has no floor, and the engine's join budget
`VARIABLE_MAX_CHARS = 256` is static: set the live value below 256 and every full-length joined
list is DROPPED from context — the exact "parks two modules away" the engine docstring guards
against; set it to 0 and every run is born with empty context → clamp to `>= 256` in the getter
(the validated-sibling shape, `META_GRAPH_TIMEOUT_SECONDS`). NITs: `order_currency` docstring
argues for preferring `presentment_currency` while the code (correctly — REST `line_items.price`
is SHOP currency) prefers `currency`, and Manas's approval repeats the docstring's claim; the
99-backlog G4 row still says "needs a ruling" in the PR that implements the ruling; `orders/updated`
advertised while nautilus#205 is OPEN (a plan on it sees 0 forever, and our own action-square tag
will wake it once it flows) — state the dependency in the body; `catalog.py` 441 → 497 (at the
line; the registration validator is the next split seam). G4 ruling RECORDED (event-catalog §Types
+ §Vendor events, canon T24 col 6) — #1078 A/02 `letter_facts` superseded.

**#1129 (Sharifajahan — titled "workflow leads dispatch immediately"; Manas REQUEST CHANGES 9 Sep at
`90246f8e` → APPROVE 10 Sep 04:46Z at `f3d0a57b`) — MY REVIEW 10 Sep at head `4de1d0c6` (one commit,
clean on `7abe5aa6`; gates on the merged tree: 1062 tests · pyrefly 0 · boundaries · 72 migrations):
REQUEST CHANGES — POSTED 10 Sep on Swaroop's go as issuecomment-5619016424.** **RE-VERIFIED 10 Sep at head `1607b339`** (two more force-pushes, retitled "events payload to get
metadata and lead_id unique fix"; Manas APPROVED again 15:04Z at this head; one commit, clean on
`7abe5aa6`; merged tree: 1063 tests · pyrefly 0 · boundaries · 72 migrations · black · isort): all
three MAJORs closed IN THE DIFF and proven by execution — `call_facts(lead)` has no ceiling and no
`dynamic` import, a 300-char declared answer reaches `record_event`'s payload whole while a declared
`outcome` is refused with the warning (T13 holds; entry.py's ceiling is the only one); `call.execute`
falls through to `get_lead_by_id` on None — a crash-retry with the accessor returning None ADOPTS its
row (same id, one row), a revisit mints a new id; the top docstring says what the code does and the
"reconciler heals within 60s" sentence is gone; the pinned-hole test flipped to
`test_the_same_visit_run_twice_is_one_lead`. **The dispatch tap stays OUT of this PR by Manas's call
("prod is dispatching") — it is in no open PR; if the backlog fix is still wanted it needs its own
PR and its own review.** NIT left: the body is still CodeRabbit's text for the dispatch feature.
**MERGEABLE.** The head is NOT the approved head: the 12:37Z force-push
DELETED the dispatch tap (`dispatch/taps.py`, its `main.py` import, `test_dispatch_schedule_tap.py`
— the fix for prod's ~17k stale BACKLOG rows that Manas said "I want today") and ADDED an unreviewed
change (`nodes/call.py` visit counter in the lead id + `test_workflow_call_visits.py` + a frozen-clock
fix in `test_workflow_walker.py`), under the old title; the tap is in no open PR. Structure verdict
on what IS here: `call_facts` + the `declared` merge in buddy's `crm_mirror.py` = RIGHT file (buddy
translates; buddy imports crm contracts only); `unenumerable_squares` per SQUARE in
`catalog_laws.py` = right file, right scope; the visit counter in `nodes/call.py` under the `lead_`
bookkeeping prefix = right file, right key. Three findings: (1) MAJOR — the head is a different PR
from the approved one and the title names the thing that left; (2) MAJOR, wrong layer —
`call_facts(lead, max_chars)` reads OUTREACH's run-context dial (`CRM_CONTEXT_VALUE_MAX_CHARS`,
"canon T20 col 12") from buddy and DROPS any declared answer over it from the letter, so the spine
never receives it (T13 verbatim law; the function's own docstring cites T13 two paragraphs above
the drop) while `entry.py` already enforces the same ceiling on the consumer side — a second reader
of one ceiling, lossy, in the producer; fix = carry every declared scalar, delete the param and the
import; (3) MAJOR — the top docstring of `call.execute` still says "a lease-retry after a crash
re-issues the same insert and the PK absorbs it" while the PR's own comment 20 lines down proves the
accessor swallows the PK violation to None (`except Exception: return None`, confirmed at
`lead_call_tracker.py`) so a crash-retry raises "lead insert returned None" and PARKS after three
attempts — the exactly-once property the corpus states for the call node is false today, and the
test `test_the_same_visit_run_twice_still_raises` pins the hole as deliberate; fix = on None, ask
`get_lead_by_id(lead_id)` (exists) — a row is ours, continue; none, raise. Also the module docstring
"the reconciler heals within 60s" is known-false per Manas's prod trace. NITs: title/body; the
frozen-clock test fix is right and unrelated (deserves its own line in the body).

**#1128 (Rabi — WhatsApp Flow buttons: named at send, submissions readable; Manas REQUEST CHANGES 9 Sep
at `1a8a43d7`, Rahul's empty APPROVE at the same SHA, four force-pushes since) — MY REVIEW 10 Sep at head
`c19233ae` (one commit, clean on `967a86df`; CI RED = the walker date bomb #1129 fixed — locally on the
merged tree 1088 tests · pyrefly 0 · boundaries · 72 migrations · black · isort, so a rebase goes green):
REQUEST CHANGES — advisory, not posted.** Right in substance and proven on a live WABA: every FLOW
button named by position, `flow_token` = the crm_message id (no wamid join), no placeholder token,
byte-identical body without a flow, `nfm_reply` decoded, `reply` answers a branchable token, `flow_token`
keyable for `match`. Manas's MAJOR (blob answer) closed by `form_submitted`; his two-BUTTONS MINOR closed
(first component only). Structure: **MAJOR (1) — the #1050 spill again, one PR later, same author:
`ApprovedTemplate.flow_button_indexes` is a Meta-shaped field on the channel-neutral registry-row shape,
and the GENERIC `db/decoders/template.py` now walks Meta's component structure (`type == "BUTTONS"`,
`button.type == "FLOW"`) — a decoder making a provider decision (rules/03 "decoders are dumb";
modules/04 "the SendRoute carries the ROW, never one provider's field; provider quirks are normalised
INSIDE the provider's face"). Extensible shape: `ApprovedTemplate.components` (T23 col 11, verbatim —
the row's own column, which the query already selects) and `providers/whatsapp/payload.py` derives the
positions (pure), so SMS-DLT reads its own thing from the same row and the decoder stays row → model.
MAJOR (2) — `record/extractors/whatsapp.py` 386 → 522: the split trigger fires in this PR (the
`nodes/` mirror rule): `extractors/whatsapp/` package, `flow.py` first (the ~110-line nfm_reply concern:
`_nfm_reply` · `_submitted` · `flow_response` · `_readable` · `flow_token` · the two constants), ENTRIES +
DERIVERS assembled in `__init__` — `SPEC_MODULES` reads only `module.ENTRIES`/`.DERIVERS`, so a package
satisfies the contract unchanged. MINOR (3) — `_wake_on_reply` (unreviewed 49-line rewrite, and a
behaviour change for EVERY source's listening square, not just WhatsApp — correct, it matches enrol) runs
the declared half through `is_bookkeeping` only while enrol runs it through `_context_from_payload`
(ceiling + scalar filter); the docstring says "the same bridge enrol uses" — make it so, one bridge.
MINOR (4) — `form_submitted` is a bare literal inside `reply()`: no constant, no catalog `values`, no
corpus word; an author must know it by folklore and the console cannot offer it — export
`FORM_SUBMITTED`, put it on the field's description, list it in event-catalog.md. MINOR (5) —
`flow_response` as one `text` variable: a multi-field form past 256 chars PARKS the run (honest, the
comment says so) but a plan cannot name ONE form field; named trigger = the first form over 256 or the
first template wanting `{address}` alone → a merchant-declared flow sub-spec, which needs the no-shadow
law amended for ADDITIVE registration under a code topic (a ruling for Swaroop, not this PR). NIT — the
`flow_token` wire key is spelled in `payload.py` (literal) and `extractors/whatsapp.py` (`FLOW_TOKEN_KEY`)
with no cross-pin test; rule 12 forbids the import but a test may import both. NIT — Rahul's empty
approval at a SHA under an open request-changes is noise; nobody has reviewed `c19233ae`.

→ rollout phase 18 message half · #1021 renumbered (070+, after #1047), rebased, extended
with `may_contact()` (Rabi) → B5 → phase 19 · #1047 event catalog review (renumbers to
068/069, rebases onto the outreach db/ split) · #1053 renumbered · X1 reshape on
nautilus#195 · recorded Shopify fixtures at shadow-live · PgBouncer before the pod count
grows again. Delivered since the last list: #1040 (Meta bay in record), #1079 (cart
`on_publish: migrate`, the T25 DELETE guard as 067, `Extracted.about`, the ingress words),
#1080 (structure PR 2), #1082 (template_status on the sending account), **#1047 (the event
catalog: T24 068 + where→conditions 069 — merged 4 Sep 12:15Z as `2a00ef44`)**, **#1085 (the WhatsApp code spec, all six
topics — merged 7 Sep 06:39Z as `ff0aaf81`)**, **#1084 (the template webhook consumer +
`connectivity/templates/` package — merged 7 Sep 09:13Z as `a55a9f53`)**, **#1057 (the `action` square + the Shopify
connector's action face via nautilus + the `outreach/nodes/` package — merged 7 Sep 11:37Z as
`f98b7ae1`)**. Owed from #1084's own
backlog: the tombstone dead end (partial unique or reopen), the provider-clock column (the
principled form of the skew), the `db/queries/template.py` split at its next builder. Next free migration = **073** (072 = `072_crm_customer_attributes_gin.sql`, 9 Sep)
release `4cfc0515`) took 070 as `070_create_ui_component.sql`; #1021 renumbers to 071+. Route inventory 34 (record 9 · outreach 12 · connectivity 11 · identity 2).
