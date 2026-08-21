# 0006 — Runtime: one image, role flags

**Status:** Accepted · **Date:** 2026-08-19

## Context
app/crm needs API serving plus long-running workers (message dispatcher, workflow
walker, status/consent consumers). Separate service vs monolith vs in-process tasks.
Pools are already tight (see 0002 — PgBouncer).

## Decision
Same clairvoyance image everywhere. CRM APIs are served by the main server; CRM workers
run as **separate pods selected by POD_ROLE-style env flags** — the pattern the voice
dispatcher already established (`ENABLE_DISPATCHER` precedent). Same Postgres cluster
(required anyway: the journey view reads chat tables in place), reached through the new
PgBouncer.

## Consequences
- Foundations scaffolds worker roles once; squads ship workers without deploy design.
- Workers never compete with API/voice traffic for an event loop; scaling a worker is a
  replica count, not a redesign. A later split into a separate service stays possible —
  the boundary is already the role flag.
