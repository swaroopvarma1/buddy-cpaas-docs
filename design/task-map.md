# Task map — product-led phases (6 people, 3 pods)

**Status:** Hydrated per ADR 0016 · **Date:** 2026-08-21 · Encodes ADR 0001–0016.

**Pods** (modules owned whole — the boundary-is-a-grant law):

| Pod | People | Owns |
|---|---|---|
| **A — Platform & Identity** | 2 | Phase-0 infra → scaffold → Identity & Record + spine consumers |
| **B — Permission** | 2 (the pair — never split) | consent · suppression · decision log · the gate · importers |
| **C — Connectivity → Outreach** | 1 + Swaroop (~1.5) | doors/pipes/manifest/send/templates → workflow engine in phase 2 |
| **UI lane** | Swaroop + Claude | console UI every phase; consumes no pod capacity |

If Pod C becomes the critical path, Pod A backfills Outreach after its record
pipeline — ownership transfers whole modules, never splits them.

---

## Phase 0 — base fixes (all pods, weeks 1–2; no crm feature code before it lands)

| ID | Task | Pod | Acceptance |
|---|---|---|---|
| P0.1 | **PgBouncer** (transaction mode) + `statement_cache_size=0` same rollout | A | pool exhaustion stops; dispatcher soak green; runbook |
| P0.2 | **DND pre-check fail-open → fail-closed** | B | unknown/error ⇒ no call; regression test |
| P0.3 | Drop `supported_channels` CHECK | C | WhatsApp DB-legal |
| P0.4 | Fix LIKE-over-JSONB phone lookup | C | indexed probe; plan verified |
| P0.5 | Migration CI: duplicate-number check; reserve 046+ | A | CI fails on dups; convention doc |
| P0.6 | Reseller backfill → 'breeze' + NOT NULL *(on CPAAS list)* | B | zero NULLs; constraint on |
| P0.7 | Hoist 180s session-lock TTL to one constant | C | one constant, four importers |

---

## Phase 1 — Customers end to end (UI included) + WhatsApp outbound + events + journey

Exit: pilot merchant's customers visible in the console with journeys; gate-checked
order-confirmation sends live; STOP honored.

### Pod A
| ID | Task | Depends on |
|---|---|---|
| A1 | Scaffold: `crm`+`platform` schemas, per-module roles/grants, migration conventions | P0.5 |
| A2 | POD_ROLE workers (`CRM_DISPATCH`, `CRM_CONSUMERS`; `CRM_WALKER` lands ph 2) | — |
| A3 | Tenancy CI + canon-conformance diff | A1 |
| A4 | Config resolver (one 4-tier precedence implementation) | — |
| A5 | `/crm` router mount + admin/S2S auth dependency | A1 |
| A6 | `crm.customer` (T05) + `resolve()` (probe order, partial uniques, race re-probe) | A1 |
| A7 | `assert_facts()` + attributes history + materialized winners (names for the UI) | A6 |
| A8 | `event_raw` (T13): partitions, dedupe, quarantine, `replay()` | A1 |
| A9 | Ingest front door: breeze-crm s2s + **Shopify relay adapter** (with anchor, X1) | A8 |
| A10 | Extractor registry + resolve processor consumer | A6, A8, A2 |
| A11 | *(absorbed into A14)* | — |
| A12 | **Journey view w/ voice arm**: per-customer timeline over event_raw events + manifest sends + lead_call_tracker calls read in place (joined by stamped customer_id) — inbound/outbound, attempted or not, outcome on the card | A6, A8, C3, A15 |
| A13 | **Transactional send consumer**: `orders/create` → template send via gate (no walker) | A10, C4, B5 |
| A14 | **Voice event mirroring** (ADR 0017): lead machine emits `lead.pushed` · `call.attempted` · `call.completed` · `call.inbound` into event_raw at existing choke points, forward-only | A8 |
| A15 | **Voice identity stamping** (ADR 0017): push handler + inbound answer call `resolve()` (sync door) and stamp `customer_id` on lead_call_tracker (one nullable column; buddy writes its own table) | A6 |

### Pod B
| ID | Task | Depends on |
|---|---|---|
| B1 | `consent_event` + `consent_state` + `record_consent()` (same-txn dual store) | A1 |
| B2 | `platform.identity` (T02) + fail-closed suppression accessor | A1 |
| B3 | blacklist → platform.identity (copy → dual-write → read flip) | B2 |
| B4 | `decision_log` (T14, partitioned) | A1 |
| B5 | **`may_contact()` gate** + send-token mint — spec final: `design/gate-mechanics.md` (ADR 0018); acceptance = its failure matrix as property tests | B1, B2, B4, A4 |
| B6 | Consent importer interface + **Shopify adapter** (pluggable, ADR 0008) | B1, A9 |
| B7 | STOP/START consent processor (legally required before first send) | B1, A8, A2 |

### Pod C
| ID | Task | Depends on |
|---|---|---|
| C1 | `connector_installation` (T11) + credentials-vault backfill | A1 |
| C2 | `channel_binding` (T12) + inbound address lookup | C1 |
| C3 | `crm.message` manifest (T16): partitions, dedupe_key, `source_kind` incl. 'human' | A1 |
| C4 | **`send()` + WhatsApp Cloud API adapter** + `message.queued` mirror (ADR 0015) | C2, C3, B5 |
| C5 | Dispatcher drain worker (simple queue, ADR 0004) | C3, A2 |
| C6 | Receipt processor: status walker (monotonic, cost) | C3, A8, A2 |
| C7 | **WABA template registry + Tech Provider APIs** + status sync (ADR 0011) | C1 |
| C8 | `/crm/connectors` endpoint (embedded signup mints door+pipe; ops path meanwhile) | C1, C2 |

### UI lane (Swaroop + Claude) — in loom, not a new app (ADR 0019; design: [console-ui](console-ui.md))
| ID | Task | Depends on |
|---|---|---|
| U1 | CRM wiring in loom: routes + `/crm/*` API client + scope.ts tenancy + per-merchant feature flag (shell/auth already exist) | A5 |
| U2 | **Customers list** `/customers`: search + segment-ready predicate filters (consent, reachability, source, activity) | A6 |
| U3 | **Customer 360** `/customers/[id]`: unified timeline (messages/calls/orders/consent) + state rail + live gate check + inline why-didn't-it-send (decision_log reason) | A12, B4 |
| U4 | Template manager `/channels/whatsapp/templates`: full lifecycle — list w/ Meta status+rejection reason, guided create, WhatsApp-style preview | C7 |

**Phase-1 integration milestone** (Swaroop owns): Shopify event → resolve → consent
import → order-confirmation template → gate → send → manifest → delivery ticks →
visible in the customer 360.

---

## Phase 2 — Flows & reach (static)

Exit: merchant launches a broadcast into a static flow; clicks branch it; funnel
readable. Abandoned checkout live.

| ID | Task | Pod |
|---|---|---|
| W1 | `workflow` (T19): document, draft→publish validator | C |
| W2 | `workflow_enrollment` (T20) + `enrol()` + wake_at timer-lease + `CRM_WALKER` role | C |
| W3 | Walker `tick()`: node exec, goal re-check (orders/create cancels), gate+send, voice node → lead machine (ADR 0010) | C |
| W4 | Entry-rule processor (reactive door) — abandoned checkout | C |
| W5 | **Click/branch consumers**: button.reply + link.clicked topics advance flows (clicks already land in event_raw) | A |
| W6 | Link tracking (linktrack source: short links, click events) | A |
| W7 | `segment` (T15) + `segment_member` (T21) + compiler (rejects inferred) + evaluate/count | A |
| W8 | `broadcast` (T17) + recipients (T18) + freeze/guards/enrol + `campaign` (T22) + POST /campaigns shim | C |
| W9 | UI: static flow builder (send → wait → branch-on-click), segment builder, broadcast launch + funnel | UI lane |
| W10 | CSV import (static lists via resolve(); capped consent path) | B |

---

## Phase 3 — Conversations

Epics (decompose at phase start): WhatsApp inbound generic responder · AI agent plugin
(chat brain + WhatsApp shell, session-per-window) · **team inbox** (crm.conversation +
conversation_message + projector, takeover/handback — ADR 0012–0015) · inbox UI ·
`customer_memory` (moved here: the agent is its only writer) · Instagram fast-follow ·
users access-model read cutover (inbox RBAC prerequisite).
Later phases: voice takeover (`telephony_numbers` → bindings, voice through the gate),
RCS/email, console RBAC completion.

---

## Cross-team (owners needed)
| ID | Task | Blocks |
|---|---|---|
| X1 | anchor: forward checkouts/create, orders/create, customer-consent (s2s) | A9 → phase-1 exit |
| X2 | Embedded-signup → `/crm/connectors` wiring | C8 polish |
| X3 | Name pilot merchant(s) + WABA/number | phase-1 exit validation |
