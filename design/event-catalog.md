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

## Vendor events — registered at enrollment (RULED 1 Sep 2026, Swaroop)

Push vendors (NammaYatri-type, Flipkart-type) send THEIR events with THEIR
schemas. We do not infer their schemas from traffic — **they register them when
they enroll**, the same onboarding step that provisions their s2s credential:
*here is who I am, here is what I will send, here is what is inside it.* The
registration is stored tenant-scoped in the DB and behaves EXACTLY like a
code-catalog entry — the catalog API merges the two layers and the editor never
knows which one answered.

**The registration is in OUR language, small and closed** — a field list (path ·
type from the closed set: Text / Number / Choice(+values) / Yes-no / Date-time /
Phone · label · keyable flag · usable-in-messages flag), because types must map
one-to-one onto the where-grammar's operators. Unknown types are rejected AT
REGISTRATION, never discovered at flow-publish. Identity keys follow the standard
push contract (customer_mobile_number / customer_name → the generic extractor;
push vendors need no custom extractor — registration covers the catalog side,
the standard keys cover resolve()).

**Worked example — NammaYatri registers `ride.cancelled`:**
```
ride_id              Text     "Ride ID"       keyable (runs are about the ride)
customer_mobile_number Phone  (identity — feeds resolve, never filterable)
fare                 Number   "Fare"          usable in messages
cancelled_by         Choice   "Cancelled by"  values: driver · customer · system
cancellation_reason  Text     "Reason"        usable in messages
```
…plus `ride.booked` as the goal topic. Their ops then builds: *when Ride
cancelled where Cancelled by is driver → wait 2m → WhatsApp "rebook?" →
wait-for-reply 15m → call* — keyed by Ride ID (two cancelled rides = two runs),
`ride.booked` goal-cancels the right one. Zero code shipped by us.

**The enforcement law: ingestion is free, automation requires a registered
schema.** The doors store every letter, always — including unregistered topics
and schema-violating payloads. But nothing hides: the events screen shows
"Unregistered: ride.completed · 312 this week — register to use in workflows",
and conformance counters surface drift ("fare — registered Number, arrived Text
in 12% of events"). The publish validator accepts conditions, entry.keys and
template variables ONLY from declared fields (code layer or registered layer).

**Identity mapping — their names, our handles (amended 1 Sep 2026).** Vendors
keep their own field names; the registration points at them: `identity` is a ROLE
(`phone` | `name`), at most one field per role — NammaYatri marks `rider_phone` as
identity:phone and never renames anything in their event bus. The DECODE step is
schema-aware: the generic extractor asks the registered mapping where the phone
lives (through the same in-process TTL cache as topic discovery — the cold-table
law stands refined: the FLOW runtime never reads T24; decode reads the cached
mapping), pulls it, normalizes to E.164, and resolve() proceeds normally.
Precedence: registered mapping first, standard keys
(customer_mobile_number/phone/customer.phone) as fallback — conventional vendors
never think about it. Registering an event with NO phone role is allowed but told
plainly: "these events will store and count, but can't start or advance any flow"
— unattributed events can neither enrol nor goal-cancel. The wizard pre-suggests
roles from sample values (+91… → phone). **The no-identity state is FLAGGED at
every later surface, not just at registration**: the "Your events" row wears a
standing "No customer identity" badge (grey — a declared choice, not a fault), and
in the flow editor such events appear DISABLED with the reason inline in the
trigger picker, the goal picker and wait_event topics (all three require
attribution) — visible-but-disabled teaches; missing confuses. The amber cousin is
drift, not choice: a schema WITH a phone role whose traffic arrives without the
value ("rider_phone missing in 8% of events → quarantined") surfaces via the
conformance counters.

**Schema changes = re-registration under the same laws**: additive lands
instantly; removing a field a live flow uses = deprecated-never-deleted + the
same validator badge Shopify deprecations get. One lifecycle, two declarers.

**Inference survives as the safety net only, never a stage**: (a) the
unregistered-topic nudge; (b) the registration wizard PRE-FILLS from the
vendor's real traffic when it exists (compute-on-read `jsonb_each` over recent
events — no Redis, no stats cache table until a named latency trigger; a
registered-schema row stores decisions, which no query can recompute).

**The console surface ("Your events", under Channels & Integrations)**: an
events table (label · topic · seen-this-week · conformance % · Registered /
amber Detected-not-registered — the amber rows are the to-do list, the parked
pattern) + a 3-step no-JSON wizard: name it → fields table pre-filled from
samples (plain-language types, example values, one radio for "identifies the
thing a run is about") → review-as-sentences → Register. The API path
(`POST /ingest/schemas`, s2s) writes the identical row for engineering-led
vendors; our ops can drive the wizard on a screen-share for enterprise
onboarding.

**Companion rulings**: legacy `entry.where` equality maps → ONE migration to the
typed condition list (definition + draft), validator accepts lists only.
Per-flow inline field types rejected outright (scattered, ungoverned, two flows
can disagree on one field's type).

## The unified path — proof there is one system, not two

Trace both kinds of event; they use identical machinery and differ at exactly ONE
lookup:

| Moment | Shopify order | NammaYatri `ride.cancelled` |
|---|---|---|
| Door | envelope door (via nautilus) | envelope door (their s2s push) — same door |
| Storage | `crm_event_raw`, raw letter | same table, same law |
| Decode | `EXTRACTORS["shopify"]` (code) | generic extractor + standard identity keys (`customer_mobile_number`/`customer_name`) — push vendors need NO custom extractor |
| Identity | `resolve()` | identical |
| Catalog lookup | **code layer** answers | **registered layer** (T24 row) answers — the only different line |
| Editor UX | typed conditions, dropdowns, variables | identical — the editor is layer-blind |
| Entry rules · walker · goal-cancel · sends · calls | identical | identical |
| Publish validator | declared fields only | same rule — their signature instead of our PR |

One spine, one editor, one validator — two declarers. The storage/lifecycle spec is
canon **T24 `crm_event_schema`** (canon/02-record.md, migration 061): detected →
registered, cold by design (discovery writes once per topic ever; counts computed on
read), and the runtime hot paths NEVER read it — registration governs authoring,
publish-time validation guarantees op↔type fit, runtime evaluates conditions
directly against the payload.

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
5. Vendor schema registration: table (record-owned, canon entry owed) +
   `POST /ingest/schemas` + catalog-API layer merge + unregistered-topic nudge +
   wizard pre-fill query. 6. Where-shape migration (maps → typed lists, one PR
   with the where-grammar).

Refs: design/ingest-doors.md (the decoder's obligations gain the catalog) ·
modules/01-record.md (registry home) · modules/05-outreach.md (entry vocabulary) ·
design/console-ui.md (segment-predicate shape).
