# Entry points and the event spine — how customers (and everything else) get created

**Status:** Draft for team review · **Date:** 2026-08-19 · Companion to `app-crm-architecture.md`.

## The problem this kills

Customers arrive from everywhere: Shopify events, WhatsApp inbound (phase 2), lead
pushes, CSV imports, chat sessions, channels that don't exist yet. If each connector
implements "get-or-create customer," we get N duplicate-detection behaviors, N places to
fix bugs, and identity logic smeared across the codebase. The same applies to consent,
enrollment, and manifest writes.

## The pattern: dumb edges · one spine · thin bridges · single writers

```
ANY source ──▶ ADAPTER ──▶ event_raw ──▶ TOPIC CONSUMERS ──▶ MODULE CONTRACTS ──▶ tables
              translate      record          bridge               write
```

1. **Adapters are dumb translators.** Verify the source (signature / s2s), stamp the
   envelope (merchant from the receiving credential, source, topic, external_id,
   schema_version), store the RAW payload, return 200. Adapters NEVER call resolve(),
   never touch domain tables. One adapter per source, living in `record/api.py`.
2. **event_raw is the only spine.** Dedupe happens here once for every source
   (`UNIQUE (merchant_id, source, external_id)`) — consumers never see duplicates.
3. **Topic consumers are the bridges** — the only event-driven callers of contracts:
   - **resolve processor** — subscribes to every handle-bearing topic, extracts handles
     via the *extractor registry*, calls `resolve()`, then `assert_facts()` for profile
     claims riding the payload.
   - **consent processor** — STOP/START keywords, importer outputs → `record_consent()`.
   - **entry-rule processor** — topics matching live workflow entry rules → `enrol()`.
   - **receipt processor** — provider statuses → the manifest walker.
4. **Contracts are the only writers.** Per-module DB grants make violations fail loudly.

**The extractor registry** is the piece that makes new channels cheap: a per-(source,
topic) map of PURE functions `payload → {handles, facts}`. Examples:
`shopify/checkouts.create → {phone, email, shopify_customer_id}` ·
`whatsapp/message.inbound → {phone}` · `lead-api/lead.pushed → {phone}`.
Adding a channel = one adapter + one extractor registration. Zero identity logic.

## Two sanctioned doors into every contract

| Door | When | Examples |
|---|---|---|
| **Async** (default) | recording what happened; caller doesn't need the result | all webhook-driven flows |
| **Sync** | caller needs the result NOW | phase-2 inbound conversation calling `resolve()` before the agent speaks; console consent capture; CSV import loop; broadcast freeze calling `enrol()` |

Both are safe because contracts are **idempotent by construction**: `resolve()` probes
deterministic partial-unique indexes; two racing callers converge on the same row
(insert race = re-probe). Idempotency at the contract level — not caller coordination —
is what lets many entry points share one writer.

## The same shape for every module

| Module | Single writer | Event bridge | Sync callers |
|---|---|---|---|
| identity | `resolve()` / `assert_facts()` | resolve processor | runtimes, imports |
| permission | `record_consent()` | consent processor | console API |
| outreach | `enrol()` | entry-rule processor | broadcast freeze |
| connectivity | manifest + `send()` | receipt processor | walker send nodes |

One sentence: **edges translate, the spine records, bridges call contracts, contracts
write.** Nothing else writes, ever.

## Facts vs commands — the spine is not a command bus

**Facts enter through the spine. Commands enter through contracts. Everything exits
through one send path. Consequences come back in as facts.**

- A **fact** is something that already happened (checkout created, message arrived,
  STOP received, call ended). Facts ALWAYS land in event_raw; consumers decide the
  consequences.
- A **command** is a caller wanting something to happen NOW and needing an answer
  (create a lead, launch a broadcast, send this message, capture this consent). Commands
  go through the API into the contract functions synchronously — they are NOT queued
  through event_raw first, because commands need synchronous validation ("400, bad
  phone") and an id back. The contract functions are already the single choke point;
  queueing commands would add latency and lose validation for zero benefit.
- Commands may still MIRROR a fact into the spine (the lead-push tap: the command
  executes synchronously, and a `lead.pushed` fact records that it happened).
- Every outbound consequence exits through ONE path — gate + send() (voice: the lead
  machine until the takeover, ADR 0010) — and its receipts/outcomes re-enter as facts,
  closing the loop (journey, goal checks, enrollment cancellation).

End-state example — "create a lead for an outbound call": fact-triggered (checkout
event) → spine → entry rule → enrol() → walker → gate → send(voice). Command-triggered
(API/console "call now") → contract directly (quick-send workflow enrolment) → same
walker, same gate, same send path. Same writers, same exit; only the entry door differs.

## Rules that keep it scalable

- **Ordering is not promised.** Consumers drain a partial-index work queue
  (`processed_at IS NULL`, `FOR UPDATE SKIP LOCKED`, `CRM_CONSUMERS` role — scale by
  replicas). Contracts must therefore be order-tolerant: resolve() is naturally;
  the consent resolver orders by `occurred_at`, not arrival time.
- **Failure isolation by quarantine.** Bad payload → `quarantine_reason` set, row kept,
  alert on the rate. `replay(event_id)` re-runs the CURRENT extractor. A broken new
  channel piles up quarantined rows; it cannot corrupt identity.
- **No consumer talks to another consumer.** Coordination happens through the domain
  tables consumers write, never through consumer-to-consumer signaling.

## Phase-1 scope notes

- **Lead-push tap (in scope):** `push_lead_handler` stays untouched + one forward-only
  mirror: a `lead-api / lead.pushed` event per push. The resolve processor builds
  `crm.customer` from live voice traffic from day one — no backfill needed later.
- **Chat sessions:** same forward-only mirroring when the conversational phase arrives
  (per 07-wiring); not phase 1.
- **WhatsApp:** ALL webhooks (statuses, template updates — and later inbound messages)
  land in event_raw from day one, so phase-2 inbound is a new consumer, not new plumbing.

## What the spine is NOT — through it, not from it

Everything from outside goes THROUGH event_raw (no exceptions, including phase-3
inbound messages). Almost nothing is SERVED from it. The letters/ledgers rule,
asked and settled 21 Aug 2026:

**A moment can be read from the spine; a state machine needs a domain table.**

| Thing | Moment or state? | Serve from |
|---|---|---|
| Order placed, checkout abandoned | Moment — happened once, never changes | T13 directly via the ADR 0020 stamp (V01 event arm) |
| Outbound message status | State machine — queued to sent to delivered to read, one webhook PER transition | `crm.message` (T16), walker updates one row; the 4 letters stay in T13 |
| Consent | State — the gate needs "may I?" in a millisecond, not a ledger replay | T08 resolved state; T07 is the ledger; T13 keeps the letters |
| Inbound message (P3) | State — threading, read state, assignment, open/closed | `crm.conversation` + `conversation_message` (ADR 0014/0015); T13 keeps the letter |

Why domain tables must exist at all:
1. **Claims vs current state.** The spine records what was said; domain tables hold
   what is true now. Aggregating N letters per read does not scale past a demo.
2. **Raw jsonb vs real columns.** Serving reads from payloads means every
   schema_version parser lives on the read path forever. Decoded is a verb —
   decode once at processing time, consequences land in columns.
3. **Invariants.** The spine's law is fail-safe: store everything, reject nothing —
   so it can enforce nothing. CHECKs, FKs and uniques live where understanding lives.
4. **Replay safety.** replay() is only safe because its output lands elsewhere.
   If live state lived in event_raw, replay would overwrite what it is meant to heal.

The ADR 0020 customer_id stamp is an index into the mailbox, not a promotion of the
mailbox into a database. If a new feature needs updates, lifecycle, invariants or
joins beyond the customer, it projects into a domain table — the spine keeps the
letter.

## Worked example — one Shopify order, end to end (every table it touches)

Ravi buys sneakers (Rs 2,499) at Kicks & Co, phone +91 98765 43210. The whole
machine, step by step. (Added 21 Aug 2026 alongside ADR 0020.)

**Write path — the fact comes in**

| # | What happens | Table / contract |
|---|---|---|
| 1 | Shopify fires `orders/create`; anchor relays (X1) to the ingest door (A9) | no CRM table — anchor is a pipe |
| 2 | Front door: verify signature → stamp envelope → store the letter RAW → 200. `customer_id = NULL` at this moment | `crm.event_raw` (T13): source=shopify, topic=orders/create, external_id=webhook id, payload verbatim |
| 3 | Consumer drains `processed_at IS NULL` (partial index, SKIP LOCKED) | reads T13 |
| 4 | The (shopify, orders/create) extractor pulls `{handles: phone/email, facts: name}` | extractor registry (pure function) |
| 5 | `resolve()`: known handle → existing id; unknown → new customer + identity rows | `platform.identity` (T02) + `crm.customer` (T05) |
| 6 | `assert_facts()` records the name claim; winners materialise | T05 `attributes` + materialised columns |
| 7 | Consent import if the order carries opt-in (ADR 0008) | `crm.consent_event` (T07) → `crm.consent_state` (T08) |
| 8 | **The stamp (ADR 0020):** one UPDATE sets `customer_id` AND `processed_at` on the event row | T13, same row as step 2 |

**Consequence — the message the order causes**

| # | What happens | Table / contract |
|---|---|---|
| 9 | Transactional send consumer (A13, topic orders/create) picks the order-confirmation template | reads T13, template registry (C7) |
| 10 | The gate renders its verdict AT DISPATCH (ADR 0018) — recorded either way; this is "why didn't it send" | reads T08 → writes `crm.decision_log` (T14) |
| 11 | PASS → `send()` writes the manifest row, **born with customer_id, never resolved** — adapter delivers via the merchant's number | `crm.message` (T16) · T11/T12 (which door, which number) |
| 12 | Meta status webhooks (one per transition per wamid) arrive as new letters; the receipt walker moves status monotonically | new T13 rows (message.status) → updates T16 |

**Read path — the 360 timeline**

| # | What happens | Table / contract |
|---|---|---|
| 13 | Console U3 asks the timeline API for the customer | `crm.journey_event` (V01) — a VIEW, moves no data |
| 14 | V01 unions five arms, every one joined by an already-stamped `customer_id`, never phone-matched at read (ADR 0017): event=T13 commerce topics · message=T16 · call=lead_call_tracker · consent=T07 · chat=phase 3 | keyset `(occurred_at, id)` |
| 15 | One card per row: "Order placed · Rs 2,499 · Shopify", then "Order confirmation · delivered" right beneath it | rendered by U3 |

**Where `customer_id` may be NULL — deliberately different answers per table**

| Table | Nullable? | NULL means |
|---|---|---|
| `event_raw` (T13) | **Yes, by design** | (a) just arrived, not yet processed — every row starts NULL; (b) processed but not about a person (template.status, receipts) — NULL forever, correctly; (c) quarantined / no handle found — kept, replay re-stamps |
| `crm.message` (T16) | **Never** | a send is born addressed; no customer_id → you may not call `send()` |
| `lead_call_tracker` | Yes | pre-CRM rows + not-yet-resolved calls (ADR 0017 stamp is forward-only) |
| `journey_event` (V01) | Moot for the 360 | the timeline query is `WHERE customer_id = X`; NULL rows cannot appear — excluded, not faked |

The rule of thumb: **NULL on the spine is a truthful state ("arrived, not yet tied
to a person"); NULL on the manifest is a bug.** The spine keeps everything; the
timeline shows only what is genuinely resolved.
