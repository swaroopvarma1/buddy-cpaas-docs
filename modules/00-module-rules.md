# Module rules — the laws EVERY app/crm module obeys

Read this first; the per-module guides assume it. When a guide and this file conflict,
this file wins. Diagram: `../diagrams/00-master-system.html`.

## Build it like this

1. **The module skeleton & layer law** (final, 23 Aug 2026 — supersedes every
   earlier same-day wording, including "no db/ subpackages"):

   ```
   app/crm/<module>/
     __init__.py    # empty — exports NOTHING
     contracts.py   # THE public surface — re-exports from the logic files;
                    #   the only file other modules may import
     api.py         # thin routes → logic (or db/accessor for trivial reads)
     schemas.py     # leaf Pydantic shapes — imports nothing internal;
                    #   api/contracts/tests import shapes from here
     <concern>.py   # BUSINESS LOGIC by name (resolve.py, facts.py, ingest.py,
                    #   suppression.py): gather → decide (PURE plan) → apply,
                    #   inside a boundary this file owns
     workers.py     # drain loops (only if the module owns one)
     db/            # ALL mechanics behind one hop — the root stays the story
       __init__.py  # the db door: re-exports transaction, DbTxn, domain errors
       accessor.py  # execute one query builder per function; NO decisions;
                    #   splits as accessor_<table>.py as the module grows
       queries.py   # SQL builders, $1 params; splits as queries_<table>.py
       decoder.py   # row → schemas model, DB-side translation only
   ```

   **Layer law**: `api → logic → db/accessor → db/queries` (raw asyncpg, no
   ORM, $1 placeholders only — the repo's three-layer SQL law unchanged).
   **Boundary law**: logic owns transaction scope — atomicity is business
   semantics — and imports db-world things ONLY from its module's db/ door
   (`transaction()`, the opaque `DbTxn`, domain-named errors, all implemented
   once in `shared/db.py`). Logic may OPEN a boundary and PASS the handle; it
   may never CALL anything on it. `import asyncpg` is legal only in
   `shared/db.py` and `db/` packages — grep-enforced. Single-statement reads
   may self-scope inside the accessor; the moment two statements share fate, a
   logic file declares the boundary.
   **Logic style**: GATHER (accessor reads) → DECIDE (pure function returning
   a plan — DB-free testable, loggable, the decision_log spirit) → APPLY
   (accessor writes). No service classes, no repository interfaces: pure core,
   thin shell. Uniform in every module at any size — big modules split INSIDE
   db/, the root shape never changes. Routes never execute queries directly;
   business logic is findable by FILENAME.

   **The package root** (`app/crm/`) holds only surface plumbing: `api.py` (root
   router mounting) and `auth.py` (route dependencies per ADR 0007 — existing JWT
   machinery, no new auth system). The test: a table/contract/SQL = a module;
   /crm-surface plumbing used by all modules = root; a pure leaf helper = `shared/`.
   Root files import no module (acyclic by construction); auth's reach into the
   existing `app.api.security` / `app.core.security` stack is the sanctioned seam.
2. **The boundary is a CI-enforced ownership map** (ADR 0001, amended 23 Aug 2026 —
   org runs one DB role, so grants are unavailable). Tables are `crm_*`/`platform_*`
   prefixed in `public` (logical `crm.x` in this corpus = physical `crm_x`). One
   module owns each table; SQL touching a table may exist ONLY in its owner's
   directory — a CI lint over SQL strings fails the PR otherwise, and buddy code may
   not mention crm/platform tables at all. import-linter enforces the module graph
   (`app.crm` never imports `app.ai`; cross-module inside crm via `contracts.py`
   only). Cross-module access goes through **contract functions** — never a foreign
   SELECT/INSERT. If there is no contract for what you need, that's a design
   conversation, not a workaround. Invariants that must never depend on discipline go
   in the tables themselves: CHECKs and triggers (append-only suppression_log,
   is_suppressed recompute) survive any caller.
3. **Tenancy**: `merchant_id` NOT NULL on every root table, first column of every unique
   index. **No table stores a reseller** — always one derived join. Child rows enter
   through a scoped parent. The CI predicate check will fail your PR otherwise.
4. **Idempotent by construction.** Every contract must be safely callable twice —
   partial-unique indexes and deterministic probes, not caller coordination. If your
   function breaks when two workers race, the design is wrong, not the callers.
5. **Facts vs commands**: things that happened enter through event_raw and a consumer;
   callers who need a result NOW call the contract directly (sync door). Consumers must
   be **order-tolerant** (no ordering guarantee exists) and never signal each other —
   coordination happens through the tables they write.
6. **Fail closed** anywhere permission-adjacent. A missing, NULL, erroring, or unknown
   input means NO. Log the honest reason.
7. **Config through the single resolver only** (static → dynamic → template → playground).
   Never re-implement precedence at a call site; never read `os.environ` in module code.
8. **Workers** run under role flags (CRM_DISPATCH / CRM_WALKER / CRM_CONSUMERS): drain
   with `FOR UPDATE SKIP LOCKED`, never hold a transaction across HTTP, jitter mass
   wake-ups, back off on provider 429s.
9. **Migrations 046+**: sequential, one table owner per migration, NEVER edited after
   merge. Canon-conformance CI diffs the DB against the sealed schema.
10. **Observability**: `set_log_context` at every entrypoint; `track_error` on degraded
    paths; if you bound coverage (top-N, sampling, no-retry) log what was dropped.

## Do NOT (each of these is a scar we already have)

- **No god files.** `template/types.py` is 3,100 lines and blocks every refactor. Split
  before 500 lines; one concept per module file.
- **No router→router imports.** Campaigns/webhooks calling `push_lead_handler` directly
  is the pattern that made buddy unrefactorable. Routers call accessors/contracts only.
- **No re-export hubs.** The 132-line `accessor/__init__.py` is a merge-conflict magnet
  and an import-cycle root. `app/crm/__init__.py` exports NOTHING; import by full path.
- **No DTO→engine imports.** `app/schemas` importing from agent internals inverted the
  layering forever. crm schemas import nothing from engines.
- **No CHECK constraints on channel/connector vocabularies.** Migration 027's
  `supported_channels` CHECK is the reason WhatsApp is DB-illegal today. Vocabulary
  lives in code dicts — a new channel is a deploy, never a migration.
- **No stored derived state that a predicate can answer** (expired, in-window, overdue).
  A stored status needs a sweeper and lies between the fact and the cron.
- **No `app/crm` → `app/ai` imports, ever.** Buddy may call crm contracts; never the
  reverse. Allowed seams: `app/core/*`, `app/services/redis`, encryption/vault accessor,
  the DB pool.
- **No new env var without registering it in the resolver.** 197 loose constants and 90
  bespoke getters is where the current config sprawl came from.
