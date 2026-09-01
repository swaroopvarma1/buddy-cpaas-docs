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
  reason. **Status path (ruled 2 Sep 2026, #1038 — amended the same day)**: webhooks
  are the ONLY path — the `template.status` spine consumer (one `register_consumer`
  line + the consumer; goes live when record's Meta ingress bay lands at C6).
  **No sync code exists**: not a timer, not a route, not a seed at onboarding. The
  arithmetic that killed the timer: at 1,000 merchants × 100 templates an hourly pass
  is ~1,000 Graph calls + ~100k row writes per hour for ~zero information, per-pod
  timers multiply it, and a sync inside the dispatcher's claim stalls sends. The
  "bridge until webhooks" argument for a reconcile route died with "we are merging,
  not releasing" — between the template PR and the webhook PR a submitted template
  simply cannot reach `approved`, which is correct while nothing is live. Every
  registry drift arrives as an event: approval/rejection/pause, deletion
  (`PENDING_DELETION` / `DELETED`), category (the money one), quality. The
  crashed-submit resume path lives in the CONSUMER: a status event whose id matches
  no row but whose (WABA, name, language) matches a `submitting` row with a NULL
  provider id stamps it. Named follow-up, not code: "import a WABA's existing
  templates" as an explicit one-shot action if a pilot merchant ever arrives with
  approved templates — still never a clock. Provider quirks (Meta's uppercase
  statuses, edit-in-place vs re-register, delete-by-name nuking every language) are
  normalised INSIDE the provider's template face, never in the generic registry
  file.
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
- **Provider package split — COMMITTED; trigger FIRED by #1038 (Swaroop rulings
  1 Sep + 2 Sep 2026)**: one provider = one flat file is correct exactly as long as
  there is one provider with ONE face. The trigger was written as "the second
  adapter"; #1038 proved the seam arrives just as surely as a second FACE of the same
  provider (onboarding + templates + a Graph client beside the send adapter — four
  WhatsApp-only flat files at module root). Ruled: the trigger is the second adapter
  OR the first non-send face. Target shape, for every provider from here:
  `providers/<name>/` = `adapter.py` (the ChannelAdapter — deliver/build/read) ·
  `classify.py` (error-code tables, outcome classification) · `payload.py` (pure
  request-building) · `onboard.py` (the `ConnectorOnboarder` port: gather →
  `OnboardResult {external_account_id, address, bundle, token_expires_at, health}`) ·
  `templates.py` (the `TemplateProvider` port: submit/edit/retire/list → normalized
  `ProviderTemplateState`; `edits_in_place` flag). Vendor-shared transport is its own
  package — `providers/meta/graph.py`: endpoint builder, ONE `_call`, error fold,
  retryable throttles; Instagram/Messenger reuse it. Ports live in `providers/base.py`;
  `providers/__init__.py` stays the single assembly point for `ADAPTERS`. **Rule 11
  becomes face-precise**: the assembly and `providers/<x>/adapter.py` are imported
  only by send.py (unchanged intent); `providers/<x>/onboard.py` / `templates.py`
  only by root `connectors.py`. Generic logic (`onboarding.py`, `templates.py`) never
  names a provider — it dispatches through `CONNECTORS`. A provider PR that parks
  vendor code at module root to dodge rule 11 is a MAJOR at review.
- **`CONNECTORS` registry (canon T11 col 3's "validated against a dict in code";
  ruled 2 Sep 2026, #1038)**: root `connectors.py` (beside `channels.py`, for the
  same rule-11 reason) holds `connector_key → ConnectorSpec(channel | None,
  onboarder, templates | None, request_model)`. Routes are connector-agnostic:
  `POST /connectors/{connector_key}/onboard` (body validated by the spec's request
  model; unknown key = 404 — the dict IS the vocabulary), `/connectors/installations`,
  templates under `/templates` (the row's `channel` decides the provider). Pins: every
  spec with a channel has it in `CHANNELS`; `create_draft` refuses a channel that is
  not a key. The generic onboarding flow is fixed: merchant lookup → `spec.onboarder.
  gather()` → credential upsert (name `{connector_key}:{merchant_id}:{account}`) →
  atom (installation + primary binding), status derived from the health level
  (`subscribed → healthy`, below → `degraded`; the light never contradicts the
  sentence).
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
