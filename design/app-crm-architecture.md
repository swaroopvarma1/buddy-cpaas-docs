# app/crm — module architecture

**Status:** Draft for team review · **Date:** 2026-08-19 · Decisions it encodes: ADR 0001, 0006–0011.
Vocabulary comes from `../canon/` (read 07-wiring first).

## 1. Package tree

```
app/crm/
├── __init__.py            # exports NOTHING — modules are imported by path, no re-export hub
├── platform/              # the platform layer: platform.identity only
│   ├── suppression.py     #   is_suppressed probe (fail closed), suppression writes
│   ├── queries.py · decoder.py
│   └── api.py             #   admin suppression endpoints
├── identity/              # crm.customer + resolve() + assert_facts()
│   ├── resolve.py         #   THE only creator of customer rows
│   ├── facts.py           #   assertion history → materialised winners
│   ├── merge.py           #   staple, never melt
│   ├── queries.py · decoder.py
│   └── api.py
├── record/                # event_raw + journey view + (later) customer_memory
│   ├── ingest.py          #   the front door: verify → stamp → store raw → 200
│   ├── topics.py          #   topic registry + consumer dispatch (work queue on processed_at)
│   ├── replay.py
│   ├── queries.py · decoder.py
│   └── api.py             #   webhook endpoints live HERE (shopify relay, whatsapp, breeze-crm)
├── permission/            # consent ledger + state + gate + decision log
│   ├── record_consent.py  #   the single ledger writer
│   ├── gate.py            #   may_contact() — owned by the pair, never split
│   ├── importers/         #   pluggable consent importers (ADR 0008)
│   │   ├── base.py        #     ConsentImporter protocol → normalized IMPORT events
│   │   └── shopify.py
│   ├── decision_log.py
│   ├── queries.py · decoder.py
│   └── api.py
├── connectivity/          # doors, pipes, templates, manifest, send()
│   ├── installations.py   #   connector_installation lifecycle + health probe
│   ├── bindings.py
│   ├── templates.py       #   WABA template registry + Tech Provider API client (ADR 0011)
│   ├── manifest.py        #   crm.message writes; dedupe_key discipline
│   ├── send.py            #   THE only provider call site; adapters imported only here
│   ├── adapters/
│   │   └── whatsapp.py    #   Cloud API adapter (voice adapter arrives at takeover)
│   ├── dispatch.py        #   simple queue drain (ADR 0004) — worker role
│   ├── receipts.py        #   status walker: wamid transitions, monotonic ordinal
│   ├── queries.py · decoder.py
│   └── api.py
├── outreach/              # segments, workflows, broadcasts, campaign registry
│   ├── segments.py        #   definition compiler (rejects inferred attrs) + evaluate/count
│   ├── workflows.py       #   document CRUD + publish validator (draft → definition)
│   ├── walker.py          #   enrol() + tick(); wake_at = timer + lease — worker role
│   ├── broadcasts.py      #   the launch button: freeze → guards → enrol
│   ├── campaigns.py       #   the badge registry
│   ├── queries.py · decoder.py
│   └── api.py
└── shared/                # ONLY cross-module leaf utilities; no business logic, no I/O
    ├── tenancy.py         #   merchant scoping helpers the CI check recognizes
    └── clock.py
```

Per-module `queries.py → accessor (the named .py files) → decoder.py` keeps the repo's
three-layer law. **No module imports another module's queries/decoder — only its
contract functions.** No `app/crm/**` file imports from `app/ai/**`. Allowed seams into
existing code: `app/core/*` (config, logging, security primitives), `app/services/redis`,
`app/services/encryption` + the credential vault accessor, `app/database` pool.

## 2. Contracts ↔ owners

| Contract (07-wiring) | Module | Squad |
|---|---|---|
| `resolve(merchant, handles{})` | identity | Identity & Record |
| `assert_facts()` | identity | Identity & Record |
| event ingest + `replay(event_id)` + topic dispatch | record | Identity & Record |
| `record_consent(event)` + importers | permission | Permission |
| `may_contact(...) → {verdict, reason, send_token?}` | permission | Permission |
| `send(send_token, message_id)` | connectivity | Connectivity |
| template registry + status sync | connectivity | Connectivity |
| `segment.evaluate/count` | outreach | Outreach |
| `broadcast.schedule(...)` | outreach | Outreach |
| `enrol(...)` + `tick()` | outreach | Outreach |
| suppression probe/writes | platform | Permission (thin) |

Boundary enforcement is a **CI-enforced ownership map** (ADR 0001, amended 23 Aug 2026:
one DB role org-wide, so per-module grants are unavailable). Each table has exactly one
owning module (permission owns consent_*, decision_log; connectivity owns installations/
bindings/templates/message); a CI lint over SQL strings fails any PR where a table is
touched outside its owner's directory, and import-linter keeps the module graph legal.
Cross-module reads go through contract functions, not foreign SELECTs — the lint makes
cheating loud at PR time. Physical naming: `crm_*`/`platform_*` prefixes in `public`.

## 3. Runtime (ADR 0006)

One clairvoyance image. Role flags decide what a pod runs:

| Role | Runs | Notes |
|---|---|---|
| main server (default) | all `/crm/*` routers + webhook ingress | mounted in app/main.py OUTSIDE `/agent/voice/breeze-buddy` |
| `CRM_DISPATCH` worker | connectivity/dispatch.py drain loop | scale by replicas |
| `CRM_WALKER` worker | outreach/walker.py tick loop | scale by replicas |
| `CRM_CONSUMERS` worker | record/topics.py consumers (receipts, consent keywords, journey) | one pod is enough for pilot |

All DB access through the new PgBouncer (Phase 0). Workers hold no transaction across
HTTP (manifest law).

## 4. API mounting & auth (ADR 0007)

- Prefix: **`/crm`**, one router per module (`/crm/customers`, `/crm/consent`,
  `/crm/workflows`, `/crm/broadcasts`, `/crm/connectors`, `/crm/templates`, …).
- Auth: existing JWT machinery from `app/core/security` + admin/S2S dependency — **no
  new auth system**; merchant-facing RBAC arrives with the console fast-follow (after
  the users access-model read cutover).
- Webhook ingress is unauthenticated-but-verified (signature / s2s per source) and lives
  under `record/api.py`: `POST /crm/ingest/{source}/...`.

## 5. Phase-1 scope fence (what is deliberately NOT built)

- No inbound WhatsApp conversations (ADR 0003) — but ALL WhatsApp webhooks land in
  event_raw from day one; phase 2 adds a responder consumer, nothing else.
- No voice through the gate, no voice manifest rows (ADR 0010) — voice nodes stamp
  enrollment_id onto lead rows; funnel joins through that.
- No customer_memory, no journey view API polish — the view ships read-only when
  Identity & Record squad reaches it; memory shelf waits for the conversational phase.
- No merchant console; no segment UI; no template UI (APIs only).
- No five-plane message dispatcher (ADR 0004).

## 6. Cross-team dependencies (track on the board)

1. **anchor**: forward `checkouts/create`, `orders/create`, customer/consent topics with
   s2s signing (ADR 0009). Critical path for M2.
2. **Embedded-signup owner**: on WhatsApp embedded signup completion, call
   `/crm/connectors` to mint the door (installation) + pipe (binding) rows. Until wired,
   ops mints them via the same API. *(Open: which team owns the signup flow?)*
3. **Infra**: PgBouncer rollout (Phase 0, ADR 0002) precedes all crm load.

## 7. Milestones

- **M1 — skeleton send**: one gated outbound WhatsApp template message end-to-end
  (seeded customer + imported consent + door/pipe + approved template →
  `may_contact` → `send` → manifest → delivery receipt walks back). Proves every
  contract seam at minimal depth.
- **M2 — pilot flows live**: abandoned-checkout + order-confirmation running for one
  Shopify/Breeze merchant via anchor relay; funnels readable from manifest +
  decision_log; broadcast is the next increment after M2.
