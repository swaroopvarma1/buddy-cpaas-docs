# Record (the event spine) — build guide

Owns: `crm.event_raw` (T13) · the ingest front door · the topic + extractor registries ·
the consumer runtime · `replay()` · (later) the journey view. Diagram:
`../diagrams/01-event-spine.html`. Squad: Pod A.

**Ingress design is sealed in [design/ingest-doors.md](../design/ingest-doors.md)** —
the two door kinds (envelope vs provider), the INGRESS registry, the decode
pipeline, and the build-order table. Read it before touching api.py or ingest.py.

## Build it like this

- **Provider bays are a slot, not a dict record fills** (ruled 2 Sep 2026, #1040):
  `record/ingress.py` holds `IngressSpec` + `INGRESS` + `register_ingress`; the
  entries are built where the provider's other faces live (connectivity) and
  registered from `app/crm/api.py` — rule 12 (record imports no subscriber) and
  the merchant lookup (connectivity's tables) both forbid the inline dict the
  first draft of design/ingest-doors.md showed. Record's `webhook_router` dispatches
  `GET|POST /ingest/webhooks/{provider}` through the slot and knows no vendor.
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
- **Poison rows quarantine too** (built 3 Sep 2026, #1062, T13 col 15): the claim SPENDS
  an attempt (`attempts = attempts + 1` in the claim statement, order kept by a CTE); a
  consumer that still raises at `CRM_EVENT_MAX_ATTEMPTS` (config, default 5) gets
  `quarantine_reason = "consumer_error after N attempts: …"` in its own savepoint, so a
  deterministic failure never sits at the head of the queue re-running resolve() every
  poll and never kills the batch. Below the ceiling the row stays pending, as before.
- **`customer_has_event(merchant, customer, topics, since, where=(field, value))`** is the
  contract the walker's goal re-check uses; `where` narrows the EXISTS to the letters
  about ONE thing (the order carrying this run's cart token — outreach's goal key, #1063),
  compared as text on the payload — still one indexed EXISTS, never a foreign SELECT.
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

## Building the consumer (A2) — decisions sealed ahead of build (24 Aug 2026)

- **The queue is the guarantee; the door may process opportunistically.** A
  return-200-and-spawn-a-task door is allowed ONLY as a latency optimization,
  because the consumer sweeps behind it. Per-event background tasks can never be
  the primary mechanism: an in-process task dies silently on deploy/crash (the
  work vanishes unless something scans for unprocessed rows — which IS the
  consumer), bursts couple processing concurrency to arrival concurrency (50k
  webhooks = 50k tasks = pool exhaustion at the door), and failed tasks have no
  durable retry or quarantine ledger. LISTEN/NOTIFY, if ever needed, is a wake-up
  signal on top of the scan — never a replacement (notifications are at-most-once).
- **The canonical drain query** — this exact shape, no variations:

  ```sql
  SELECT id, merchant_id, source, topic, schema_version, payload
  FROM crm_event_raw
  WHERE processed_at IS NULL
  ORDER BY received_at
  LIMIT 100
  FOR UPDATE SKIP LOCKED;
  ```

  Process the batch, stamp `customer_id + processed_at` in the SAME transaction,
  commit. The `LIMIT` (50–200) is both the backpressure valve and the lock-window
  knob. Keep the transaction short; never hold it across an external call
  (extractors are pure Python — keep them that way).
- **Claim semantics — why two workers can't collide**: `FOR UPDATE` locks the
  batch inside the worker's transaction (the lock IS the claim — no claimed_by
  column, no coordination); `SKIP LOCKED` makes other workers skip, not wait;
  the stamp inside the same transaction means a committed row never matches the
  predicate again. Crash mid-batch → transaction rolls back, locks evaporate
  (Postgres, no reaper), rows reappear pending. That is at-least-once delivery,
  so the processor MUST stay idempotent — and is by construction: resolve() is
  upsert-shaped, assert_facts appends, re-stamping is a no-op.
- **Ordering is approximate FIFO only** (`ORDER BY received_at` + SKIP LOCKED =
  best-effort). Consistent with the standing promise: no consumer may assume
  cross-event ordering.
- **Quarantine leaves the queue**: quarantine SETS `processed_at` too. Three
  clean states — pending (NULL/NULL) · done (set/NULL) · quarantined (set/set) —
  and `replay()` resets both to re-queue. This keeps the pending partial index
  exact; a quarantined row with processed_at NULL would be re-picked forever.
- **Capacity, for sizing conversations**: extractor is microseconds; ~two small
  transactions per event → 5–15 ms/event → roughly 70–200 events/sec per worker,
  ~5–15M/day per replica, near-linear with replicas via SKIP LOCKED. Phase-1
  volume is <100k/day — one replica at a 1s poll is ~1% duty cycle, and the A15b
  backlog is minutes of work. The real constraint is the CONNECTION BUDGET:
  PgBouncer lands before A2 multiplies connections (P0, ledgered).

## The consumer registry — COMMITTED for the next PR (Swaroop, 31 Aug 2026)

Today the pass's entry-rules slot imports its one subscriber by name
(`record/workers.py` -> `outreach.contracts.consume_attributed_event`) — the import
cycle this created in #1029 was broken by the composition-root exception (record's
contracts no longer export the pass; `worker_main` takes it directly). That fix
avoids the cycle; the registry makes it impossible:

- `record/workers.py` holds `CONSUMERS: list` and calls each per row, inside the
  row's savepoint, before the stamp — it imports NO other module.
- `app/crm/worker_main.py` (the composition root) registers subscribers at startup:
  `register_consumer(consume_attributed_event)` — outreach today, segments in P2,
  each a one-line registration.
- A consumer raise still rolls back only its row (unchanged); registration order is
  declared in worker_main, one place.

Precedent: EXTRACTORS (#1020), NODE_TYPES (#1029). Not deferred to consumer #2 —
the correct-from-the-start ruling applies; lands in the next record-touching PR.

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
