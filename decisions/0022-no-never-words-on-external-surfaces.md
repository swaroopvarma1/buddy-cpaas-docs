# 0022 — No never-words on external surfaces

**Status:** Accepted · **Date:** 2026-08-30

## Context
The positioning is locked: Buddy is a **customer engagement platform** — explicitly
not a CRM, not a CDP, not a CPaaS (those are the never-words). Meanwhile the build
mounted its API family at `prefix="/crm"` (`/crm/ingest/events`, `/crm/customers/*`)
with `CRM *` OpenAPI tags — surfaces that outside teams integrate against and that
appear in swagger, producer docs, and merchant-visible tooling. Nothing is live yet,
so a rename today is free; after the first external integration it is a breaking
change forever.

## Decision
**No never-word may appear on any externally visible surface**: URL paths, OpenAPI
tags, response headers, producer/consumer docs, error strings, console-visible
labels. The `/crm` prefix is dropped — the doors mount as `/ingest/events` and
`/customers/*` (a neutral umbrella like `/engage` is acceptable if a grouped family
is wanted; the ruling is only that `crm` cannot be in it). Tags go neutral
(`Ingest`, `Customers`, `Journey`).

**Internal names are out of scope**: the `app/crm/` package, `crm_*` table prefixes,
`CRM_ROLE`, internal env vars. They are invisible outside the codebase; renaming
them is churn with no positioning value. If that boundary ever changes, it gets its
own ADR.

## Consequences
- One line owns the fix today (`app/main.py` router mount) plus hardcoded paths in
  docs/tests and the nautilus relay client URL. Called out on clairvoyance#1025 /
  nautilus#195 before merge.
- Every future door inherits the rule: name external surfaces by what they do
  (ingest, customers, journey), never by a category word.
- Corpus prose keeps using CRM/CPaaS freely as internal shorthand — the corpus is
  not an external surface.
