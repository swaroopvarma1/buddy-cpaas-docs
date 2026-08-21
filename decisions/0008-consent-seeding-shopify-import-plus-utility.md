# 0008 — Consent seeding: Shopify import + utility purpose, importer is pluggable

**Status:** Accepted · **Date:** 2026-08-19

## Context
The gate fails closed and `consent_state` starts empty — unseeded, the pilot sends
nothing. Something legitimate must authorize the first WhatsApp sends.

## Decision
Two mechanisms compose for the pilot:

1. **Shopify consent import** — the merchant's Shopify customer marketing-consent fields
   map to **IMPORT-class consent events** for the whatsapp channel (artifact_ref = the
   Shopify payload in event_raw; evidence capped per the ledger's law).
2. **Order-confirmation is a utility purpose** — a transactional `purpose_key` aligned
   with Meta's *utility* template category, carrying a lower consent bar than marketing.
   Abandoned-checkout remains `marketing.promotional.*` and rides imported consent.

**The importer is connector-shaped from day one**: a pluggable interface
(source → normalized IMPORT consent events). New sources (POS, CSV, Judge.me, future
platforms) are new adapters — never schema changes, never new write paths into the
ledger. `record_consent()` stays the single writer.

## Consequences
- The Permission squad owns the importer interface + the Shopify adapter as its first
  deliverable after the ledger/state/gate core.
- Compliance posture is explicit and auditable: every pilot grant traces to a Shopify
  artifact or a utility-purpose classification — nothing is silently assumed.
