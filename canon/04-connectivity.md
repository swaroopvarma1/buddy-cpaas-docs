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

### crm.message (T16) — 22 columns

The universal outbound row. A blocked message is still a row.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | IX |  |
| 3 | `customer_id` | uuid | IX | No FK — partitioned, high volume. Stamped at write by resolve(), never joined at read — the timeline rule |
| 4 | `sent_to_address` | text |  | The exact number/address it went to, as it stood at send time — consent’s address twin: the audit must say WHERE, and the customer row is alive while this ledger is frozen |
| 5 | `channel` | text |  | Stamped, not derived through binding_id — the gate refuses BEFORE a pipe is picked, so binding_id is NULL on exactly the rows compliance cares about most, and permission is per-channel |
| 6 | `binding_id` | uuid | FK | → channel_binding — which pipe it left on. NULL on blocked rows: no pipe was ever picked |
| 7 | `source_kind` | text | CK | broadcast \| workflow \| agent \| transactional |
| 8 | `source_id` | uuid |  | What caused this send: broadcast id · workflow_enrolment id · chat_session id (agent) · event_raw id (transactional) |
| 9 | `purpose_key` | text |  | What we claimed it was for — checked against the grant, auditable forever |
| 10 | `template_id` | text |  | The registered template this send used — WABA template name on WhatsApp (quality tracking), DLT template id on SMS (the TRAI audit). One column; the channel decides the registry. Settles the T08 eviction debt with zero new columns |
| 11 | `variables` | jsonb |  | What we actually posted to the provider: the values filled into the registered template — Meta and DLT render the final string, we never send one, so storing a "rendered" copy would store our own simulation of their render. Free-text agent sends carry no variables; the words are the transcript turn, one link away via source_id |
| 12 | `status` | text | CK·IX | queued · blocked · accepted · sent · delivered · read · failed · dead. The winner materialised: the timestamps are the evidence, status is the stamped word — the dispatch queue reads it, and the monotonic receipt walker (Meta delivers out of order) needs one comparable ordinal, never four NULL-checks |
| 13 | `reason` | text |  | One column, meaning follows status — the fold made three times now. blocked → the gate’s no: no_consent, quiet_hours, frequency_cap. failed \| dead → the provider’s code: 131049. A row is refused by us or failed by the wire, never both — two columns meant one was always NULL |
| 14 | `provider_message_id` | text | UQ | How an inbound receipt finds this row — the wamid; the whole metrics chain hangs on this UNIQUE |
| 16 | `attempt` | smallint |  | Position on the retry ladder |
| 17 | `cost_micros` | bigint |  | THEIR claim, never our arithmetic — filled from the pricing object on the provider’s receipt webhook (category, billable), by the walker whose hand is already on the row. On WhatsApp Meta bills per conversation: cost lands on the opening row, NULL elsewhere. Safe cut if ever wrong — the pricing object lives verbatim in event_raw, one replay recovers it |
| 18 | `decision_id` | bigint |  | → decision_log — the gate decision that authorised it, reasoning included. This table’s last_event_id |
| 19 | `created_at` | timestamptz | PART | RANGE partition key |
| 20 | `sent_at` | timestamptz |  |  |
| 21 | `delivered_at` | timestamptz |  |  |
| 22 | `read_at` | timestamptz |  | Only channels with read receipts fill it — WhatsApp’s ticks free from Meta, email via the pixel event |
| 23 | `dedupe_key` | text | UQ | NEW 17 Aug, from the research pass: nullable, UNIQUE where not null — the pre-send idempotency the manifest lacked. Workflow sends key on (enrollment, node, attempt-generation); transactional on their triggering event; broadcasts already had T18’s. n8n’s comment, verbatim: this index — not the scheduler’s claim, lease, or epoch fencing — is what suppresses a duplicate effect when at-least-once delivery redelivers. The industry trend is away from advisory locks toward exactly this partial unique index |


**Wiring**
- One row per attempt; **blocked attempts are rows too** (reason, no provider call). Stores
  `template_id` + variables — never a rendered string; the provider renders.
- `send(send_token, message_id)`: verifies and consumes the gate token, then invokes the
  channel adapter (WhatsApp Graph API; voice = wrapper over the lead machine), records
  `provider_message_id` (wamid) and honest status. Adapters are imported only inside send()'s
  module — the single call site is grep-enforceable.
- The dispatcher drains `WHERE status='queued'` (partial index) with `FOR UPDATE SKIP
  LOCKED`, backs off on provider 429s, never holds a transaction across HTTP; `dedupe_key`
  (UNIQUE where not null) makes retries and reaper releases exactly-once.
- Status ladder driven by provider webhooks (per-wamid transitions; out-of-order arrivals
  must not regress a later state). Statuses also land as journey events.
- Written by: the workflow walker (every send node). Read by: journey view, reports,
  why-didn't-it-send.
