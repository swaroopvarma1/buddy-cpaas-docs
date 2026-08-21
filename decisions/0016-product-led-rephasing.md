# 0016 — Product-led re-phasing: customers e2e first, static flows second, AI+inbox third

**Status:** Accepted · **Date:** 2026-08-21 · Amends ADR 0007 (UI timing) and the
phase LABELS in 0003/0012 (sequence unchanged).

## Context
The original phasing was engineering-led (skeleton send → pilot flows → inbound).
Swaroop re-phased it product-led: every phase must end with something a merchant can
see and use. Team is 6 (three pods, ADR/task-map).

## Decision — the phases

**Phase 1 — Customers end to end.**
Customers e2e INCLUDING UI · WhatsApp outbound (transactional — e.g. order
confirmation fires immediately on the Shopify event; no workflow engine yet) ·
the event spine · a basic journey view (what this customer went through).
Exit: pilot merchant's customers visible in the console with journeys, and
gate-checked order-confirmation sends live.
- UI amendment to ADR 0007: the customer console (list + 360/journey) ships IN
  phase 1, built by Swaroop + Claude on the same /crm APIs — pods' capacity untouched.

**Phase 2 — Flows and reach (static).**
Workflow engine (walker) + entry rules · abandoned-checkout delayed flow · WhatsApp
template flows: send → button click / link click → next message (static branching —
clicks are webhooks already landing in event_raw; a consumer advances the flow) ·
segmentation · WhatsApp broadcast + campaign registry · flow/segment/broadcast UI.
Exit: merchant launches a broadcast into a static flow and watches the funnel.

**Phase 3 — Conversations.**
WhatsApp inbound (generic message support) · the AI agent plugin (chat brain +
WhatsApp shell) · the team inbox (ADR 0012–0015) · customer_memory (MOVED here from
phase 2: its only writer is the agent by corpus law — in phase 2 the shelf would be
empty) · Instagram fast-follow. Later phases: voice takeover, console RBAC
completion, RCS/etc.

## Consequences
- ADR 0003's "inbound in phase 2" now reads "phase 3" — sequence identical, labels
  moved. ADR 0012's inbox timing likewise.
- The pilot's two flows split across phases: order-confirmation (ph 1,
  transactional consumer — no walker), abandoned-checkout (ph 2, walker + goal
  cancel).
- STOP/START consent handling stays in phase 1 — legally required the day we send.
- Task map re-hydrated to these phases (design/task-map.md).
