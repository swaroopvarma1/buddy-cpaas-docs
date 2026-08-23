# Execution ledger — built vs pending

The running account of what is MERGED versus what is OPEN, updated as PRs
land. The [task map](task-map.md) is the plan; this is the score. Update
protocol: when a PR merges, move its items here with the PR number and
date; when scope changes, the task map changes first.

## Merged

### PR [clairvoyance#1016](https://github.com/juspay/clairvoyance/pull/1016) — CPaaS foundation *(status: approved-pending-merge, 2026-08-23)*

| Item | Delivered |
|---|---|
| P0.5 | Migration CI: numbering + immutability guards; 026/034 renumbered with tracker reconciliation |
| A1 | `app/crm` scaffold to the sealed skeleton (contracts · api · schemas · logic · db/ door) |
| A5 | `/crm` router mount + admin auth + s2s verifier (dormant until A9) |
| A6 | `crm_customer` (049) + `resolve()` — INCLUDING staple-on-collision + ADR 0021 evidence ladder (pulled forward) |
| A7 | `assert_facts()` + assertion history + winners + inferred 0.5 cap (pulled forward) |
| A8 ½ | `crm_event_raw` (051) + `record_event()` with dedupe + ADR 0020 stamp passthrough |
| A14 | Voice event mirroring: lead.pushed · call.attempted · call.completed · call.inbound |
| A15 | Voice identity stamping at the lead-creation choke point (hook registry — data layer imports nothing) |
| B2 | `platform_identity` (048) + suppression contracts, liveness-aware, trigger-derived |
| — | `check_crm_boundaries.py` (10 CI rules) · atomic grammar · building-modules.md · CLAUDE.md |

## Open — phase 1

| Lane | Items | Notes |
|---|---|---|
| A (Identity & Record) | A2 consumer runtime · A4 config resolver · **A9 ingest front door** · **A10 extractor registry + stamping processor** · **A12 journey view V01 + timeline API** · A13 transactional send consumer | A9→A10→A12 is the read-path critical chain |
| B (Permission) | T07/T08 + `record_consent()` · B3 blacklist backfill · B4 `decision_log` · **B5 `may_contact()` gate** (token, tz ladder, quiet hours, caps — ADR 0018 spec done, zero code) · Shopify consent importer | |
| C (Connectivity) | C1 installations · C2 bindings · C3 `crm_message` manifest · **C4 `send()` + WhatsApp adapter** · C5 dispatcher · C6 receipt walker · C7 WABA template registry · C8 `/crm/connectors` | zero code |
| X (external) | **X1 anchor relay — blocks phase-1 exit, NO OWNER** · X2 embedded signup — no owner · X3 pilot merchant + WABA — Swaroop | |
| U (Swaroop + Claude) | U1 loom wiring · U2 customers list (**backend ready** — `GET /crm/customers` merged) · U3 customer 360 (needs A12+B4) · U4 template manager (needs C7) | design complete (ADR 0019) |
| P0 remainder | PgBouncer (**before** A2 multiplies connections) · fail-closed voice DND · P0.4 LIKE-over-JSONB fix · reseller backfill | |

## Open — follow-ups created during the foundation build

- DB-integration test harness (CI Postgres) for resolve()/suppression race tests
- `crm_event_raw` partitioning when volume demands (documented in migration 051)
- Corpus migration into `clairvoyance/docs/crm/` + review skill into repo `.claude/` once phase-1 code stabilizes
- A10's consumer stamps events the voice taps recorded unattributed in the interim

## State in one sentence

The write path exists end-to-end for voice (call → customer → event recorded);
nothing reads yet (no consumers, no journey) and nothing sends yet (no gate,
no manifest, no adapter) — critical path: A9→A10→A12 ∥ C1–C6 + B4/B5 → A13,
with X1 as the ownerless external blocker.

## Suggested next slices

PR-2: A8 completion + A9 + A2 (the spine goes live) · PR-3: A10 + A12
(journeys exist → unblocks U3) · C-pod starts C1–C3 in parallel · B-pod
starts T07/T08 + B4.
