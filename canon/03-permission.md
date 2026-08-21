# Permission — the ledger, the state, the gate, the diary

Two stores, never one: the ledger answers "prove she agreed"; the state answers "may I send
right now". They commit in the same transaction and can never disagree. Every verdict —
allow or refuse — lands in the decision log.

### crm.consent_event (T07) — 9 columns

Append-only — enforced by REVOKE UPDATE, DELETE and a trigger that refuses both, not by intentions. Answers "prove she agreed" — a legal question, possibly years later.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | IX |  |
| 3 | `customer_id` | uuid | FK·IX | → customer |
| 4 | `address` | text |  | The handle the consent was collected against. Without it a grant belongs to a person rather than a way of reaching them — when +91-98xxx is reassigned after 180 days, this column is how the new holder does not inherit Priya’s opt-in. Add it later and the answer is gone for every row already written |
| 5 | `event_type` | text | CK | REQUEST · GRANT · WITHDRAW · IMPORT · CONFIRM. EXPIRE retired: expiry is arithmetic on the state (expires_at < now()), and no human acted — a cron must not manufacture ledger rows |
| 6 | `channel` | text | IX |  |
| 7 | `purpose_key` | text | CK·IX | Dotted — marketing.promotional.winback. CHECK against a hardcoded list; a purpose table waits for custom purposes |
| 9 | `occurred_at` | timestamptz | IX | When the human acted. DEFAULT now() — for a live event the act and the write are the same moment; an import supplies the claimed date. The receipt clock lives where receiving happens: event_raw.received_at, one join through artifact_ref |
| 15 | `artifact_ref` | text |  | ONE pointer to the primary evidence: event_raw id, transcript span, form id, DLT receipt, CSV row. The receipt clock, the channel’s own ids and the words she saw all live behind this pointer — the ledger asserts, the artifact proves |


**Wiring**
- Physically append-only: INSERT-only grants + a trigger rejecting UPDATE/DELETE. No hash
  chain — grants + trigger are the enforcement.
- Single writer: `record_consent(event)` — validates channel/purpose vocabulary, enforces the
  evidence ladder (an IMPORT may never claim above the lowest class), resolves the exact
  notice text shown to `notice_sha256` (texts kept in a small reference table keyed by hash).
- Fed by: WhatsApp STOP/START keywords (via event topics), console consent capture, imports.

### crm.consent_state (T08) — 7 columns

The resolved cache, read on every send. Answers "may I send right now", thousands of times a minute.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `merchant_id` | text | PK |  |
| 2 | `customer_id` | uuid | PK·FK | → customer · CASCADE |
| 3 | `channel` | text | PK·CK | whatsapp · sms · email · voice · rcs · push · instagram · messenger |
| 4 | `purpose_key` | text | PK·CK | One column; the root is split_part(purpose_key, ’.’, 1) |
| 6 | `status` | text | CK·IX | granted · withdrawn · prohibited · pending_confirm — four things that were done, never things that elapsed. Expired is a predicate, not a status: a stored expired needs a sweeper, and lies between grant-expiry and cron-run. The negative is stored, not inferred — "no row" and "row saying no" are different answers, and only one is fixable by asking |
| 10 | `expires_at` | timestamptz | IX | The one clock a status may carry, read as a predicate on every gate check — what runs out depends on the status. granted → permission ends (India promotional: ~7 days). withdrawn → the re-ask embargo lifts (India opt-out: ~90 days of not nagging; a re-request is a send through the gate like any other). pending_confirm → the confirm link dies. The gate asks: status = granted AND (expires_at IS NULL OR expires_at > now()) |
| 13 | `last_event_id` | uuid | FK | → consent_event — which event produced this row. The state asserts, the ledger proves: granted_at and withdrawn_at are that event’s occurred_at, one PK join away |


**Wiring**
- One row per `(merchant, customer, channel, purpose)`, rewritten inside `record_consent`'s
  transaction. Resolver: most-specific wins → strictest wins on tie → fallback *prohibited*.
  IMPORT resolves to granted with capped evidence.
- One clock (`expires_at`); meaning depends on status (granted → permission ends · withdrawn
  → re-ask embargo lifts · pending_confirm → confirm link dies). Expiry is ALWAYS read as a
  predicate `expires_at < now()` — never swept into a stored status.
- Read by the gate as one indexed probe.

### crm.decision_log (T14) — 6 columns

Every automated decision, written the only moment it could be — the verdict and the ink. The one genuinely unreconstructable table: the accountability bill the inversion runs up, paid in rows.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | bigserial | PK | The page number — handed out by the database (INSERT … RETURNING id), stamped onto the manifest row in the same transaction so the staple never dangles. Meaningless, monotonic, never reused; gaps from aborted transactions are normal |
| 2 | `merchant_id` | text |  |  |
| 3 | `customer_id` | uuid |  | No FK — partitioned and high-volume. On a merge: the survivor; both parties live in the ink |
| 4 | `decision_kind` | text | CK | send_or_hold · identity_merge — the two deciders that exist. channel_choice and offer return WITH their deciders: new values, never migrations |
| 5 | `chosen` | jsonb | CK | The ink — the decider’s working state, dumped honestly. verdict always (CHECK chosen ? ’verdict’), reason on a refusal, checks as whatever this decision’s ladder actually evaluated, config knobs frozen because they are mutable at home (on_cap, the version of the code that decided). No format: nothing mechanical ever reads the ink — aggregates go to the manifest and the state tables; the only reader is eyes on an audit day, and eyes read any honest shape |
| 9 | `decided_at` | timestamptz | PART | RANGE partition key — retention is partition drop, horizon a counsel number. A pruned page’s pointer on the manifest survives as a tombstone: the send still provably went through the process |


**Wiring**
- One row per gate verdict — allow AND refuse — with a jsonb snapshot of the inputs that
  produced it. Identity merges also log here (`decision_kind='identity_merge'`). Monthly
  partitions. The one genuinely unreconstructable table: written from the very first gated
  send.
- Read by: the why-didn't-it-send screen (per-recipient reasons), audits, attribution.

## The gate — `may_contact(merchant, customer, channel, purpose)`

Inputs, all mandatory: consent_state probe · platform.identity suppression boolean ·
frequency cap (Redis rolling counter per identity+channel+day) · quiet hours evaluated in the
CUSTOMER's timezone (unknown timezone → no). **Any missing, NULL, erroring or unknown input →
verdict no, with the honest reason.** There is no override parameter.

On yes: mints a single-use send token (short TTL, bound to customer+channel+purpose) that
`send()` requires and consumes. Every verdict writes a T14 row. Built and owned by a pair of
engineers; never split — both own the entire predicate.
