# Connectivity — build guide

Owns: `crm.connector_installation` (T11) · `crm.channel_binding` (T12) · the template
registry · `crm.message` (T16) · `send()` · the dispatcher · the receipt walker.
Diagram: `../diagrams/04-connectivity.html`. Squad: Pod C.

## Build it like this

- **Doors point at the vault**: `credential_id` references the existing credential
  store; one credential row = one installation's whole key bundle; rotation = new
  bundle + pointer flip. The health probe walks configured → authenticated →
  subscribed → heartbeat → healthy and writes `health_detail {level, why, checked_at}`
  — a why is mandatory below healthy. `last_event_at` is stamped by arriving traffic
  (catches the token-valid-but-subscription-dropped silent killer).
- **Pipes**: `address` is the unique inbound which-merchant lookup. `capabilities`
  jsonb clamps composition (buttons? session window?) BEFORE anything composes a
  message. `is_primary` partial unique per (merchant, channel). `paused` refuses sends
  and expects the pipe back; `retired` refuses forever and the row never dies —
  manifest rows point at it.
- **Templates (ADR 0011)**: create/submit/list via Meta's Tech Provider APIs; local
  registry (name, language, category, status) kept honest by the
  `message_template_status_update` webhook through event_raw. A send referencing a
  non-approved template is refused BEFORE the provider call — manifest row, honest
  reason.
- **The manifest**: one row per outbound attempt, **blocked attempts included**
  (reason, no provider call). Stores template_id + variables. `dedupe_key` (NOT NULL,
  total unique per merchant — strengthened 29 Aug, T16 amendment) makes retries and
  reaper releases one-row. Status ladder is one stamped word driven by receipts; the
  monotonic walker never regresses a later state on out-of-order arrival. Ships
  UNPARTITIONED (T16 col 19 trail — partitioning returns when volume calls).
- **`send(send_token, message)`**: verifies the gate token names THIS message, invokes
  the channel adapter, records provider_message_id (wamid) and honest status. Free-form
  bodies are never mirrored anywhere — per ADR 0015 the thread linkage is a POINTER ROW
  in `conversation_message` (written by the conversations lane when it builds; the
  words live in the transcript, the send in the manifest). Adapters are imported ONLY
  inside send()'s module — CI rule 11 with red tests (as built, #1037).
- **Channel registries (as built, #1037)**: `providers/` holds `ADAPTERS` (the channel
  vocabulary — one adapter file + one registry line per channel) behind the send door;
  root `channels.py` holds `CHANNELS` metadata (today the gate's handle kind; W8's
  pacing and quality-tier defaults join as fields). Two registries because rule 11
  confines providers/ behind send.py — anything dispatch needs per channel must live
  outside the confined package. Pinned `ADAPTERS ⊆ CHANNELS`; a channel missing from
  either fails closed.
- **Provider package split — COMMITTED at the second adapter (Swaroop ruling,
  1 Sep 2026)**: one provider = one flat file is correct exactly as long as there is
  one provider. The PR that adds the SECOND adapter must ship the package shape for
  BOTH — `providers/<name>/` per provider, the concerns that today share whatsapp.py
  split by kind: `adapter.py` (the ChannelAdapter subclass — deliver/build/read),
  `classify.py` (the provider's error-code tables and outcome classification),
  `payload.py` (pure request-building utilities: recipient normalisation, parameter
  assembly). `providers/__init__.py` stays the single assembly point (registry
  unchanged, rule 11 unchanged — the confinement path doesn't move). Same law as
  record's `extractors/`: the split lands when the second entry arrives, and the
  mover defines the seam. A second adapter PR that stacks another 350-line flat file
  beside whatsapp.py is a MAJOR at review.
- **Dispatcher (ADR 0004)**: drain `status='queued'` with SKIP LOCKED, back off on
  429s, no transaction across HTTP. Simple queue; scale by replicas. As built: the
  suppression gate slice runs first (fail closed, own deadline), then send() — each
  may burn one `CRM_MESSAGE_SEND_TIMEOUT_SECONDS`, and the test suite pins
  `batch × 2 × timeout ≤ lease` so no dial drifts past the others.
- **Receipt walker**: per-wamid transitions; `cost_micros` filled from the provider's
  pricing object (THEIR claim, never our arithmetic); statuses also land as journey
  events.

## Do NOT

- **Never store a rendered template string.** Meta/DLT render; storing our simulation
  of their render is a lie waiting for an audit. Free-form body rides the mirror
  event into the timeline cache — that is the ONLY sanctioned copy.
- **Never call a provider outside send().** Not in a worker, not in a test helper that
  escapes, not "just this once" in the walker. One call site or the audit story dies.
- **No CHECK on `channel` or `connector_key`** — vocabulary in code (the 027 scar).
- **Don't let the dispatcher hold row locks during HTTP** — claim, release, call,
  record. The dedupe_key absorbs the crash window.
- **Don't copy delivery ticks anywhere** — manifest is the single authority; every
  other surface joins.
- **Don't retry around the gate.** A refused send completes per the caller's rules;
  re-asking the gate in a loop is an override with extra steps.
- **Don't put secrets in installation rows** or logs. The vault pointer exists so
  consoles and sweepers can read lifecycle without ever reading a token.
- **Blocked ≠ skipped**: if a send was proposed and refused, it MUST be a manifest row.
  Only pre-proposal deaths (workflow guards) live elsewhere.

## Scale & future-fit

- Throughput governance is Meta quality tiers, not channel counts — surface tier +
  per-second budget per pipe in capabilities; the dispatcher respects it.
- Scale-out path if measured: shard the drain by merchant or promote hot merchants to
  a Redis ready list — the manifest contract must not change.
- Route reads (binding → installation → vault) run PER SEND, uncached, on purpose:
  a paused pipe or rotated credential must bite the very next message. If route reads
  ever show in queue-lag, any cache is FAIL-CLOSED-CONSTRAINED — short TTL, never
  caching a refusal away; the gate itself is never cached, at any TTL (law).
- Voice adapter lands at the takeover as just another adapter behind send() — if that
  requires more than an adapter + a binding, this module drifted.

Refs: 04-connectivity.md (corpus) · ADR 0004 / 0009 / 0011 / 0015.
