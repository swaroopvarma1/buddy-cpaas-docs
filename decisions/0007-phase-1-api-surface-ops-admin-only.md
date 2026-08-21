# 0007 — Phase 1 API surface: ops/admin only

**Status:** Accepted · **Date:** 2026-08-19

## Context
Someone must author the pilot's workflows, segments and templates before anything sends.
Merchant-facing console screens would drag frontend work and the unfinished users
access-model cutover into the critical path.

## Decision
Phase 1 ships clean **`/crm/*` endpoints guarded by existing JWT + admin/S2S auth** —
workflow CRUD + publish, segment CRUD, broadcast launch, consent capture/import,
connector/binding admin, template management. Our team (not merchants) drives them for
the pilot. The merchant console is a **fast-follow after M2, built on these same
endpoints** — no throwaway surface.

## Consequences
- The users JSONB→grant-table read cutover stays off the critical path; it becomes a
  prerequisite of the console fast-follow instead.
- API design is still done to console quality (naming, pagination, errors) — the console
  will consume it unchanged.
