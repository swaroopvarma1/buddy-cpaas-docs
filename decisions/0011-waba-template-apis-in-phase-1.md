# 0011 — WABA template APIs built in phase 1

**Status:** Accepted · **Date:** 2026-08-19

## Context
Every WhatsApp marketing/utility send uses a Meta-approved template; the manifest stores
`template_id` + variables. Templates have their own lifecycle (pending, approved,
paused, rejected — Meta can flip these at any time). The company is a Meta Tech
Provider, so the management APIs are available to us.

## Decision
Build **template management into `/crm/*` in phase 1**: create/submit, list, delete
against Meta's Tech Provider template APIs, plus a local template registry kept honest
by the `message_template_status_update` webhook (through `event_raw`, like every other
inbound). Send-time behavior: a send referencing a non-approved template is **refused
before the provider call** — a manifest row with the honest reason, not a provider error.

## Consequences
- More Connectivity-squad scope in phase 1 (accepted trade), zero manual Meta-console
  work per merchant, and the console fast-follow gets template management for free.
- The template registry is a reference table (name, language, category, status, WABA),
  not a content store — Meta renders; we never store a rendered string (manifest law).
