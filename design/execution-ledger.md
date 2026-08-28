# Execution ledger — built vs pending

The running account of what is MERGED versus what is OPEN, updated as PRs
land. The [task map](task-map.md) is the plan; this is the score. Update
protocol: when a PR merges, move its items here with the PR number and
date; when scope changes, the task map changes first.

## Merged

### CPaaS foundation — MERGED 2026-08-23 via [#1010](https://github.com/juspay/clairvoyance/pull/1010) (`9669eab`)

*Authored and reviewed as [#1016](https://github.com/juspay/clairvoyance/pull/1016); manas-narra force-pushed #1016's final tree byte-identically onto #1010's branch and merged under that number; #1016 closed. Review history (CodeRabbit round, tenancy thread, skill review) lives on #1016.*

| Item | Delivered |
|---|---|
| P0.5 | Migration CI: numbering + immutability guards; 026/034 renumbered with tracker reconciliation |
| A1 | `app/crm` scaffold to the sealed skeleton (contracts · api · schemas · logic · db/ door) |
| A5 | `/crm` router mount + admin auth + s2s verifier (dormant until A9) |
| A6 | `crm_customer` (049) + `resolve()` — INCLUDING staple-on-collision + ADR 0021 evidence ladder (pulled forward) |
| A7 | `assert_facts()` + assertion history + winners + inferred 0.5 cap (pulled forward) |
| A8 ½ | `crm_event_raw` (051) + `record_event()` with dedupe + ADR 0020 stamp passthrough |
| A14 | Voice event mirroring: lead.pushed · call.attempted · call.completed · call.inbound. Customer traffic only — `is_non_customer_lead` excludes \*_TEST, playground and DAILY_STREAM (transport-only service, client-driven payloads → numbers aren't trusted identities). HOLD_TRANSFER deliberately mirrors (reversed 23 Aug on engineer evidence: live configs dial real customers, e.g. the ride booker; a future staff-consult config gets a per-config exclusion, not a mode ban). Stamp pass-through (23 Aug): call.inbound mirrors from the created tap sequenced after the stamp (born attributed); call.attempted/completed carry `lead.customer_id` (column surfaced on the model); lead.pushed born NULL for the A10 consumer — resolution stays singular, mirrors never resolve |
| A15 | Voice identity stamping at the lead-creation choke point (hook registry — data layer imports nothing) |
| B2 | `platform_identity` (048) + suppression contracts, liveness-aware, trigger-derived |
| — | `check_crm_boundaries.py` (10 CI rules) · atomic grammar · building-modules.md · CLAUDE.md |
| — | Review-round hardening: full-scope recompute trigger · composite tenant-pinned merge FK + no-self-merge · event_raw immutability trigger · whitespace-handle fix (zero-handle law, regression-tested) · PII-free logs · inbound-mirror persistence guard · list endpoint ships `CrmCustomerSummary` (attributes jsonb never fetched for lists) · s2s verifier unit-tested · 71 pure tests total |

### PR [clairvoyance#1014](https://github.com/juspay/clairvoyance/pull/1014) — journey view, call arm *(MERGED 2026-08-25, `737be0e`)*

| Item | Delivered |
|---|---|
| A12 ½ | `crm_journey_event` view (052) — call arm only, reads LCT in place, `WHERE customer_id IS NOT NULL`, direction lowercased; 12-column canon contract |
| A12 ½ | `GET /crm/customers/{id}/journey` — keyset cursor `(started_at, id)`, record-owned (`timeline.py` logic seam); view registered in TABLE_OWNERS |

Review: skill round bounced module placement (journey→record), accessor-in-contracts, OFFSET pagination — all fixed by author. Remaining arms (message · consent · commerce `event`) land with their lanes via CREATE OR REPLACE VIEW.

## Open — phase 1

| Lane | Items | Notes |
|---|---|---|
| A (Identity & Record) | A2 consumer runtime · A4 config resolver · **A9 ingest front door** · **A10 extractor registry + stamping processor** · A12 remaining arms (message/consent/commerce — with their lanes) · A13 transactional send consumer | A9→A10 is the read-path critical chain; call-arm journey SHIPPED (#1014) |
| B (Permission) | T07/T08 + `record_consent()` · B3 blacklist backfill · B4 `decision_log` · **B5 `may_contact()` gate** (token, tz ladder, quiet hours, caps — ADR 0018 spec done, zero code) · Shopify consent importer | |
| C (Connectivity) | C1 installations · C2 bindings · C3 `crm_message` manifest · **C4 `send()` + WhatsApp adapter** · C5 dispatcher · C6 receipt walker · C7 WABA template registry · C8 `/crm/connectors` | zero code |
| X (external) | **X1 anchor relay — blocks phase-1 exit, NO OWNER** · X2 embedded signup — no owner · X3 pilot merchant + WABA — Swaroop | |
| U (Swaroop + Claude) | U1 loom wiring · U2 customers list (**backend live** — `GET /crm/customers` returns `CrmCustomerSummary` rows; detail GET carries full attributes) · U3 customer 360 (needs A12+B4) · U4 template manager (needs C7) | design complete (ADR 0019) |
| P0 remainder | PgBouncer (**before** A2 multiplies connections) · fail-closed voice DND · P0.4 LIKE-over-JSONB fix · reseller backfill | |

## Open — follow-ups created during the foundation build

- DB-integration test harness (CI Postgres) for resolve()/suppression race tests
- `crm_event_raw` partitioning when volume demands (documented in migration 051)
- Corpus migration into `clairvoyance/docs/crm/` + review skill into repo `.claude/` once phase-1 code stabilizes
- A10's consumer stamps events the voice taps recorded unattributed in the interim
- **A15b — SCHEDULED (decided 23 Aug): historical LCT stamp, backfill night after
  release.** Runbook:
  1. Prereq: release deployed, migrations 048–051 applied. Release does NOT wait
     for the backlog — leads finishing pre-deploy are history lost forever; leads
     mid-flight at deploy are safe (fail-open, deduped, partial-but-truthful).
  2. Scope: **status='FINISHED' rows only** (never mid-flight — their updated_at
     feeds reaper timing), customer_id IS NULL, payload phone present. Dry-run a
     100-row sample first; watch the unparseable-phone rate.
  3. The sweep: batches ~500 with sleeps, off-peak; per row →
     `resolve(merchant, {phone}, evidence="observed", source="lct-backfill")` →
     optional `assert_facts(name, "observed")` from payload customer_name (decided:
     include it — nameless lists are useless) → stamp via
     update_lead_customer_id_query. Idempotent + kill-safe (customer_id IS NULL
     guard); resumable keyset on created_at.
  4. **HARD RULE: customers only via resolve() — never INSERT INTO crm_customer**
     (raw SQL skips E.164 normalization → CHECK violations, skips the platform
     registry, violates the boundary law the CI guard enforces).
  5. Populates exactly: crm_customer (one per distinct merchant+phone),
     platform_identity (one per distinct phone, registry only — zero suppression
     state), and the LCT stamp. NO events, NO consent, NO suppressions.
  6. Verify: stamped-by-status counts, customer/registry counts, leftover NULLs =
     unparseable phones (truthful). Straggler pass ~1 week later, same script.
  Known cosmetics (accepted): first_seen_at = backfill night (affects only future
  staple survivor ties); flat last_seen ordering for the historical block in U2.

## State in one sentence

The write path exists end-to-end for voice (call → customer → event recorded)
and the first read exists (call-arm journey, #1014); no consumers drain yet and
nothing sends yet (no gate, no manifest, no adapter) — critical path:
A9→A10 ∥ C1–C6 + B4/B5 → A13, with X1 as the ownerless external blocker.

## Suggested next slices

PR-2: A8 completion + A9 + A2 (the spine goes live) · PR-3: A10 + remaining
journey arms (unblocks U3 fully) · C-pod starts C1–C3 in parallel · B-pod
starts T07/T08 + B4.
