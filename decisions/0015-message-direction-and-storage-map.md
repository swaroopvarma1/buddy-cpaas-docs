# 0015 — The message direction & storage map (who writes what, when)

**Status:** Accepted · **Date:** 2026-08-21

## Context
Follow-up clarifications to ADR 0014 that deserve their own record, because the team
will ask the same questions: is crm.message inbound too? and where does a conversation
live when a human (not the AI) is handling it? Vocabulary: the **agent** is the AI;
humans in the inbox are **teammates**.

## Decision

**Direction map — fixed:**
- **`crm.message` is outbound-only, forever.** One row per send *attempt* (blocked
  attempts included). Every column is about a send: gate decision, purpose, template +
  variables, retry attempt, dedupe_key, cost, delivery/read ticks. Inbound never
  touches it.
- **Inbound has exactly two stops:** `event_raw` (raw fact, always) →
  `crm.conversation_message` (decoded timeline row). Plus, only while the AI is
  driving, it is fed into the bot's session memory.

**Who-uses-what over a thread's life:**
- **AI handling (default):** inbound → event_raw → conversation_message AND into the
  bot's chat_session/chat_message; bot replies via gate → send() → crm.message →
  pointer row in conversation_message.
- **Teammate handling (after takeover):** the bot's chat_session is ENDED; chat tables
  are not involved at all. Inbound → event_raw → conversation_message. Human replies
  via gate → send() → crm.message → pointer row. Notes = conversation_message rows.
  A human-handled conversation lives ENTIRELY in crm tables.
- **Handback:** a fresh chat_session is minted, `conversation.bot_session_id` updated
  in the same transaction, and the responder injects the human-era timeline slice as
  opening context. The bot never reads crm tables itself.

**The one-line rule:** the thread always lives in `crm.conversation` /
`conversation_message` no matter who is talking; `chat_session` comes and goes as the
AI picks up and puts down the phone.

## Consequences
- No schema implications beyond ADR 0014 — this record fixes vocabulary and flow so
  the inbox, responder, and console teams share one mental model.
- Any future channel (Instagram, RCS…) inherits this map unchanged.
