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
- **Stamps are pass-through; resolution is singular** (sealed 23 Aug 2026): `resolve()`
  runs at a channel's system-of-record (the LCT stamp for voice; chat_session when chat
  arrives) or at the processor — never inside a mirror, never in an adapter. Mirrors
  CARRY an already-resolved customer_id when they have one: the voice created-tap
  sequences stamp → `call.inbound` mirror in one task so inbound events are born
  attributed; `call.attempted`/`call.completed` pass the lead's stamp through. Rows
  born before the stamp exists (`lead.pushed`) stay NULL and the consumer attributes
  them. This cannot diverge — the stamp is envelope state and the spine can always
  recompute it from raw payload (consumer + replay). Chat generalizes the same law:
  stamp the session the moment identity becomes known; earlier turns stay NULL until
  the consumer catches them.

## Do NOT

- **Don't parse before storing.** If the adapter understands the payload, you've built
  a coupling that replay can't fix. Raw first, 200 fast, understand later.
- **Don't let adapters touch domain tables or call resolve().** Adapters translate;
  the resolve PROCESSOR calls resolve(). One funnel or none.
- **Don't call resolve() from a mirror or any emit path.** The only emit-side
  resolution that exists is the system-of-record stamp (LCT / chat_session); every
  mirror passes that result through or records NULL for the consumer. A producer
  resolving per-event at emit is the side-loading pattern that multiplies resolve
  load and reintroduces N funnels.
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
- **The whole voice-mirror layer is bridge-period scaffolding.** When buddy retires
  and telephony enters through the spine's front door, the taps, hook registry,
  non-customer gate and pass-through all get DELETED, not migrated: calls become a
  native domain table, the per-channel resolution exception collapses into the
  processor law, and V01's call arm drops its legacy casts. Mirrors being
  forward-only with stable topic-qualified external_ids is what makes bridge-period
  events and post-retirement events one continuous stream — nothing behind the
  bridge needs rewriting when it goes.

## The voice bridge: known stamp variance, and how retirement fixes it

Accepted state (23 Aug 2026, PR #1016): the four voice topics answer "is the event
born with its customer_id?" three different ways, because each mirror is a separate
emit point in a different task:

| topic | born attributed? | why |
|---|---|---|
| `call.inbound` | yes | mirrored from the created tap, sequenced after the stamp |
| `call.attempted` / `call.completed` | yes | pass-through of the lead's stamped column |
| `lead.pushed` | no (NULL) | emitted the same instant the stamp task starts; A10 attributes it |

This variance is **task topology leaking into data shape** — locally correct at every
site, documented at every site, and harmless (nothing reads `lead.pushed` before the
consumer; the journey's call arm reads LCT, which is always stamped). Do NOT "fix" it
by adding more sequencing tricks; each trick is more bridge scaffolding.

The real fix is structural and comes with the CRM-native migration:

1. **Order becomes architecture, not an `await` someone remembered.** A pushed lead
   becomes a command: `resolve()` first, the native call state-machine row created
   referencing that customer, lifecycle events emitted FROM that row — identity
   exists before any event does, by construction. No racing tasks, nothing to
   sequence.
2. **NULL gets one uniform meaning.** Every external source (telephony callbacks,
   Shopify, WhatsApp) lands raw with `customer_id` NULL and is stamped by the
   processor in the same UPDATE that sets `processed_at`. The NULL window becomes a
   designed pipeline stage with one meaning ("not yet understood") and one metric
   (queue lag) — not a per-mirror accident needing a comment per site.

The tell that the end state is right: today's behavior takes four comments to
explain; the end state takes one sentence — *events are understood at processing
time* — and that sentence is already the law above.

Refs: 02-record.md (corpus) · entry-points-and-the-event-spine.md · ADR 0009, 0015, 0017, 0020.
