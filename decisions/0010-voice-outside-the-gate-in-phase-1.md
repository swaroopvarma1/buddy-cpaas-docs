# 0010 — Voice stays outside the gate in phase 1

**Status:** Accepted · **Date:** 2026-08-19

## Context
Pilot workflows are mixed-channel (calls AND WhatsApp). The gate fails closed, and no
voice-channel consent rows exist in phase 1 — routing calls through `may_contact()`
would block every call. The gate's law forbids an override parameter, so a "voice
bypass flag" inside the gate is not an option.

## Decision
A workflow **voice node enqueues a lead into today's dispatch machine**, governed by its
existing checks — DND (fail-closed after Phase 0), blacklist, calling hours. The
enrollment id is stamped on the lead row so funnels join runs ↔ calls. WhatsApp nodes
always pass the new gate. Voice adopts `may_contact()` at the **voice takeover** phase
(when `telephony_numbers` → `channel_binding` lands), with voice-channel consent seeding
designed then.

## Consequences
- No gate bypass exists — voice simply isn't behind the gate yet. One send discipline
  per channel, honestly stated.
- Voice sends don't write manifest rows in phase 1; the funnel reads calls via the
  stamped enrollment id. Manifest coverage for voice arrives with the takeover.
