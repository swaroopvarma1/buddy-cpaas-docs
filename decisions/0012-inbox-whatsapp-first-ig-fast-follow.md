# 0012 — Team inbox: ships with phase-2 WhatsApp inbound; Instagram fast-follows

**Status:** Accepted · **Date:** 2026-08-19

## Context
Requirement: a shared team inbox — customers message the merchant on WhatsApp /
Instagram DM; multiple teammates manage all conversations from one account. Channel
scope and timing decide whether the IG connector enters the plan now.

## Decision
The inbox ships with **phase-2 WhatsApp inbound** — one channel, real conversations,
agent+human takeover proven end to end. **Instagram DM is the immediate fast-follow**:
by then it is a new door + adapter + extractor (`crm.customer.igsid` and
`channel_binding` already anticipate it), and the inbox UI treats channel as an icon on
the thread — no UI rework.

## Consequences
- Phase 2 = inbound WhatsApp responder + inbox, one integrated deliverable.
- The IG messaging API's differences (auth surface, 7-day human-agent window, webhook
  shapes) are isolated to the connector layer and a per-channel window constant on
  `channel_binding.capabilities`.
- UI is designed/developed by Swaroop + Claude; the team receives APIs and contracts,
  not pixel specs.
