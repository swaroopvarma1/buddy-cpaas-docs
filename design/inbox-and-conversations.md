# Team inbox & conversations — design record

**Status:** Draft for team review · **Date:** 2026-08-19 · Decisions: ADR 0012–0014.
The shared multi-teammate inbox over WhatsApp (Instagram fast-follow): customers message
the merchant; the AI agent answers first; humans watch, take over, and reply — all from
one account.

## Scope & sequencing (ADR 0012)

- Ships with **phase-2 WhatsApp inbound**. Instagram DM is the immediate fast-follow —
  by then it is a new door (`connector_key='instagram'`) + adapter + extractor;
  `crm.customer.igsid` and `channel_binding` already anticipate it. The inbox UI treats
  channel as an icon on the thread, so IG adds no UI rework.
- UI is designed and developed by Swaroop + Claude (architect-level direction, not
  pixel specs to the team). Realtime via SSE/WebSocket (chat SSE precedent).

## Interaction model (ADR 0013)

- **AI first, human takeover.** The channel-agnostic chat brain answers every inbound
  through a WhatsApp shell. The inbox is a live window onto agent conversations.
- **Takeover = assignment.** Threads are unassigned while the agent handles them.
  Whoever takes over becomes the assignee (exclusive — this is also the collision
  guard); the agent goes silent; journey `handled_by` flips agent -> human/both.
  Escalations (agent stuck, customer asks for a human, keyword rules) land in a shared
  **Needs attention** queue anyone can claim; managers can reassign. Zero routing
  config in v1; manager-routing and auto-assign are later options.
- Handback: the assignee can return the thread to the agent; thread goes unassigned.

## Storage (ADR 0014) — two new tables, chat tables reused as the cockpit

### crm.conversation — the thread
One row per (merchant, customer, channel) live thread: `status` (open · pending ·
resolved), `assignee_user_id` (NULL = agent-handled), `escalated` flag,
`last_inbound_at`, `last_message_at`, `bot_session_id` (pointer to the CURRENT
chat_session, if any — the voice_lead_id precedent), light jsonb assignment trail.
Laws applied: the WhatsApp 24h service window is COMPUTED (`last_inbound_at + 24h >
now()`) — expiry is a predicate, never stored. Presence, typing, and reply-collision
soft locks are Redis ephemera, never columns.

### crm.conversation_message — the thread timeline (a projection)
Written by a new spine consumer (the conversation projector). Three row kinds:
- **inbound** — decoded once from event_raw; provenance = event_raw id (+ wamid).
- **outbound** — a REFERENCE to the manifest row (never duplicated); status/read
  ticks live on the manifest and are joined, not copied.
- **note** — internal team notes. They live here because `customer_memory` is
  agent-only by corpus law; no console logic may read it.
Optional later: `conversation_read_state` per teammate (v1 ships a team-level unread
flag on the conversation row).

### Why NOT chat_session / chat_message as the store
They are the bot's **cockpit**, not the customer **ledger**:
1. Lifecycles differ — a session is a disposable, template-bound runtime episode
   (idle-swept, per-turn locked); a thread spans weeks and many sessions.
2. chat_session has no customer_id (predates crm.customer; widget rows are anonymous)
   — retrofitting couples buddy runtime tables into crm's spine, against the
   boundary-is-a-grant law.
3. chat_message.content_blocks is the LLM's replay history (tools, splices, healer
   output) — not the wire's customer-facing record; human replies/notes injected there
   would corrupt model replay.
4. No (event_raw | manifest) provenance or per-message delivery shape.
**Reuse that DOES happen:** the WhatsApp responder mints a chat_session per service
window and runs run_chat_turn unchanged; conversation points at it; on takeover the
session ends while the thread lives on.

## The reply path (no new machinery)

Human reply -> `may_contact()` (service purpose) -> inside the 24h window: free-form;
outside: template-only (send-time policy) -> `send()` -> manifest row with
`source_kind='human'` (one CHECK extension) -> the reply handler inserts its own
outbound reference row synchronously (read-your-writes); the mirrored event no-ops on
the partial unique. Every inbox reply is gate-checked and manifest-recorded.

## Amendments from the adversarial review (2026-08-20 — see ADR 0014 amended text)

- **Outbound rows carry a nullable, rebuildable `body` render-cache**, populated from a
  `message.queued` mirror event send() emits at manifest-insert (the manifest stores no
  rendered words by T16 law; the mirror carries the free-form body). Template sends:
  body NULL; console renders template_id + variables locally. The projector consumes
  event_raw topics ONLY (`message.inbound`, `message.queued`) — it never reads
  chat_message.
- **Takeover** = compare-and-set claim on assignee + honest END of the bot session;
  responder checks assignee before every send. **Handback** mints a fresh session whose
  first turn receives the human-era timeline slice via run_chat_turn's existing
  client-context mechanism. **Sweeper contract**: on `session_ended`, the responder
  mints a new session and updates `bot_session_id` in the same transaction.
- **Console "view bot reasoning"** deep-links read-only into the cockpit
  (chat_session/chat_message) via the journey-view read grant — never a write.
- First inbox migration also drops the `supported_channels` CHECK and adds the inbox
  list index; plan occurred_at partitioning for conversation_message if volume grows.
- Hygiene finding (not inbox scope): the 180s session-lock TTL is hardcoded in four
  modules — hoist to one constant.

## Open items for the build (design-doc level, no user decision needed)

- Escalation rule vocabulary (confidence threshold, keyword list, explicit
  ask-for-human intent) — start hardcoded, config later.
- Auto-resolve policy for silent threads (e.g. resolved after N days of no inbound).
- IG specifics at fast-follow: 7-day human-agent window vs WhatsApp's 24h; same
  computed-predicate pattern, different constant per channel (lives on
  channel_binding.capabilities).
