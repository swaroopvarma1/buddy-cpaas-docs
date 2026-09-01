# The record — events, journey, memory

Everything that happens lands as an event; the journey is a view over the sources; memory is
what the agent carries between conversations.

### crm.event_raw (T13) — 12 columns

Everything that arrives, verbatim and immutable. Replay is the only recovery mechanism that survives being wrong about the schema. (The payload is immutable; the envelope carries processing state — `processed_at`, `quarantine_reason`, and since ADR 0020 the `customer_id` stamp.)

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK | The handle replay(event_id) takes |
| 2 | `merchant_id` | text | UQ | NOT NULL — every event arrives through a merchant-owned door (T11), so every row has an owner by construction. Made NOT NULL 18 Aug 2026 when col 13 retired; see the trail |
| 3 | `source` | text | UQ | shopify · juspay · whatsapp · telephony · widget · linktrack — half the dedupe key, and the parser router |
| 4 | `topic` | text |  | The routing key — what parsers, workers and workflow entry rules subscribe to: orders/create · message.delivered · link.clicked · optin.granted |
| 5 | `schema_version` | text |  | Stamped at ingest from the source’s own header, never inferred later — the column that makes replaying two years of events survivable when the parser was wrong |
| 6 | `external_id` | text | UQ | Idempotency — webhook id, CallSid, wamid + status (Meta sends one webhook per status transition on the same wamid). The unique key is (merchant_id, source, external_id) — plain UNIQUE, since merchant_id is NOT NULL |
| 7 | `payload` | jsonb |  | Raw, always — the letter they sent, never our understanding of it. Decoded is a verb, not a noun: decode(payload, schema_version) runs at processing time and its consequences land in the domain tables |
| 8 | `received_at` | timestamptz | PART | RANGE partition key, monthly |
| 9 | `occurred_at` | timestamptz |  | Event time when the source credibly gives one; falls back to received_at. Triggered sends measure wake_at from this, never from processing time. A claim, clamped — never later than received_at |
| 10 | `processed_at` | timestamptz |  | NULL = pending — the partial index on this IS the work queue |
| 12 | `quarantine_reason` | text |  | NULL = clean; NOT NULL = quarantined, and why — one column carries the state and the answer |
| 14 | `customer_id` | uuid | IX | Nullable — stamped by the processor right after `resolve()`, same pattern as lead_call_tracker (ADR 0017). NULL = arrived but not yet (or never) resolved to a person. The journey's commerce arm reads `WHERE customer_id IS NOT NULL`; replay re-stamps. Partial index `(merchant_id, customer_id, occurred_at)`. Added 21 Aug 2026, ADR 0020 |


**Wiring**
- The front door: verify signature → stamp envelope (merchant from the receiving credential,
  `source`, `topic`, `schema_version`, `external_id`) → store raw payload → return 200
  **before anything is understood**. One POST may fan out to N rows.
- Dedupe: `UNIQUE (merchant_id, source, external_id)` — conflict = silent drop, still 200.
- Decode failure = `quarantine_reason` set, row kept — never reject. `replay(event_id)`
  re-runs the current parser over stored raw; alert on quarantine rate, not per row.
  Trail (24 Aug 2026, A2 design): quarantine ALSO sets `processed_at`, so quarantined
  rows leave the work queue — states: pending (NULL/NULL) · done (set/NULL) ·
  quarantined (set/set); replay resets both.
- Producers: Shopify webhooks (orders/checkouts/customers topics), WhatsApp webhooks
  (message.inbound, message.status per wamid, template.status), chat turns (mirrored
  forward-only), CSV importer side-effects.
- Consumers subscribe to **topics, never source tables**: resolve-and-journey processors,
  `record_consent` (STOP/START keywords), workflow entry rules, delivery-status consumer.
- The resolve-and-journey processor stamps `customer_id` onto the row it just decoded
  (ADR 0020) — identity is resolved once, at processing time, never at read time.
- Trail (23 Aug 2026): voice mirrors PASS THROUGH the lead's creation-time stamp —
  `call.inbound` is mirrored from the created tap sequenced after the stamp (born
  attributed), `call.attempted`/`call.completed` carry `lead.customer_id`; only
  `lead.pushed` is born NULL for the consumer. Pass-through is carry, not a second
  resolution — the stamp stays envelope state the spine can always recompute.
- Monthly RANGE partitions on `received_at`. **Trail (23 Aug 2026, PR #1016): phase-1 ships
  UNPARTITIONED** — Postgres requires a partitioned table's unique constraints to include the
  partition key, which would break the load-bearing dedupe `UNIQUE (merchant_id, source,
  external_id)`; the dedupe wins. Partitioning (and partition-drop retention) lands as a
  later migration when volume demands.
- As built (051): a BEFORE UPDATE trigger freezes the ingestion fields — only `processed_at`,
  `quarantine_reason` and `customer_id` may ever change. The letter is immutable by trigger,
  not by intention.

### crm.event_schema (T24) — 12 columns *(sealed 1 Sep 2026 — vendor schemas registered at enrollment)*

One row per (merchant, source, topic) a push vendor sends — the REGISTERED contract
for their events (design/event-catalog.md §Vendor events). The fields document is
the whole registration; the row is cold by design.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | IX | Tenancy — first column of the unique below |
| 3 | `source` | text |  | The vendor's declared source (e.g. nammayatri) |
| 4 | `topic` | text |  | e.g. ride.cancelled |
| 5 | `label` | text |  | "Ride cancelled" — presentation, renameable |
| 6 | `fields` | jsonb |  | THE registration, whole: [{path, type, label, keyable, variable, values[], identity, deprecated}] — one document, never per-field rows. Type vocabulary lives in the REGISTRATION VALIDATOR (code), never a CHECK (the 027 scar); unknown types are rejected at registration, not discovered at flow-publish |
| 7 | `status` | text | CK | detected · registered. detected = the discovery upsert saw an unregistered topic (fields empty — the nudge row); registered = a human/vendor signed the schema. Registration upgrades the same row |
| 8 | `version` | integer |  | Bumped on every re-registration — the audit stamp (T19's precedent); removals are deprecations inside `fields`, never deletions |
| 9 | `registered_by` | text |  | Who signed: the s2s credential (API) or the console user |
| 10 | `first_seen_at` | timestamptz |  | Written ONCE by discovery. No per-event counters here — "312 this week" is computed on read from event_raw; this table stays cold |
| 11 | `created_at` | timestamptz |  |  |
| 12 | `updated_at` | timestamptz |  | Touched by the registration accessor's SQL |


**Wiring**
- UNIQUE (merchant_id, source, topic) — merchant-first per the tenancy law.
- Three writers, all cold: the event worker's DISCOVERY upsert (INSERT ON CONFLICT
  DO NOTHING, status=detected — one write per new topic EVER, with a small
  in-process known-set cache so the hot path never probes); the registration API
  (`POST /ingest/schemas`, s2s — same door family as the envelope door); the
  console wizard (admin route, same accessor).
- Readers: the catalog API merges this layer with the code CATALOG into one shape —
  the editor is layer-blind. The PUBLISH validator receives the merged catalog as an
  argument (gather → pure validate(definition, catalog) → apply — validate stays
  pure). **The runtime hot paths (entry evaluator, walker) NEVER read this table**:
  the validator guaranteed op↔type fit at publish; conditions evaluate directly
  against the payload.
- Migration 060. Owner: record (TABLE_OWNERS). Registration validator + code
  CATALOG + layer merge live in `record/catalog.py` (concern-named logic file);
  contracts export the reads the console needs.

### crm.journey_event — VIEW (V01) — 12 columns

One ordered history per customer — a union over stores that already exist; consent grants and withdrawals join the union as first-class moments. A view, deliberately: the table is the customer-event stream NAMED TRIGGER (P2’s first event-recency segment), built then as a rebuildable projection.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | text |  | The underlying row’s id — the deep link every card needs; with source_kind, the provenance pair. text, not uuid (fixed 23 Aug 2026): the call arm’s underlying `lead_call_tracker.id` is varchar, so every arm casts to text in the union |
| 2 | `merchant_id` | text |  |  |
| 3 | `customer_id` | uuid |  | → customer. NULL for chat sessions until identity reaches chat — see wiring R1 |
| 4 | `channel` | text |  | The card’s icon — call vs whatsapp vs instagram |
| 5 | `direction` | text |  | inbound \| outbound — she reached out vs we did; the card reads differently |
| 7 | `handled_by` | text |  | agent \| human \| both — "you spoke with the assistant" vs "Meera replied"; the takeover story |
| 8 | `started_at` | timestamptz |  | THE clock — every union arm exposes it; the timeline’s sort key and pagination cursor |
| 9 | `ended_at` | timestamptz |  | The call card’s "2m 40s" — NULL where duration is meaningless |
| 10 | `outcome` | text |  | The card’s one-line verdict — resolved · callback promised · no answer. Consent rows carry granted \| withdrawn here |
| 11 | `recording_ref` | text |  | The card’s play button — the ink that proves |
| 12 | `transcript_ref` | text |  | The card’s read-transcript link |
| 17 | `source_kind` | text |  | call · chat · message · consent · **event** — which store this row came from. (source_kind, id) is every arm’s provenance; replaces the legacy_* pair. The `event` arm (ADR 0020) carries commerce moments — order placed, checkout abandoned |


**Wiring**
- A VIEW — moves no data, cannot drift from its sources. Unifies chat sessions, call records,
  `crm.message`, consent moments **and the commerce arm** — `event_raw` filtered to commerce
  topics `WHERE customer_id IS NOT NULL` (the T13 stamp, ADR 0020) — into one ordered stream
  per `(merchant_id, customer_id)`, keyset-ordered `(occurred_at, id)`.
- Read by the timeline API (console 360) and by the agent's context assembly. Rows that
  resolve no customer are excluded, not faked.

### crm.customer_memory (T09) — 6 columns

One row per (merchant, customer, slug): a named markdown note the agent keeps — like Claude’s own memory. The agent scans the index, opens what matters, rewrites notes as a side-effect of the conversation. Nothing mechanical ever reads a note.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `merchant_id` | text | PK |  |
| 2 | `customer_id` | uuid | PK·FK | → customer · CASCADE — erasure deletes the shelf with the row |
| 3 | `slug` | text | PK | The note’s name: profile · promises · topics the agent coins (delivery-issues, diwali-order). profile and promises are reserved — always loaded, never auto-trimmed |
| 4 | `summary` | text | CK | One line, ≤200 chars — the index. The agent scans every summary before opening anything: retrieval by reading, not by similarity |
| 5 | `body` | text | CK | Markdown, ≤4,000 chars by CHECK — the note itself. Guesses live under a marked "steer — never say" heading, hedged, by convention |
| 6 | `updated_at` | timestamptz | IX | The shelf’s one clock — staleness is visible per note; the console sorts by it |


**Wiring**
- Contract (agent-only): `index() -> [{slug, summary}]` · `note(slug) -> body` ·
  `upsert_note(slug, body, summary)`. No segment, campaign or console logic may query it.
- Conversation start: index + relevant notes (always `promises`) load into agent context.
  After each turn: extraction upserts what was learned; speculative content goes under a
  hedged "guesses" heading — steers tone, never quoted to the customer.
- Structured facts (name, locale, timezone) do NOT live here — they go through
  `assert_facts()` on T05.
- Trim never drops an open commitment (the promises guard); a resolved commitment may age out.
