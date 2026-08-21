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
