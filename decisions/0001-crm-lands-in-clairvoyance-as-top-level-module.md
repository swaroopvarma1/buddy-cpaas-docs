# 0001 — CPaaS lands in clairvoyance as a top-level `app/crm` module

**Status:** Accepted · **Date:** 2026-08-19 · **Amended 2026-08-23** (enforcement
mechanism — see the amendment at the end)

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

## Amendment (2026-08-23) — the boundary is enforced in code, not by grants

Org database policy: one database, one role, everything in `public` — per-module
Postgres roles and separate schemas are not available. The *boundary itself* is
unchanged (one writer per table; contracts are the only doors); its **enforcement
moves from the database to the build**:

1. **Prefixes replace schemas.** Physical tables are `crm_*` and `platform_*` in
   `public`. The corpus keeps writing logical names (`crm.customer`,
   `platform.identity`); they map 1:1 to `crm_customer`, `platform_identity`. The
   prefix carries the tenancy law: `crm_` tables always have `merchant_id NOT NULL`,
   `platform_` tables never have a merchant column.
2. **A CI SQL-ownership lint is the grant substitute.** A table→owning-module map
   (from the canon) plus a lint over SQL strings that fails the PR when: SQL touching
   a table appears outside its owner's module directory; any INSERT/UPDATE/DELETE on
   a table appears outside its owner; or buddy code (`app/ai`, `app/api`) mentions
   any `crm_*`/`platform_*` table at all (buddy imports sync-door contract functions
   only). This catches violations at PR time — earlier than a grant would.
3. **import-linter guards the module graph.** CI contracts: `app.crm` never imports
   `app.ai`; cross-module imports inside `app/crm` go through each module's
   `contracts.py` only. This closes the path the SQL lint can't see — one module
   calling another's internal query helpers.
4. **Invariants that must never depend on discipline live in the tables.** Grants are
   gone; constraints and triggers are not: normalization CHECKs (E.164, lowercased
   email), an append-only trigger on `platform_identity.suppression_log` (rewriting
   history raises), and a trigger recomputing `is_suppressed` on every write to
   `suppressions` so the compliance boolean cannot drift from the jsonb regardless of
   who writes it.

**Accepted trade-off:** runtime blast radius is not covered — a compromised process
can read anything, which is the org's baseline for every system on this database.
What is preserved is everything this ADR wanted: one writer per table, contracts as
the only doors, and violations caught mechanically instead of by review vigilance.
