# The Event Catalog — schema-driven flows (sealed 1 Sep 2026)

*The ruling in one sentence: **nobody types a field name — every field the product
shows, filters on, keys by, or templates with comes from ONE declared registry that
grows in the same PR as the decoder.***

The workflow editor's trigger picker, condition builder, "runs are about" selector,
goal picker, template-variable menus — and phase 2's segment builder — all render
from the catalog. Free-typed fields are how a flow filters on `gatway` and silently
never matches; the catalog is how that mistake becomes untypeable.

## The object

One registry: **(source, topic) → what this event is and what's inside it.**

```
shopify · orders/create
  label: "Order placed"          group: Shopify        goalable: true
  keyable: [order_id]            ← legal entry.key choices ("runs are about")
  fields:
    payload.gateway     enum    "Payment method"  ops: is · is not · in    e.g. COD
    payload.total       number  "Order total"     ops: > ≥ < ≤ =           e.g. 1850
    items_count         number  "Items"  DERIVED  ops: > < =
    payload.customer_mobile_number  phone  "Customer phone"  (identity — not filterable)
  context_fields: [total, items_count, first_item_name]   ← ride into {templates}
```

- **Paths are identity, labels are presentation** — labels rename freely; paths never.
- **Additive-or-deprecate**: entries and fields are deprecated, never deleted. The
  publish validator warns on deprecated fields; the flows list badges flows using them.

## Ownership — the decoders ⇒ UI law

The catalog lives in **record**, beside `EXTRACTORS` (the fourth house registry:
EXTRACTORS · NODE_TYPES · INGRESS · CATALOG). A new source or topic ships FOUR things
in ONE PR: ingress entry (if a new door) · extractor · **fixtures from recorded
payloads** · catalog entry. Pin tests enforce the square:

1. no extractor without a catalog entry, no catalog entry without an extractor;
2. **every declared field appears in at least one fixture** — fixtures double as
   catalog conformance tests, so provider drift (Shopify renames `total`) fails CI
   when fixtures are refreshed, not production when flows stop matching;
3. every `derived` field has its `derive()` exported by the same decoder module.

One API serves it (ETag-cached, version-stamped); the console renders ONLY what the
catalog declares and never hardcodes a field. Adding Meta = the four-part PR; the
trigger picker, filter builder and variable menus update themselves on next load.

## The where-grammar (v1, sealed)

`entry.where` graduates from an equality map to typed conditions
`[{field, op, value}]`, ANDed. The op set is **closed and dual-implementable**:

| Type | Ops |
|---|---|
| string · enum | `is` · `is not` · `in` |
| number | `>` · `≥` · `<` · `≤` · `=` |
| any | `exists` |

Explicitly NOT v1: regex, contains, array-any. The same predicate shape compiles to
SQL for segments (P2, per console-ui's segment-predicate rule) — so **an op lands in
the catalog + validator + Python evaluator + SQL compiler in one PR, or not at all**
(parity pin test when the segment compiler exists). The UI shows exactly the ops the
catalog declares for the field's type; the engine implements exactly those; no layer
promises what another can't keep.

Nested objects: flat dot-paths. Arrays: addressable ONLY through declared derived
fields (`items_count`) — the matcher never learns array semantics.

## Merchant events — the third layer, with a lifecycle (amended 1 Sep 2026)

Push merchants (Flipkart-type, NammaYatri-type) send THEIR events with THEIR
schemas. We do not guess at them — we contract them. The catalog has three layers:

1. **Declared in code** (global, ours): shopify · meta · voice — the four-part
   decoder PR. Authoritative.
2. **The merchant catalog** (tenant-scoped, a real table, record-owned — canon
   entry owed): every (merchant_id, source, topic) a push merchant sends becomes an
   entry with a LIFECYCLE — `observed → draft → published`:
   - *observed*: the event worker's discovery upsert creates the entry the first
     time an unknown (merchant, topic) arrives — topic, first/last seen, count.
     One row per new topic, never per event.
   - *draft*: opening "Define this event" in the console runs the inference query
     (jsonb_each over that merchant's recent events, compute-on-read, ETag-cached)
     and PRE-FILLS the draft: fields, inferred types, sample values. Inference
     writes the first draft; it is never the authority.
   - *published*: a human (merchant admin or our ops at onboarding) reviews,
     corrects, publishes. The event becomes first-class: trigger picker, typed
     conditions, keyable, context_fields — identical UX to Shopify. Hand-declaring
     a custom field = editing the draft; declared once, reused by every flow.
   The fields document follows T19's draft/publish discipline; edits never
   silently change what live flows validated against.
3. **Observed decoration** (assist only): sample values and seen-counts enrich
   published fields' dropdowns; never consulted by the validator.

**The enforcement law: ingestion is free, automation requires a published schema.**
The doors store any letter, always (the spine law does not move). But the publish
validator accepts conditions, entry.keys and template variables ONLY on published
catalog fields — code layer or merchant layer, no third source of truth. Draft
fields render greyed in the editor ("define this event to use it"). The product
moment this buys: "New event detected: ride.cancelled · 2.4k this week — define it
to use in workflows" → pre-filled draft → two-minute review → publish → their ops
builds the reschedule flow with the same typed UX as a Shopify merchant.

Implementation rulings (1 Sep, for the catalog build): observed-overlay option =
compute-on-read at define-time + decoration (NO Redis — tenancy lives in table
shape and a flush must never empty dropdowns; NO stats cache table until the named
latency trigger — the merchant catalog table stores DECISIONS/lifecycle, which no
query can recompute, so it is not derived state). Legacy `entry.where` equality
maps: ONE migration rewrites definition + draft to the typed condition list;
validator accepts lists only — dual-shape parsing is refused at its cheapest
moment. Per-flow inline field types are rejected outright (scattered,
ungoverned, two flows can disagree on one field's type).

## Drift observability — seen vs matched

The entry processor counts, per flow: entry events EVALUATED vs runs STARTED (and
the skip reason breakdown it already logs). Surfaced on the flow list and editor:
"saw 240 Order placed · matched 3" — a filter gone stale (drifted field, wrong
value) is a dashboard fact within hours, never a merchant complaint within weeks.

## Deferred, with named triggers

- **Logical events** ("Order placed — any store" aliasing several concrete topics):
  flows bind to concrete (source, topic) in v1, honestly. Trigger: second storefront
  source for one merchant.
- **Keyed-flow field check**: a flow keyed by field F requires F present in its goal
  topics' catalog entries — publish-validator rule; lands with the first keyed flow
  (joins that trigger bucket).
- **Array operators**: only if a real flow needs them past derived fields.
- **Catalog i18n / size**: non-issues for years; one cached GET.

## Owed (backend, small, in order)

1. CATALOG registry + pin tests in record (with the Shopify orders/create entry).
2. Catalog API endpoint (ETag).
3. Typed where-grammar: validator + entry-processor evaluator (+ canon touch:
   `entry.where` shape).
4. Seen-vs-matched counters on the entry processor.
5. Merchant catalog table (observed→draft→published; discovery upsert in the
   event worker; define-time inference endpoint) — canon table entry owed.
6. Where-shape migration: equality maps → typed condition lists (with the
   where-grammar change, one PR).

Refs: design/ingest-doors.md (the decoder's obligations gain the catalog) ·
modules/01-record.md (registry home) · modules/05-outreach.md (entry vocabulary) ·
design/console-ui.md (segment-predicate shape).
