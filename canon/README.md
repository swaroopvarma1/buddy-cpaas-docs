# Buddy CPaaS — build overview

The customer data model beneath an agentic CRM for WhatsApp/voice commerce (Indian D2C
merchants). The design inversion everything follows from: **the agent writes the customer
record as a side-effect of doing the work, and reads it before it speaks.** Humans approve
decisions; nobody types into the record.

Schema: **18 tables + 1 view**, PostgreSQL, two namespaces (`crm`, `platform`) — realized
as `crm_*`/`platform_*` table prefixes in `public` (one DB role org-wide; ADR 0001
amendment, 23 Aug 2026). Logical `crm.x` here = physical `crm_x`. This corpus is
the implementation-facing description: final column shapes plus how the pieces wire together.
Column numbers are stable; gaps in numbering are retired columns (never reused).

## Tree

```
canon/
├── README.md            this file — system thesis, laws, vocabulary, reading order
├── 01-identity.md       platform.identity (T02) · crm.customer (T05) — who people are
├── 02-record.md         event_raw (T13) · journey_event view (V01) · customer_memory (T09)
├── 03-permission.md     consent_event (T07) · consent_state (T08) · decision_log (T14) · the gate
├── 04-connectivity.md   connector_installation (T11) · channel_binding (T12) · message (T16) · send()
├── 05-audiences.md      segment (T15) · segment_member (T21)
├── 06-outreach.md       broadcast (T17) · broadcast_recipient (T18) · workflow (T19)
│                        workflow_enrollment (T20) · workflow_version (T25, ADR 0023)
│                        · campaign (T22) — the launch-button model
└── 07-wiring.md         end-to-end flows · module contracts · tenancy law · clairvoyance transform
```

Read 07-wiring.md after the table files; it assumes their vocabulary.

## The laws (bind every design decision)

1. **Merchant records are separate.** `merchant_id` is globally unique (the merchant registry
   PK) and every merchant has exactly one reseller (`merchants.reseller_id` NOT NULL). Root
   crm tables store `merchant_id` NOT NULL; pure child rows (`broadcast_recipient`,
   `segment_member`) ride their parent's scope. **No table stores a reseller** — it is always
   one derived join. CI fails any query missing its tenancy predicate.
2. **The platform layer only says yes or no.** One table (`platform.identity`); no row
   represents a human, no column is named after one.
3. **Permission = one merchant + one channel + one purpose**, and it expires.
4. **Two consent stores, never one**: append-only ledger ("prove she agreed") + resolved
   state ("may I send right now"), written in the same transaction.
5. **Staple records, don't melt them.** Merge = pointer flip, reversible in one statement.
6. **Fail closed.** A missing or unknown input to the gate means no. Always.
7. **Memory is a shelf of named notes**, agent-only, human-legible markdown. No vector store.
   Guesses may steer, never speak, never decide (inferred confidence ≤ 0.5, DB-enforced).
8. **Outreach proposes · Permission disposes · Connectivity transmits.** One send path.
   A broadcast to 8,400 people is 8,400 separate permission checks.

## Vocabulary

| term | means |
|---|---|
| customer | The customer as ONE merchant knows them; handles (phone, email, igsid…) are columns on this row |
| journey_event | A thing that already happened; all of them in order are her journey (a view, not storage) |
| workflow | The plan — the whole treatment, as one document; templates and waits live on its nodes. Each publish is an immutable version (T25) and a run executes the version it entered under (ADR 0023) |
| workflow_enrollment | One person's run through a workflow |
| broadcast | **The launch button**: one deliberate execution — a segment admitted into a workflow, now or scheduled |
| campaign | A registered tag on executions — never a container |
| the gate | `may_contact()` — the single permission predicate every outbound passes |
| the manifest | `crm.message` — one row per outbound attempt, blocked attempts included |
