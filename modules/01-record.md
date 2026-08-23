# Record (the event spine) — build guide

Owns: `crm.event_raw` (T13) · the ingest front door · the topic + extractor registries ·
the consumer runtime · `replay()` · (later) the journey view. Diagram:
`../diagrams/01-event-spine.html`. Squad: Pod A.

## Build it like this

- **The front door does four things and stops**: verify the source (signature / s2s) →
  stamp the envelope (merchant from the receiving credential, source, topic,
  schema_version, external_id) → store the RAW payload → return 200. **Before anything
  is understood.** One POST may fan out to N rows.
- **Dedupe once, for everyone**: `UNIQUE (merchant_id, source, external_id)`; conflict =
  silent drop, still 200. Consumers never see duplicates and must not implement dedupe.
- **Partitioning deferred in P1** (trail in canon T13): unpartitioned because the dedupe
  UNIQUE must not include a partition key; revisit on volume. Ingestion fields are frozen
  by the 051 immutability trigger — only the processing envelope may change.
  `occurred_at` is the source's claim,
  clamped ≤ `received_at`. Timers downstream measure from `occurred_at`.
- **The work queue is a partial index**: `WHERE processed_at IS NULL`, drained with
  `FOR UPDATE SKIP LOCKED` by CRM_CONSUMERS replicas.
- **Decode failures quarantine, never reject**: set `quarantine_reason`, keep the row,
  alert on the RATE. `replay(event_id)` re-runs the CURRENT parser over stored raw —
  replay is the recovery mechanism that survives being wrong about a schema.
- **Extractors are pure functions** registered per (source, topic): payload →
  `{handles, facts}`. Adding a channel = one adapter + one extractor registration.
- **The processor stamps `customer_id`** (nullable, ADR 0020) on the event_raw row in
  the same update that sets `processed_at`, right after `resolve()`. This is how the
  journey's commerce arm finds "order placed" per customer: resolve once at processing
  time, plain indexed read forever after. Replay re-stamps; unresolved rows stay NULL
  and appear on no timeline.
- **Mirrors are forward-only**: lead machine call events (ADR 0017), send()'s
  `message.queued`, later chat turns. No backfill exists anywhere in this plan.

## Do NOT

- **Don't parse before storing.** If the adapter understands the payload, you've built
  a coupling that replay can't fix. Raw first, 200 fast, understand later.
- **Don't let adapters touch domain tables or call resolve().** Adapters translate;
  the resolve PROCESSOR calls resolve(). One funnel or none.
- **Don't make the spine a command bus.** Commands need synchronous validation and an
  id back; queueing them loses both (ADR: facts vs commands). If an API handler wants
  to "emit an event and hope", it's a command wearing a costume — call the contract.
- **Don't subscribe consumers to source tables or to each other.** Topics only.
  Consumer-to-consumer signaling creates ordering dependencies nobody promised.
- **Don't UPDATE or DELETE payloads, ever.** The letter they sent is the letter they
  sent. Corrections are new events.
- **Don't skip external_id because a source "doesn't have one".** Compose one
  (hash of payload + timestamp bucket if truly absent) — losing idempotency here
  poisons every consumer.
- **Don't put per-row alerting on quarantine.** Rate alerts; a quarantined row is a
  fact awaiting a better parser, not an incident.

## Scale & future-fit

- Consumers scale by replicas; the partial index keeps the scan cheap regardless of
  table size. Partition drops are the retention story.
- A new channel (Instagram, RCS, email) must require: adapter + extractor + topics.
  If it requires touching a consumer's core loop, the registry abstraction leaked —
  fix the registry, not the consumer.
- Journey view arrives later as a VIEW over existing stores — never a copy table.

Refs: 02-record.md (corpus) · entry-points-and-the-event-spine.md · ADR 0009, 0015, 0017.
