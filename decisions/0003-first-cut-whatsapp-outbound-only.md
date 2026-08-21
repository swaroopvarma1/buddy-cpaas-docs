# 0003 — First cut is WhatsApp outbound only; inbound mechanics designed-in

**Status:** Accepted · **Date:** 2026-08-19

## Context
The original walking-skeleton sketch included inbound WhatsApp (webhook → resolve →
agent replies). The company is already a **Meta Tech Provider using embedded signup**
(onboarding exists outside this repo), but no WhatsApp send/receive path exists anywhere
yet. Answering inbound requires wiring a conversational brain to a new transport — a
product surface of its own.

## Decision
- **Phase 1 (first cut): outbound only** — Shopify/Breeze event → workflow → gate →
  `send()` → WhatsApp Cloud API → status walker updates the manifest.
- **Inbound moves to phase 2**, but the mechanics are built inbound-ready from day one:
  `channel_binding.address` is the inbound which-merchant lookup; `event_raw` ingests
  ALL WhatsApp webhooks from day one (delivery statuses are already inbound webhooks);
  STOP/START keyword consent flows off those same webhook events. Phase 2 "inbound"
  then only adds: routing `message.inbound` topics to a responder.
- Candidate for the phase-2 responder (not yet decided): reuse the channel-agnostic chat
  brain (`run_chat_turn`) with WhatsApp as a new transport shell, the way the widget
  added voice.

## Consequences
- Delivery-status and STOP-keyword handling force the webhook ingress + event_raw spine
  to exist in phase 1 anyway — inbound conversation later is routing, not plumbing.
- Consent capture in phase 1 relies on STOP/START + imports + console capture, not
  in-conversation opt-in.
