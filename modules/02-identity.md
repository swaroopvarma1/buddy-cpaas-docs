# Identity — build guide

Owns: `crm.customer` (T05) · `resolve()` · `assert_facts()` · merge. Diagram:
`../diagrams/02-identity.html`. Squad: Pod A.

## Build it like this

- **`resolve(merchant_id, handles{}) -> customer_id` is the ONLY creator of customer
  rows** — grep-enforceable: no other INSERT into crm.customer exists anywhere.
- **Deterministic probe order**: phone → email → igsid → shopify_customer_id →
  external_ref, each against its partial unique `(merchant_id, handle) WHERE
  status='active'`. Hit = attach the new handles to that row. Miss = create.
  **Insert race = re-probe** — the unique index is the referee; two racing callers
  converge on one row.
- **Normalize at write**: phone to E.164 (enforced by CHECK), email lowercased. The
  partial unique IS the duplicate detector — a collision means "resolve to the existing
  customer", never an error page.
- **Profile beliefs go through `assert_facts()`** into the `attributes` assertion
  history, with evidence classes declared > observed > imported > inferred; inferred is
  capped at 0.5 by a DB CHECK. Winners materialize into `display_name` /
  `primary_locale` / `timezone`. Master data: never evicted, never token-budgeted.
- **Merge = staple**: one UPDATE on the younger row (`status='merged_away'`,
  `merged_into_id`, `merged_at`); survivor = older `first_seen_at`, tie → lower uuid;
  path-compressed to one hop; undo = one UPDATE. Reads across a pair:
  `WHERE merchant_id=$1 AND (id=$2 OR merged_into_id=$2)`.
- **Buddy integration (ADR 0017)**: push_lead_handler and the inbound answer path call
  resolve() through the sync door and stamp `customer_id` on THEIR OWN lead rows.
  Buddy writes buddy's table; identity never writes buddy's tables.

## Do NOT

- **No fuzzy matching, ever.** Similarity-scored identity is how CRMs invent people.
  If handles don't match deterministically, they are two customers until evidence
  (in-conversation, order data) staples them.
- **No get-or-create anywhere else.** A connector implementing its own lookup is the
  rot this module exists to prevent. Callers pass handles; resolve() decides.
- **Never store a guess as a column.** Inferred locale/timezone stays in the assertion
  history at ≤0.5; materialized winners come from declared/observed evidence.
  Unknown timezone → the gate says no; that pressure is a feature, not a bug to patch.
- **No LIKE-over-JSONB phone joins** (the P0.4 scar). Handles are COLUMNS with
  indexes; anything matching by pattern is wrong by construction.
- **Don't add handle types as rows** (an identities side-table). Handles-as-columns is
  deliberate: a new handle type is a migration — a considered event, not a Tuesday.
- **Don't put profile facts in customer_memory** (agent shelf) or vice versa. Name,
  locale, timezone are master data here; impressions and promises are notes there.
- **Don't melt merges** (copying fields between rows, deleting the loser). Staples are
  reversible; melts are forever.

## Scale & future-fit

- Every probe is an index-only lookup; resolve() must stay callable inline from a
  conversation runtime (P3) without a queue in front of it.
- `last_seen_at` is updated lazily/batched — never on the hot path per event.
- CSV import is a loop over the same contract — if bulk needs a special path, the
  contract is too slow; fix the contract.

Refs: 01-identity.md (corpus T02/T05) · ADR 0017 · entry-points doc (sync door).
