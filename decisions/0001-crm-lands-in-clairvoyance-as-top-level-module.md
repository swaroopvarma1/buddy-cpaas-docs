# 0001 — CPaaS lands in clairvoyance as a top-level `app/crm` module

**Status:** Accepted · **Date:** 2026-08-19

## Context
The build docs (07-wiring) mandate the code lands inside clairvoyance with `crm` +
`platform` Postgres schemas. The open question was package placement: nested under the
existing breeze_buddy tree, or a new top-level boundary. Breeze_buddy's tree carries known
structural debt (3,100-line types.py, router→router imports, schema→agent layering
inversion) that a new domain must not inherit.

## Decision
`app/crm/` is a **new top-level package**, sibling to `app/ai` / `app/api` — one
sub-module per contract owner (identity, permission, connectivity, record, outreach),
each owning its tables via per-module Postgres roles ("the boundary is a database grant,
not a convention"). CRM routers mount outside the `/agent/voice/breeze-buddy` prefix.

Breeze_buddy itself is NOT restructured now. After the CPaaS core lands, buddy
(`app/ai`, `app/api`) gets standardized **toward** the `app/crm` pattern.

## Consequences
- The crm module starts clean: no imports from `app/ai/**`; the existing credential
  vault, Redis, and config primitives are reached through thin, named seams.
- Two conventions coexist in the repo for a while — accepted cost, paid down when buddy
  is standardized later.
