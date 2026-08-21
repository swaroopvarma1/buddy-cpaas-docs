# 0002 — Phase 0 base fixes precede all CRM work

**Status:** Accepted · **Date:** 2026-08-19

## Context
07-wiring names four ground bugs to kill first. Separately, production is already seeing
**connection-pool exhaustion** — clairvoyance currently runs *without* PgBouncer.
Clairvoyance's migration sequence has duplicate numbers (two 026s, two 034s) and the CPaaS
plan reserves 046+.

## Decision
No `crm` feature code until this Phase 0 lands:

1. **Introduce PgBouncer** (transaction pooling mode) in front of Postgres — fixes the
   live pool-exhaustion issue. The asyncpg `statement_cache_size=0` change ships **in the
   same rollout** (transaction mode breaks prepared statements otherwise — the docs'
   warning becomes part of this task, not a separate one).
2. Invert the DND/pre-check **fail-open → fail-closed** (highest-consequence bug).
3. Drop the `supported_channels` CHECK (WhatsApp becomes DB-legal).
4. Fix the LIKE-over-JSONB phone lookup.
5. Migration governance: CI check for duplicate migration numbers; reserve 046+ for CPaaS.
6. Reseller backfill: all NULL `reseller_id` merchants → house reseller **'breeze'**,
   then NOT NULL (tracked on the Slack CPAAS list).

## Consequences
- Phase 0 is pure execution — parallelizable across the team immediately, no design gate.
- PgBouncer rollout must be verified under the dispatcher's connection patterns before
  crm work adds new pool consumers.
