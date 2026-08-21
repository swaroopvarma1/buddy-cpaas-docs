# 0014 — Inbox storage: crm.conversation + crm.conversation_message; chat tables stay the bot's cockpit

**Status:** Accepted (amended 2026-08-20 after adversarial review) · **Date:** 2026-08-19

## Context
The inbox needs a thread model and a queryable customer-facing timeline. Inbound
messages today exist only as raw webhooks in event_raw. The reuse question ("why not
chat_session/chat_message?") was challenged and put through an adversarial review:
two advocate agents argued each side against the actual code; a neutral judge verified
every load-bearing claim in the repo and ruled. **Verdict: new tables — decision
confirmed, rationale corrected, four amendments adopted.**

## Decision
Two new crm tables; chat tables deliberately NOT extended:

- **`crm.conversation`** — one row per (merchant, customer, channel) thread, forever
  (`UNIQUE (merchant_id, customer_id, channel)`; reopen flips status): status
  (open · pending · resolved), assignee (NULL = agent-handled), escalated, unread
  (team-level, v1), `bot_session_id` pointer (no FK — voice_lead_id precedent),
  last_inbound_at (24h window = predicate, never stored), preview, assignment trail.
- **`crm.conversation_message`** — the timeline, fed by event_raw topics ONLY:
  - **inbound**: decoded once from event_raw (provenance = event id + wamid;
    partial UNIQUE on event_raw_id makes replay idempotent)
  - **outbound**: a reference to the manifest row (partial UNIQUE on message_id)
    **plus a nullable, rebuildable `body` render-cache** — populated from the
    `message.queued` mirror event that send() emits at manifest-insert. The mirror
    carries the free-form body because the manifest, by T16 law, stores no rendered
    words. Template sends: body NULL; the console renders template_id + variables
    locally (a display burden reuse carries equally — its own concession C4).
  - **note**: internal team notes (author_user_id NOT NULL); customer_memory is
    agent-only by law and cannot hold them.

`chat_session`/`chat_message` are reused **as the bot's runtime cockpit**: the WhatsApp
responder mints a session per service window and runs run_chat_turn unchanged;
`conversation.bot_session_id` points at it. Console "view bot reasoning" reads the
cockpit **read-only** through the grant the journey view already has (07-wiring:66) —
recovering reuse's genuine bot-internals advantage without a write.

## Why not reuse chat tables as the store (verified rationale)
1. **Boundary law is decisive.** Write-ownership is per TABLE ("one squad writes each
   table", 07-wiring:15) — column grants cannot split a row INSERT. chat_session's
   `reseller_id NOT NULL` + nullable `merchant_id` (migration 027:35-36) are unfixable
   against crm tenancy canon while the widget shares the rows. 07-wiring:66 already
   legislated this exact question: chat tables "stay — journey view reads in place;
   new turns also stamp event_raw (forward-only)" — read grants yes, extension no.
2. **Lifecycles are enforced, not conventional.** ENDED sessions refuse turns
   (turn_core.py:194-199); the idle sweeper ends sessions and triggers downstream
   consumers; chat_message/tool_approvals/turn_metrics CASCADE on session delete.
   A note on a resolved three-week-old thread has no valid parent row.
3. **chat_message is single-writer by construction.** PK (session_id, idx) with
   idx = MAX+1, correct only under the per-session Redis turn lock (TTL 180s,
   hardcoded in four modules); console writers would contend a lock sized for LLM
   round-trips or race the allocation.
4. **"Full context for free" is false** — verified beyond the original rationale:
   the bot writes synthetic role=user rows the customer never typed and
   content=NULL assistant rows (UI-only/internal turns); even the widget must
   sanitize via _sanitize_messages_for_widget before display. A faithful human
   timeline off chat_message needs per-read block-level filtering of a foreign
   module's replay log.
5. **Corrected claim (the review's finding):** the earlier "notes corrupt LLM replay"
   rationale was wrong as stated — replay whitelists roles (turn_core.py:250, 340).
   The real problems: the replay cap counts rows BEFORE the role filter (foreign rows
   starve the bot's memory window), every reader assumes the two-role enum (the
   dashboard preview subquery would surface an internal note), and handback would
   force foreign rows back INTO replay — reuse defeats its own shield.
6. **Precedent on these exact tables:** the last one-row-reuse (voice_lead_id, 030)
   shipped with recordings disabled (widget/handlers.py:8-10); 027's
   supported_channels CHECK is a named ground bug already ordered dropped.

## Amendments adopted (from the review)
- **Handback specified:** takeover = compare-and-set claim
  (`SET assignee_user_id=$u WHERE assignee_user_id IS NULL`) + honest END of the bot
  session; responder checks assignee before every send; a turn mid-flight still lands
  its manifest row (honest record). Handback mints a fresh session whose first turn
  receives the human-era timeline slice via run_chat_turn's existing client-context
  mechanism (context_placement; answer-nudge precedent). The bot process never queries
  crm tables; the responder composes context.
- **Sweeper contract:** responder treats `session_ended` as "mint new session, update
  bot_session_id in the same transaction."
- Human replies insert their reference row synchronously (read-your-writes); the
  mirrored event later no-ops on the partial unique.
- First migration also drops the supported_channels CHECK and adds the inbox index
  (merchant_id, status, last_inbound_at DESC); partition conversation_message by
  occurred_at if volume approaches crm.message's.

## Accepted costs (recorded honestly)
- The projector is real machinery with eventual consistency for bot/inbound rows.
- Inbound text is decoded once out of event_raw into the timeline (storage noise).
- Incidental review finding, not inbox scope: the 180s session-lock TTL is hardcoded
  in four modules ("must stay in sync") — hoist to one constant; goes on the hygiene
  list.
