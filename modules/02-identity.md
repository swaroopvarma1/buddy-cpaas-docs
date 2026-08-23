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
- **Attach only FREE handles — a collision is evidence, not an error** (settled
  23 Aug 2026, caught in PR #1010 review). When an arriving handle is owned by a
  DIFFERENT active customer, do not raise and do not overwrite: the co-occurrence of
  two customers' handles in one trusted payload IS the staple evidence the canon
  names ("order data staples them") → merge. Survivor = older first_seen_at; the
  loser's partial uniques free its handles (status flip), the survivor attaches
  them; the loser keeps its copies as audit + undo. The attach-free check and the
  staple trigger are the same code path. A handle occupied on the SAME row
  (customer changed email) is kept, never overwritten — the dropped value converges
  later via a staple if it ever co-occurs again; never-overwrite is deliberate
  (recycling risk), revisit only with pilot churn data.
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
- **Don't grow `platform.identity` kinds for channel-scoped ids** (asked and settled
  23 Aug 2026). Kinds are GLOBAL identifiers only — phone, email, device: same value =
  same endpoint at every merchant, which is what makes a cross-merchant suppression
  registry coherent. WhatsApp rides `phone` (wa_id is the E.164 number). An IGSID is
  merchant-scoped by construction (different id per IG business account, no linkage
  from Meta), so a "global" suppression on one is a category error — scoped ids are
  handle COLUMNS on crm.customer probed by resolve(), and their don't-contact state is
  per-merchant consent (T08). A genuinely new global identifier class = one migration
  swapping the CHECK. Normalization on T02 is compliance-critical in the dangerous
  direction: a probe that misses a differently-formatted suppressed value CONTACTS
  someone who said stop.

## Scale & future-fit

- Every probe is an index-only lookup; resolve() must stay callable inline from a
  conversation runtime (P3) without a queue in front of it.
- `last_seen_at` is updated lazily/batched — never on the hot path per event.
- CSV import is a loop over the same contract — if bulk needs a special path, the
  contract is too slow; fix the contract.
- **T02's growth map (asked 23 Aug 2026): vocabulary, never structure.** `kind` grows
  ~never (global identifier classes only; RCS/WhatsApp ride `phone`, scoped ids go to
  T05). What grows routinely: channel keys inside `suppressions` (sms, rcs — new key,
  no DDL) · reason/source vocabulary and writers (TRAI NCPR/DND registry sync when
  voice/SMS enter the gate; legal holds; bounce-vs-complaint) · readers (P2 broadcast
  pre-flight batch check; an ops view over the log). Never lands here: reachability
  ("not on WhatsApp" is capability, not suppression), consent (per-merchant yes, T07/
  T08), profile data, scoped ids. Ten columns and one boolean in five years too.

Refs: 01-identity.md (corpus T02/T05) · ADR 0017 · entry-points doc (sync door).
