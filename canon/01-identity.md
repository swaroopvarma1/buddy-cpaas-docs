# Identity — who people are

Two tables. `platform.identity` is the cross-merchant safety layer (suppression only).
`crm.customer` is the per-merchant person record — one row is one relationship.

### platform.identity (T02) — 10 columns

One row per identifier: suppression, grouping and lifecycle. The whole platform layer.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `kind` | text | CK·UQ | phone \| email \| device |
| 3 | `value` | text | UQ | Normalised raw identifier — phone E.164, lowercased email, device id. UNIQUE (kind, value); joins directly against the matching customer handle column. Raw by decision, 13 Aug 2026 — see the note below. The only index the send path uses |
| 4 | `is_suppressed` | boolean |  | The hot path — one boolean. True if any live suppression exists on any channel; recomputed on every write to col 5 and by the expiry sweep |
| 5 | `suppressions` | jsonb |  | Resolved state keyed by channel, "*" = every channel. Entries carry reason, from, until, source, evidence_ref |
| 6 | `suppression_log` | jsonb |  | Append-only array, never rewritten — the audit trail. No hash chain: nobody forges "please stop contacting me" |
| 7 | `first_seen_at` | timestamptz |  | NULL on registry-only rows |
| 8 | `last_seen_at` | timestamptz |  | Updated lazily, in batch — the only frequently-written column |
| 10 | `created_at` | timestamptz |  |  |
| 11 | `updated_at` | timestamptz |  |  |


**Wiring**
- Written by: suppression events ("delete me and never contact me"), the blacklist migration,
  bounce/complaint handlers. As built (migration 048, PR #1016): `is_suppressed` is DERIVED
  by a full-scope trigger — fires on every insert/update, so even a direct
  `SET is_suppressed = false` is overwritten by recomputation; entries carry `from`/`until`
  and liveness is `until IS NULL OR until > now()` (expiry-as-predicate); `suppression_log`
  is append-only by trigger (strict-prefix guard); kind-conditional format CHECKs (E.164 /
  lowercase) refuse unnormalized values at the table.
- Read by: the gate, exactly once per send — probe the unique `(kind, value)` index with the
  customer's own handle values, read one boolean. On any DB error the accessor returns
  *blocked* (fail closed).
- Never: name/profile columns, per-merchant data, embeddings. Cross-merchant reads exist only
  for the yes/no answer.

### crm.customer (T05) — 18 columns

The customer as one merchant knows them — and their handles, as columns. One row is one relationship.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | UQ·IX | First column of every unique index below — the tenant boundary |
| 4 | `display_name` | text |  | Resolved winner of the name assertions — see "how a customer gets a name" below the table list. Never an identifier, never merge evidence |
| 5 | `primary_locale` | text |  | Resolved, never inferred — a guess about language must not become a column |
| 6 | `timezone` | text |  | Drives quiet hours. Unknown → do not send |
| 7 | `phone` | text | UQ·CK | Stored in E.164 — "+", country code, number, one canonical spelling — normalised at write, enforced by CHECK (phone ~ ^\+[1-9][0-9]{6,14}$). UNIQUE (merchant_id, phone) WHERE status=’active’ — the partial unique index is also the duplicate detector; a collision at insert means "resolve to the existing customer instead". Renamed from phone_e164, 13 Aug 2026 — the format contract moved from the name into the CHECK |
| 9 | `email` | text | UQ | Same partial unique shape |
| 14 | `igsid` | text | UQ | Instagram-scoped id — the only identifier Instagram’s API provides; phone and email are never exposed, and Meta gives no cross-channel linkage. An igsid-only customer is the correct state until in-conversation or order evidence attaches the phone (attach) or staples records (journey 6). Display name and @username arrive as declared-class facts — never handles, a username is changeable |
| 15 | `shopify_customer_id` | text | UQ |  |
| 16 | `external_ref` | text | UQ | Whatever the merchant’s own system calls them |
| 19 | `status` | text | CK·IX | active \| merged_away \| erased. Every unique index is partial on active — a merged-away row keeps its handles without blocking the survivor |
| 20 | `merged_into_id` | uuid | FK·IX | Self-reference · staple, never melt · path-compressed to one hop |
| 21 | `merged_at` | timestamptz |  |  |
| 23 | `first_seen_at` | timestamptz |  | The merge survivor is the older row, tie-broken on the lower uuid |
| 24 | `last_seen_at` | timestamptz | IX | With merchant_id, DESC — the list view |
| 25 | `created_at` | timestamptz |  |  |
| 26 | `updated_at` | timestamptz |  |  |
| 27 | `attributes` | jsonb |  | NEW 13 Aug 2026 — assertion history per profile attribute: {name: [{o,e,k,src,at,…}], locale: […]}. Winners are materialised into cols 4–6; owned by Identity like the rest of the row. Never evicted, never part of any token budget — this is master data, not agent memory |


**Wiring**
- Created ONLY by `resolve(merchant_id, handles{}) -> customer_id` — deterministic, no fuzzy
  matching. Probe order: phone → email → igsid → shopify_customer_id → external_ref, each
  against its partial unique index `(merchant_id, handle) WHERE status='active'`. Hit =
  attach new handles; miss = create; insert race = re-probe. Every adapter (WhatsApp, Shopify,
  chat, CSV import) funnels through it.
- Profile beliefs go through `assert_facts()` into `attributes` (assertion history with
  evidence classes: declared > observed > imported > inferred; inferred capped at 0.5 by
  CHECK). Winners are materialised into `display_name` / `primary_locale` / `timezone`.
- Merge = one UPDATE on the younger row (`status='merged_away'`, `merged_into_id`,
  `merged_at`); survivor = older `first_seen_at`, tie → lower uuid; path-compressed to one
  hop; undo = one UPDATE. Reads across a pair: `WHERE merchant_id=$1 AND (id=$2 OR
  merged_into_id=$2)`. As built (049, PR #1016): the staple cannot cross tenants or point at
  itself — composite FK `(merchant_id, merged_into_id) → (merchant_id, id)` +
  `CHECK (merged_into_id <> id)`; and a handle-history trigger appends any replaced handle
  value into `attributes._handle_history` (ADR 0021 lock #4 — no caller can destroy an old
  handle).
- Read by: everything. `consent_state`, `customer_memory`, enrolments, recipients and the
  journey view all key on `(merchant_id, customer_id)`.
