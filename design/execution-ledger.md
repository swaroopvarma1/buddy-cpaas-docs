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

PR-next: #1052 rebased onto the merged catalog as the engine's SECOND SPEC — `extractors/whatsapp.py`
exporting ENTRIES (+ one `SPEC_MODULES` line), template/account entries `about: merchant` (Rabi)
→ rollout phase 18 message half · #1021 renumbered (070+, after #1047), rebased, extended
with `may_contact()` (Rabi) → B5 → phase 19 · #1047 event catalog review (renumbers to
068/069, rebases onto the outreach db/ split) · #1053 renumbered · X1 reshape on
nautilus#195 · recorded Shopify fixtures at shadow-live · PgBouncer before the pod count
grows again. Delivered since the last list: #1040 (Meta bay in record), #1079 (cart
`on_publish: migrate`, the T25 DELETE guard as 067, `Extracted.about`, the ingress words),
#1080 (structure PR 2), #1082 (template_status on the sending account), **#1047 (the event
catalog: T24 068 + where→conditions 069 — merged 4 Sep 12:15Z as `2a00ef44`)**. Next free
migration = 070. Route inventory 34 (record 9 · outreach 12 · connectivity 11 · identity 2).
