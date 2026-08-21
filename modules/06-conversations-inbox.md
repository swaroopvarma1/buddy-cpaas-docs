# Conversations & inbox — build guide (phase 3)

Owns: `crm.conversation` · `crm.conversation_message` · the conversation projector ·
takeover/handback · the responder's session contract. Diagram:
`../diagrams/06-conversations-inbox.html`. ADRs 0012–0015 (0014 adversarially verified).

## Build it like this

- **One thread row forever**: `UNIQUE (merchant_id, customer_id, channel)`; reopening
  flips status, never inserts. Assignee NULL = agent-handled; the 24h service window is
  the predicate `last_inbound_at + interval '24 hours' > now()` — computed at read,
  every time.
- **The projector consumes events ONLY**: `message.inbound` and `message.queued`
  (which carries the free-form outbound body — ADR 0015). Conversation upsert →
  timeline insert with partial uniques on `event_raw_id` / `message_id` making replay
  idempotent → SSE publish. Rebuild = truncate + replay.
- **Timeline rows are thin**: inbound = decoded once with provenance; outbound = a
  manifest REFERENCE plus a nullable, rebuildable body render-cache; notes =
  `author_user_id NOT NULL` + body. Ticks/status are JOINED from the manifest, never
  copied. Template sends: body NULL; the console renders template_id + variables.
- **Human replies are synchronous**: the reply handler runs gate → send() → inserts its
  own timeline reference row (read-your-writes); the mirrored event later no-ops on the
  partial unique.
- **Takeover is a compare-and-set**: `SET assignee_user_id=$u WHERE assignee_user_id
  IS NULL` — exclusivity IS the collision guard. Takeover honestly ENDS the bot
  session (its approvals die with it, by existing CASCADE design). A turn already
  mid-flight still lands its manifest row — an honest record, not a kill switch.
- **Handback mints a fresh session** whose first turn receives the human-era timeline
  slice through run_chat_turn's EXISTING client-context mechanism (the answer-nudge
  precedent). The bot process never queries crm tables; the responder composes context.
- **Session lifecycle contract**: on `session_ended` the responder mints a new session
  and updates `conversation.bot_session_id` in the same transaction.
- **Escalation**: agent-stuck / asks-for-human / keyword rules flip `escalated`; the
  Needs-attention queue is `WHERE escalated AND assignee IS NULL`, claimable by anyone.
- **Presence, typing, soft locks: Redis ephemera.** Unread is a team-level flag in v1.

## Do NOT

- **Never write chat_session/chat_message from crm code** — and never read them for the
  timeline. The ONE sanctioned read is the "bot reasoning" deep link, read-only,
  through the journey-view grant, sanitized. (The full why is ADR 0014: lifecycles,
  the single-writer idx lock, synthetic rows, the boundary law — adversarially
  verified; don't relitigate it in code review.)
- **Don't inject human replies or notes into the cockpit's history** — foreign rows
  starve the bot's capped replay window and pollute the dashboard preview. Handback
  context goes through the context mechanism, not the history table.
- **Don't store window state** ("in_session boolean") — it's the expiry-as-predicate
  law; a stored window lies within minutes.
- **No assignee bypass of the gate.** A human reply is a send like any other: purpose
  service, window policy applied at send time (free-form inside, template outside).
- **Don't build per-teammate read state in v1** — team-level unread ships first;
  per-user read receipts are a fast-follow table, not a v1 column.
- **Don't let the projector call the brain, or the brain query crm.** The responder is
  the only piece that faces both, and it only calls contracts.
- **Don't fan ticks into timeline-row updates** — one authority (manifest), one join.

## Scale & future-fit

- Inbox list = one index: `(merchant_id, status, last_inbound_at DESC)`. Needs-attention
  = partial index. If a list query needs a join to render, the preview/denorm columns
  on conversation are the fix — not a wider query.
- Partition conversation_message by occurred_at when volume approaches the manifest's.
- Instagram (P3+): a new door + adapter + a per-channel window constant read from
  `channel_binding.capabilities` (7-day human-agent window vs 24h) — if IG needs more
  than that, this module drifted.

Refs: inbox-and-conversations.md · ADR 0012 / 0013 / 0014 / 0015.
