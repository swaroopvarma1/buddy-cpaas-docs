# 0013 — Inbox interaction model: AI first; takeover IS assignment

**Status:** Accepted · **Date:** 2026-08-19

## Context
Who answers an inbound WhatsApp message by default, and how threads get owned by
teammates — the two decisions that define conversation states, the escalation queue,
and the phase-2 responder.

## Decision
1. **AI first, human takeover.** The channel-agnostic chat brain (run_chat_turn)
   answers every inbound through a WhatsApp transport shell. The inbox is a live
   window onto agent conversations; journey `handled_by` records agent | human | both.
2. **Takeover = assignment.** Threads stay unassigned while agent-handled. Whoever
   takes over becomes the exclusive assignee (which is also the reply-collision
   guard); the agent goes silent. Escalations (agent stuck, customer asks for a human,
   keyword rules) land in a shared **Needs attention** queue anyone can claim;
   managers can reassign; the assignee can hand back to the agent.

Zero routing configuration in v1 — manager-routing and round-robin are later options,
added as policy, not schema.

## Consequences
- Matches the corpus inversion: the agent does the work and writes the record; humans
  approve/intervene.
- Presence, typing indicators, and soft locks are Redis ephemera — no tables.
- Escalation rule vocabulary starts hardcoded; configurable later.
