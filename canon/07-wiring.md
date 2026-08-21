# System wiring — flows, contracts, tenancy, transform

## Module contracts (freeze these before feature code)

| contract | owner module | shape |
|---|---|---|
| `resolve(merchant, handles{}) -> customer_id` | Core | deterministic; the only creator of customer rows |
| `may_contact(merchant, customer, channel, purpose) -> {verdict, reason, send_token?}` | Permission | fail closed on any missing input |
| `send(send_token, message_id)` | Connectivity | consumes the token; the only provider call site |
| notes: `index() · note(slug) · upsert_note(slug, body)` | Core | agent-only memory shelf |
| `segment.evaluate(id) / count(id)` | Core | computed membership; rejects inferred fields |
| `broadcast.schedule(segment, workflow, when)` | Outreach | the launch button |
| `enrol(workflow, customer, trigger)` + `tick()` | Outreach | reactive door + the walker |

One squad writes each table; the boundary is a database grant, not a convention.

## Flow 1 — inbound WhatsApp from a stranger

Meta webhook → binding by receiving number (T12) → merchant → event_raw rows (message.inbound,
wamid) → `resolve()` finds nothing, creates the customer (T05) → agent loads notes (T09) +
journey (V01), answers → post-turn extraction upserts notes; structured facts via
`assert_facts()`. Nothing was typed by a human.

## Flow 2 — STOP

Inbound "STOP" → event topic → `record_consent(withdraw)` → ledger row (T07) + state flip
(T08) in one transaction. If platform-wide ("never contact me"): platform.identity
suppression write (T02). Every later send attempt: gate reads state + suppression → no, with
a T14 row and a manifest row carrying the reason.

## Flow 3 — deliberate launch (the user's gesture)

Console: pick segment → pick workflow (or quick-send mints a one-node flow) → now/schedule →
broadcast row (T17). At dispatch: freeze into T18 → guards (reenter/cooldown) skip with
reasons → everyone else becomes a run (T20, `source_broadcast_id` stamped) → the walker
proposes each send node → gate → `send()` → manifest (T16) → provider statuses walk back per
wamid → journey + reports. Funnel: T18 skips + runs' manifest rows.

## Flow 4 — reactive (abandoned checkout)

Shopify posts checkouts/create → event_raw → workflow entry rules match → enrol (T20,
`wake_at = occurred_at + wait`, no broadcast) → walker claims at wake (lease = wake_at push)
→ **re-check the goal at fire time** → gate → send → orders/create later cancels open
enrolments. Survives deploys and worker crashes with no reaper.

## Tenancy and CI

- `merchant_id` is globally unique (registry PK); `merchants.reseller_id` NOT NULL → any row
  → merchant → reseller is one total join; no crm table stores a reseller.
- CI check (runs on every PR): root-table query without a merchant_id predicate fails the
  build; child tables must be entered through a scoped parent. Plus a canon-conformance
  check: introspect the DB, diff against the sealed schema — MISSING is pending, EXTRA or
  MISMATCHED fails.

## Transform from the running system (clairvoyance)

No backfill exists anywhere in this plan. The journey view reads existing chat/call tables in
place; `resolve()` creates lazily; the Shopify connector's first sync populates the record
the normal way. Only live operational state moves:

| today | becomes | move |
|---|---|---|
| `campaign` (bulk lead push) | broadcast + one-node workflow; `crm.campaign` = tag only | POST /campaigns shim creates the new shape; old rows read-only history |
| `blacklisted_numbers` | platform.identity suppressions | one-time copy → dual-write window → flip reads; keep fail-closed-on-error |
| `credentials` (no lifecycle) | connector_installation rows pointing at the vault | backfill a door row per live credential |
| `chat_session` tables | stay — journey view reads in place | new turns also stamp event_raw (forward-only) |
| `telephony_numbers` | channel_binding, at the voice takeover | NOT in this cut; pool stays platform-side |
| `supported_channels` CHECK | dropped | one migration; WhatsApp becomes DB-legal |
| `merchants` | + reseller_id NOT NULL | backfill direct merchants to the house reseller, add constraint |

## Implementation ground rules

- Code lands inside clairvoyance: `crm` + `platform` schemas, sequential SQL migrations
  (046+), three-layer pattern queries → accessor → decoder, asyncpg, no ORM, parameterized
  `$1` placeholders only.
- Tables are created by the task that owns them (vertical slices) inside the shared scaffold
  (schemas, module roles/grants, migration conventions).
- Known ground bugs to kill first: DND pre-check fails OPEN (invert it — highest-consequence
  bug); PgBouncer transaction mode requires `statement_cache_size=0` in the same PR;
  `supported_channels` CHECK rejects WhatsApp; phone lookup does LIKE-over-JSONB.
