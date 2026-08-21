# 0017 — Voice telephony plugs into the spine in phase 1 (events + identity + journey; NOT the takeover)

**Status:** Accepted · **Date:** 2026-08-21 · Complements ADR 0010 (voice stays outside
the gate) and ADR 0016 (product-led phases).

## Context
Phase 1 must show the customer end to end — and for existing merchants the customer's
history IS voice calls. Waiting for the "voice takeover" phase to connect voice to the
CRM would force a retrofit later and ship a phase-1 journey that lies by omission.

## Decision
Voice joins the CPaaS spine in phase 1 through three structural moves — none of which
touch dialing, the dispatcher, or the gate:

1. **Forward-only event mirroring.** The lead machine emits facts into event_raw at
   its existing choke points: `lead.pushed` (push handler, campaigns, webhooks),
   `call.attempted` (dispatch attempt — including never-connected: no-answer, failed),
   `call.completed` (outcome, duration, recording ref), `call.inbound` (/answer).
   Retries are new attempts, each its own event. Same pattern 07-wiring prescribes
   for chat turns ("new turns also stamp event_raw, forward-only"). No backfill.
2. **Identity via the sync door, stamped on buddy's own rows.** push_lead_handler and
   the inbound answer path call `resolve()` (idempotent contract; buddy MAY call crm
   contracts — the reverse import is what's forbidden) and stamp `customer_id` on
   lead_call_tracker (one nullable column). Buddy writes buddy's table; crm never
   does. Customers are ingested from every voice touchpoint, connected or attempted.
3. **The journey's voice arm.** The journey view reads lead_call_tracker in place —
   exactly V01's design ("unifies chat sessions, call records and crm.message") —
   joined by the stamped customer_id, never by phone-matching at read. The customer
   360 shows inbound/outbound calls, attempted-or-delivered, outcome on the card.

**Explicitly NOT in this cut** (unchanged from ADR 0010): telephony_numbers →
channel_binding, voice through may_contact(), voice manifest rows. The takeover
arrives in a later phase and will find the spine already speaking voice.

## Consequences
- Phase-1 exit now includes: a pilot merchant's existing voice traffic visibly
  building customers and journeys from day one.
- New tasks A14 (voice event mirroring), A15 (resolve()+customer_id stamping);
  A12 (journey) gains the voice arm. Lead-tap task A11 is absorbed into A14.
- The later voice takeover becomes a swap of the send path only — identity, events,
  and journey are already unified, so nothing gets force-fitted.
