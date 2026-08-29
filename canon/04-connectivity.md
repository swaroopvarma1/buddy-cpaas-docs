# Connectivity — doors, pipes, and the manifest

A door (installation) is one merchant's account on one connector. A pipe (binding) is one
endpoint under it — the real-world facts (numbers, senders) live per pipe. The manifest
(message) is one row per outbound attempt, and `send()` is the only code path that talks to a
provider.

### crm.connector_installation (T11) — 11 columns

One installation per merchant per connector — the lifecycle today’s credentials table completely lacks.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | UQ |  |
| 3 | `connector_key` | text | UQ | shopify · whatsapp · instagram · messenger · zendesk · juspay — validated against a dict in code: a new connector is a deploy, never a migration |
| 4 | `external_account_id` | text | UQ | WABA id, shop domain, Zendesk subdomain — the composite unique lets one merchant hold two shops on one connector |
| 5 | `display_label` | text |  | The connections screen’s name for the account — external ids are opaque |
| 6 | `credential_id` | uuid |  | → the EXISTING credential vault, reused — this table never holds a secret, it points, so consoles and sweepers read the lifecycle without ever being able to read a token. One credential row = one installation’s whole key BUNDLE (Shopify {api_key, api_secret, access_token} · WhatsApp {system_user_token, app_secret, verify_token}) — shape decided by connector_key, validated in code; multiple keys are fields in the bundle, never rows. Rotation with overlap = new bundle row + pointer flip, old bundle kept until cutover — reversible in one statement |
| 7 | `status` | text | CK | connecting · healthy · degraded · revoked · disabled. expired LEFT the vocabulary at the round — the T08 law applied here too: expiry is the predicate token_expires_at < now(), and a STORED expired needs a sweeper and lies between token-death and cron-run |
| 9 | `token_expires_at` | timestamptz |  | NULL = PERMANENT — Shopify offline tokens and Zendesk API tokens never die; Meta system-user tokens run ~60 days. T08’s expires_at semantics exactly: the predicate is (IS NULL OR > now()), and the proactive-refresh job watches only the non-NULL rows — the only thing between us and every Meta connector dying silently every 60 days |
| 11 | `last_event_at` | timestamptz |  | The traffic heartbeat — catches the silent killer: token valid, webhook subscription quietly dropped, every monitor green, zero traffic. No probe can fake it; it is stamped by arriving events themselves |
| 12 | `health_detail` | jsonb |  | The readable why + checked_at, written in the same probe transaction. status is the traffic light; this is the sentence under it — Meera’s door goes degraded at 3am, she opens the screen at 9, and it says "Meta revoked the token — reconnect" with a button, no logs read (the T20 last_error precedent). checked_at inside makes the green light believable: healthy, as of 4 minutes ago |
| 13 | `installed_at` | timestamptz |  | The birth |


**Wiring**
- One row per merchant per connector account. `credential_id` POINTS at the existing vault —
  never holds secret material; rotation = new bundle + pointer flip.
- Created by connect flows (Shopify OAuth → installation; WhatsApp signup) and by the
  credentials backfill (every live credential gets a door row).
- Health: a scheduled probe walks configured → authenticated → subscribed → heartbeat →
  healthy and writes `health_detail {level, checked_at, why}` — a why is mandatory below
  healthy.
- Inbound events are stamped with the merchant from the receiving credential — which is why
  every event has an owner by construction.

### crm.channel_binding (T12) — 8 columns

One binding per actual pipe — a number, a Page, a from-address.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | UQ |  |
| 3 | `channel` | text | UQ | Deliberately no CHECK — migration 027’s CHECK is exactly why a channel cannot be added today |
| 4 | `installation_id` | uuid | FK | → connector_installation |
| 5 | `address` | text | UQ | phone_number_id, Page id, IG account, from-address — the API handle the adapter dials, and inbound’s which-merchant lookup: Meta’s webhook says only "a message arrived on phone_number_id 1067…", and this unique index answers whose it is. The human-readable number lives in capabilities |
| 8 | `capabilities` | jsonb |  | Rich media? buttons? session window? read receipts? Queryable data that clamps the agent’s tool schemas before composition (§06) — voice never offers a button. Also carries the display number, and the purpose LANE where the lane is a choice rather than a prefix: offers@ is promotional because the merchant said so, and the dispatcher reads it here |
| 9 | `is_primary` | boolean |  | The send path’s default when no binding is specified — partial unique (merchant_id, channel) WHERE is_primary |
| 10 | `status` | text | CK | active · paused · retired — KEPT through his knife because it is the only column that is both a gate operand and a tombstone. paused: the marketing number’s quality goes red, Meera pauses the pipe, and sends through it REFUSE (invariant 6 — fail closed; without this the dispatcher has no operand and "paused" lives in someone’s head). retired: she gives the number back in January and the recycle may hand it to ANOTHER merchant within months — the row cannot be deleted because thousands of manifest rows stamp binding_id at it and "what number did this August send leave on" must answer forever. paused expects the pipe back; retired has surrendered it — only one is reversible by a click |


**Wiring**
- One row per endpoint (WhatsApp number, sender address) under a door. `address` is the
  unique inbound lookup: WhatsApp webhook → binding by receiving number → merchant. One
  number serves exactly one merchant.
- Capabilities (concurrency, buttons) are per-pipe jsonb. Voice numbers unify here at the
  voice takeover (clairvoyance `telephony_numbers` is the proto-binding); the available-number
  pool stays a platform-side ops table.

### crm.template (T23) — 19 columns

The channel template registry — sealed 29 Aug 2026, the C7 table the map had left
unnamed. One registry for every channel that pre-registers message shapes:
WhatsApp (WABA templates) first, SMS-DLT (the TRAI audit) second, email later.
`T16.template_id` stores the name this table registers — "one column; the channel
decides the registry."

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | UQ | Tenancy; first in the natural key |
| 3 | `channel` | text | UQ | whatsapp · sms · … — NO CHECK (027 scar: provider vocabulary lives in code) |
| 4 | `provider_account_ref` | text |  | The WABA (or DLT entity) that owns the template — templates are namespaced per provider account, not per merchant. Becomes `installation_id FK → T11` when C1 lands (trail) |
| 5 | `name` | text | UQ | What T16.template_id stores at send time |
| 6 | `language` | text | UQ | Meta's real key is (WABA, name, language) — the same name exists once per language; the send path picks by name+language |
| 7 | `provider_template_id` | text | UQ | Meta's stable id. Globally unique per provider → partial unique ALONE (the wamid exception to merchant-first) |
| 8 | `category` | text |  | MARKETING · UTILITY · AUTHENTICATION — THEIRS, current. NO CHECK: Meta has renamed categories before. Determines billing class |
| 9 | `submitted_category` | text |  | What WE asked for. Meta recategorizes silently and the price changes with it — the diff between 8 and 9 is money made visible |
| 10 | `category_updated_at` | timestamptz |  | When Meta last moved it |
| 11 | `components` | jsonb |  | NOT NULL. HEADER/BODY/FOOTER/BUTTONS + variables + example values — THEIR registered structure, verbatim (store the letter). Never a rendered message string (that law lives on T16) |
| 12 | `status` | text | IX | draft · submitted · pending · approved · rejected · paused · deleted (+ whatever Meta adds). NO CHECK — vocabulary dictionary in code. Editing an approved template puts the SAME row back to pending (Meta re-reviews in place; history = template.status events in the spine, replayable) |
| 13 | `status_updated_at` | timestamptz |  |  |
| 14 | `rejection_reason` | text |  | Meta's reason VERBATIM — the U4 console surfaces it word for word (console-ui law) |
| 15 | `quality` | text |  | GREEN · YELLOW · RED · UNKNOWN — theirs, changes over time, no CHECK |
| 16 | `quality_updated_at` | timestamptz |  |  |
| 17 | `last_synced_at` | timestamptz |  | C7's periodic Tech-Provider full sync — the drift healer |
| 18 | `created_at` | timestamptz |  |  |
| 19 | `updated_at` | timestamptz |  | Touch trigger (rules: every updated_at gets one) |

Natural key: `UNIQUE (merchant_id, channel, name, language)`.

**Wiring**
- Written by: the U4 console (create/edit drafts) → C7 Tech Provider API (submit to
  Meta). The UI never talks to Meta directly.
- Status updates arrive THROUGH THE SPINE: Meta's `template.status` webhooks land in
  `event_raw` raw (T13 already names the topic), and a connectivity-owned
  template-status consumer updates this registry — replayable when Meta's payload
  surprises us. Plus the periodic full sync (17) healing drift.
- Read by: U4 (list/detail with status + verbatim rejection + quality), `send()`
  (template lookup by name+language at send time), and the gate indirectly
  (category informs purpose mapping).
- Owner: connectivity. Physical `crm_template`.

### crm.message (T16) — 24 columns

The universal outbound row. A blocked message is still a row.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | IX |  |
| 3 | `customer_id` | uuid | FK·IX | Trail (29 Aug 2026, PR #1031): P1 ships a COMPOSITE tenant-pinned FK (merchant_id, customer_id) → crm_customer — we are unpartitioned (see created_at trail), so the FK costs little and stops a wrong merchant_id filing a message against another tenant's customer (table self-defense). The original "No FK — partitioned, high volume" ruling returns WHEN partitioning lands; stamped at write by resolve(), never joined at read — the timeline rule |
| 4 | `sent_to_address` | text |  | The exact number/address it went to, as it stood at send time — consent’s address twin: the audit must say WHERE, and the customer row is alive while this ledger is frozen |
| 5 | `channel` | text |  | Stamped, not derived through binding_id — the gate refuses BEFORE a pipe is picked, so binding_id is NULL on exactly the rows compliance cares about most, and permission is per-channel |
| 6 | `binding_id` | uuid | FK | → channel_binding — which pipe it left on. NULL on blocked rows: no pipe was ever picked |
| 7 | `source_kind` | text |  | broadcast \| workflow \| agent \| transactional. Trail (29 Aug 2026): CK REMOVED per the 027 scar — vocabulary lives in code; the validating dictionary MUST land with the first producer (open obligation, PR #1031) |
| 8 | `source_id` | uuid |  | What caused this send: broadcast id · workflow_enrolment id · chat_session id (agent) · event_raw id (transactional) |
| 9 | `purpose_key` | text |  | What we claimed it was for — checked against the grant, auditable forever |
| 10 | `template_id` | text |  | The registered template this send used — WABA template name on WhatsApp (quality tracking), DLT template id on SMS (the TRAI audit). One column; the channel decides the registry. Settles the T08 eviction debt with zero new columns |
| 11 | `variables` | jsonb |  | What we actually posted to the provider: the values filled into the registered template — Meta and DLT render the final string, we never send one, so storing a "rendered" copy would store our own simulation of their render. Free-text agent sends carry no variables; the words are the transcript turn, one link away via source_id |
| 12 | `status` | text | CK·IX | queued · **sending** · blocked · accepted · sent · delivered · read · failed · dead. `sending` added 29 Aug 2026 (PR #1031): the in-flight state a claim stamps — required because the claim must COMMIT before the provider call (no transaction across HTTP), so "claimed" must be visible in the row, not in a lock. failed = the provider refused; dead = we ran out of retries — a merchant asking why nothing arrived needs to know which. The timestamps are the evidence, status is the stamped word |
| 13 | `reason` | text |  | One column, meaning follows status — the fold made three times now. blocked → the gate’s no: no_consent, quiet_hours, frequency_cap. failed \| dead → the provider’s code: 131049. A row is refused by us or failed by the wire, never both — two columns meant one was always NULL |
| 14 | `provider_message_id` | text | UQ | How an inbound receipt finds this row — the wamid; the whole metrics chain hangs on this UNIQUE |
| 16 | `attempt` | smallint |  | Position on the retry ladder |
| 17 | `cost_micros` | bigint |  | THEIR claim, never our arithmetic — filled from the pricing object on the provider’s receipt webhook (category, billable), by the walker whose hand is already on the row. On WhatsApp Meta bills per conversation: cost lands on the opening row, NULL elsewhere. Safe cut if ever wrong — the pricing object lives verbatim in event_raw, one replay recovers it |
| 18 | `decision_id` | bigint |  | → decision_log — the gate decision that authorised it, reasoning included. This table’s last_event_id |
| 19 | `created_at` | timestamptz | PART | RANGE partition key. Trail (29 Aug 2026): P1 ships UNPARTITIONED — a partitioned table's uniques must include the partition key, which would break the dedupe index; same ruling as T13. Partition when volume calls |
| 20 | `sent_at` | timestamptz |  |  |
| 21 | `delivered_at` | timestamptz |  |  |
| 22 | `read_at` | timestamptz |  | Only channels with read receipts fill it — WhatsApp’s ticks free from Meta, email via the pixel event |
| 23 | `dedupe_key` | text | UQ | NEW 17 Aug; **strengthened 29 Aug 2026 (PR #1031): NOT NULL, total UNIQUE (merchant_id, dedupe_key)** — an omittable protection is a protection silently skipped, so every producer must name the logical send (triggering event · enrolment:node · broadcast recipient). This index — not the claim or lease — is what suppresses a duplicate ROW when at-least-once redelivers; it cannot stop one row reaching the provider twice (that risk is bounded by the claim lease) |
| 24 | `claimed_at` | timestamptz |  | NEW 29 Aug 2026 (PR #1031): set while a worker holds the row — the persistent claim marker the no-txn-across-HTTP law requires (a SKIP LOCKED lock dies with its transaction; the provider call outlives it). Doubles as the lease: the stale sweep requeues `sending` rows whose claimed_at expired, and reclaimed ids are logged BY NAME (a double-send investigation starts there) |
| 25 | `next_attempt_at` | timestamptz |  | NEW 29 Aug 2026 (PR #1031): when the row may next be tried — now() at birth, pushed forward by retry backoff (exponential + jitter). The queue index filters AND orders on it, so a retry waits its turn instead of jumping ahead on created_at. The gate's quiet-hours deferral writes its next_allowed_at here too |


**Wiring**
- One row per attempt; **blocked attempts are rows too** (reason, no provider call). Stores
  `template_id` + variables — never a rendered string; the provider renders.
- `send(send_token, message_id)`: verifies and consumes the gate token, then invokes the
  channel adapter (WhatsApp Graph API; voice = wrapper over the lead machine), records
  `provider_message_id` (wamid) and honest status. Adapters are imported only inside send()'s
  module — the single call site is grep-enforceable.
- The dispatcher's claim is ONE self-locking statement (as built, PR #1031):
  `UPDATE … SET status='sending', claimed_at=now(), attempt=attempt+1 WHERE id IN
  (SELECT … WHERE status='queued' AND next_attempt_at<=now() ORDER BY next_attempt_at
  LIMIT n FOR UPDATE SKIP LOCKED) RETURNING …` — the claim commits BEFORE the provider
  call (never a transaction across HTTP), attempt burns AT claim so a worker-killing
  message cannot retry forever, and outcome writes are guarded `AND status='sending'`
  so a stale worker's result is discarded. Stale `sending` rows are requeued by the
  claimed_at sweep (this is the dispatch pattern; T20's wake_at-push lease remains the
  walker's — two sealed claim styles, per worker-runtime.md). Backs off on 429s with
  exponential+jittered `next_attempt_at`; `dedupe_key` makes producer retries one-row;
  delivery is at-least-once, bounded by the lease.
- Status ladder driven by provider webhooks (per-wamid transitions; out-of-order arrivals
  must not regress a later state). Statuses also land as journey events.
- Written by: the workflow walker (every send node). Read by: journey view, reports,
  why-didn't-it-send.
