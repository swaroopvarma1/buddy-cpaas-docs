# 0020 — The journey's commerce arm: stamp customer_id on event_raw

**Status:** Accepted · **Date:** 2026-08-21 · Amends the canon (T13 +1 column, V01 +1
arm). Extends the stamping pattern of ADR 0017; feeds the U3 timeline (ADR 0019,
`design/console-ui.md`).

## Context
Phase 1's customer 360 shows commerce moments — order placed, checkout abandoned — on
the journey timeline (ADR 0016). Task A12 describes the journey as "per-customer
timeline over event_raw events … joined by stamped customer_id". But the canon as
sealed had a hole: `event_raw` (T13) carried no `customer_id` column, and the journey
view (V01) unioned four arms — calls, chats, messages, consent — all of which live in
tables that already carry `customer_id`. A raw Shopify order letter had no path onto a
per-customer timeline. (Caught by Swaroop asking the simplest possible question: "the
event table doesn't have a customer — how will it show?")

Three candidate mechanisms:
1. Stamp a nullable `customer_id` onto `event_raw` at processing time.
2. Project decoded commerce events into a new `crm.customer_event` domain table.
3. Resolve identity at read time by matching payload phone/email in the journey query.

## Decision
**Option 1 — stamp `event_raw`.** T13 gains one nullable `customer_id` column, written
by the resolve-and-journey processor immediately after `resolve()`, in the same update
that sets `processed_at`. V01 gains a fifth arm: `event_raw` filtered to commerce
topics, `WHERE customer_id IS NOT NULL`, with `source_kind = 'event'`.

Why this one:
- **It is already our pattern.** ADR 0017 stamps `customer_id` onto
  `lead_call_tracker` the same way: resolve once at write/processing time, then every
  read is a plain indexed join. One pattern, not two.
- **Immutability is preserved where it matters.** The *payload* is the immutable
  letter; the *envelope* already carries mutable processing state (`processed_at`,
  `quarantine_reason`). The stamp is more envelope state. Replay re-runs the parser
  and re-stamps — self-healing when identity was wrong.
- **No new table, no second copy.** Option 2 is the P2 TRIGGER table (the canon
  explicitly defers materializing the customer-event stream to phase 2, "built then
  as a rebuildable projection"). Pulling it forward buys nothing phase 1 needs.
- **Option 3 violates a standing law.** ADR 0017: never phone-match at read. Read-time
  resolution re-runs identity logic per page view and silently diverges the moment a
  handle is re-glued to a different customer.

Mechanics:
- Partial index `(merchant_id, customer_id, occurred_at) WHERE customer_id IS NOT
  NULL` for the timeline read, per partition.
- Events whose extractor finds no handle (or whose resolve is quarantined) keep
  `customer_id NULL` and simply do not appear on any timeline — excluded, not faked
  (V01's existing law).
- The stamp is which-customer only. Facts extracted from the payload (name, locale)
  still go through `assert_facts()`; consequences still land in domain tables.
  Decoding remains a verb.

## Consequences
- Canon amended: T13 is 12 columns (col 14, with trail note); V01's `source_kind`
  vocabulary gains `event`.
- A-pod: the resolve-and-journey processor task gains the stamp write; A12's journey
  view definition now has its commerce arm specified rather than implied.
- The U3 timeline renders commerce cards from V01 exactly like every other arm —
  provenance `(source_kind='event', id)` deep-links to the raw event for debugging.
- When P2 materializes TRIGGER, the stamped `event_raw` rows are its backfill source —
  the projection is rebuildable from day one, which is the property the canon wanted.
