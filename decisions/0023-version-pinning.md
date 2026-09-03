# 0023 — Runs execute the definition version they entered under

**Status:** Accepted — authored by Swaroop in the repo (`docs/crm/adr/0023-version-pinning.md`, 3 Sep 2026) and BUILT the same day (clairvoyance #1067–#1071, rollout phases 10–14); mirrored here 3 Sep 2026. Supersedes the "audit stamp, never an execution pin" sentence of migration 057, canon T19 §edits and T20 col 4. · **Date:** 2026-09-03

## Context

Canon T19 and migration 057 chose ONE live `definition` per plan: publish copies `draft` → `definition` in place, every run not yet past the change follows the new document, and `version` is "an AUDIT stamp ('she entered under v3'), never an execution pin". Safety came from the publish validator, which BLOCKED the stranding edits (removing a node open runs stand on; changing `entry` while runs are open).

That is right for runs measured in minutes: a cart-recovery board lives a day and its squares are almost always empty, so fixing a template name reaches every waiting run, which is a feature. It fails for runs measured in weeks: a loan-onboarding board has a token on nearly every square at all times, so nearly every meaningful edit hits the stranding guard, and the only exits were archive-and-eject every journey or a second plan beside the first — which is why the funnel first shipped as five disposable clocks. Every enterprise journey engine (Braze Canvas, SFMC Journey Builder; Temporal and Step Functions for code) pins in-flight runs to the version they entered under and lets new entrants take the new one. This record reverses the 057 sentence and makes "edits reach everyone" an opt-in mode.

## Decision

1. **Every enrollment executes the definition version recorded in `crm_workflow_enrollment.workflow_version` at entry.** The column is the execution pin: the walker resolves the run's definition by `(workflow_id, workflow_version)` on every claim, never by reading `crm_workflow.definition`. No such version row = an honest park, never a fallback to the live document.
2. **Publish creates a new, immutable version row** — `crm_workflow_version` (canon T25: merchant_id, workflow_id, version, definition, on_publish, published_by, published_at; BEFORE UPDATE trigger refuses every edit). `crm_workflow.definition` stays the LATEST version's document, for the entry consumer and the console.
3. **A plan declares `on_publish: "pin"` (default) or `"migrate"`.** `pin`: new entrants take vN+1; runs in flight finish vN; the occupied-node and entry-change refusals do not apply. `migrate`: inside the publish atom every open run is re-pinned to vN+1, allowed only when the stranding validator passes (occupied nodes kept, `entry` unchanged) — 057's semantics as a mode a short board opts into.
4. **The entry consumer reads twice.** Entries are evaluated against the LATEST version (new runs start on the newest document). Goals and `wait_event` listening are evaluated per OPEN RUN against that run's pinned version — a v3 run is ended by v3's goals and woken by v3's listening even after v5 changed them; every write names the run it is about. One consumer, two reads; a `migrate` plan is the degenerate case.
5. **Old versions are kept for the life of the plan — never deleted.** (Amended before merge, rollout phase 14: the designed retention sweep was dropped. A version row is one small document, a plan publishes tens of them, every read is a point lookup on the unique index, and an exited run's `workflow_version` must keep answering "what did this run execute". Migration 064's comment that omits a DELETE guard "because the retention sweep deletes unreferenced old versions" is superseded; **a DELETE-refusing trigger is a named follow-up migration** — with one DB role, invariants live in tables.)
6. **A template a retained version names may not be retired while runs reference that version** — nor while a live or paused plan's latest document names it (its next entrant would be pinned to a withdrawn template). Retire refuses (409) with both counts. The check and the local withdrawal share one transaction, ahead of the provider call, under a transaction-scoped Postgres advisory lock keyed by `(merchant, channel, template name)` (`app/crm/shared/locks.py`): every pinning path — enrol, publish, migrate-forward — holds the key SHARED for each template the pinned document sends; retirement holds it EXCLUSIVE, so it waits for in-flight pinners and counts their rows, later pinners wait for its verdict, and pinners never block one another. The guard's count is REGISTERED from the composition root (`worker_main.register_retire_guard(outreach.template_references)` — the record/consumers.py inversion), so connectivity never imports outreach.

## Consequences

- **Storage:** T25 `crm_workflow_version` (migration 064, owner outreach), backfilled from every live plan's current `definition`; open runs on unrecoverable older versions re-pinned to the current one at migration time.
- **Walker:** reads the pinned definition by `(workflow_id, version)` through a small process-local LRU (`outreach/definitions.py`, 512 entries — valid because rows are immutable and never deleted); the live row is read for STATUS only (archived ejects, paused snoozes).
- **Entry consumer:** iterates the customer's open runs (goals, listening, per pinned version) then the merchant's live plans (entries, latest); run-scoped cancel/resume builders.
- **Tooling:** `POST /workflows/{id}/versions/{from}/migrate?to=N` (migrate-forward: the stranding validator as a pure function + the target's template-approval check, one atom), `GET /workflows/{id}/versions` (open-run count per version), the template-retirement guard. No retention sweep.
- **Guards demoted:** the occupied-node and entry-change refusals survive only as `migrate`-mode preconditions and as the migrate-forward check.
- **Canon text:** 057's comment cannot be edited (merged migrations are immutable) — this record supersedes it; T19 §edits ("edits reach everyone not yet past them") is the definition of `migrate` mode, not the law; T20 col 4 reads "execution pin".
- **Vocabulary:** one new word on the definition (`on_publish`), in code, validated at publish; a closed CHECK on the column that stores it (a status enum, law 11).

## Alternatives rejected

- **Keep blocking.** A three-week board would be unpublishable for most edits; the loan funnel stays clocks forever and every long journey pays the reporting-join tax.
- **Per-node versioning.** Too fine: a run would mix nodes from several versions and no document would describe what it is executing.
- **Copy the document into each run's context at entry.** Bloats every row with the whole plan, breaks "one document" for the console, makes a version-wide fix impossible.

## Rollout (as landed, 3 Sep 2026)

Phase 11 storage + the word (#1068) · 12 the walker reads the pin (#1069) · 13 the consumer's two reads (#1070) · 14 migrate-forward + the retirement guard, retention dropped (#1071). Existing plans default to `pin`; the shipped cart template carries no `on_publish` word and therefore pins — the notes' intent for it is `migrate` (a one-line follow-up on `docs/crm/plans/cart-recovery*.json`).
