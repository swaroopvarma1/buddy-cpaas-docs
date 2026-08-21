# Audiences — segments and lists

A segment is a stored question; membership is computed at read time, never stored. A static
list is the one exception: an imported set IS its members, stored as narrow rows.

### crm.segment (T15) — 9 columns

A stored, honest question about customers — or a named frozen list whose members live in T21. Missing from four revisions before anyone noticed.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | IX |  |
| 3 | `name` | text | UQ | Unique per merchant |
| 6 | `definition` | jsonb |  | The stored question — dynamic segments only; NULL for a static list, whose members live in segment_member (T21). Definitions read customer.attributes and events — memory notes are never queryable — and the compiler rejects any inferred attribute: a segment that gates a campaign is a decision, and guesses may never decide |
| 7 | `member_count` | integer |  | For a static list: exact forever, stamped at import. For a dynamic segment: cached, because merchants refresh this obsessively — display-only, nothing ever decides on it |
| 8 | `computed_at` | timestamptz |  | The honesty stamp — the count says how old it is instead of lying |
| 9 | `created_by` | text |  | Who built the thing that just addressed 8,400 people |
| 10 | `created_at` | timestamptz |  |  |
| 11 | `updated_at` | timestamptz |  |  |


**Wiring**
- Definition JSON compiles to ONE SQL query over customer (+ journey-derived predicates).
  The compiler REJECTS any inferred attribute — enforcement lives in the compiler, not the
  UI, so no caller can bypass it.
- `evaluate(id) -> customer_id[]` (chunked) · `count(id) -> {n, computed_at}` — counts always
  carry their computed_at stamp.
- Read by: broadcast dispatch (the freeze evaluates the segment once at `resolved_at`).
  A segment is never a permission list.

### crm.segment_member (T21) — 4 columns

Static membership, stored honestly — a frozen list IS its member set. NEW 14 Aug 2026.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `segment_id` | uuid | PK·FK | → segment. With customer_id, the composite PK — a set needs no surrogate |
| 2 | `customer_id` | uuid | PK·FK | → customer |
| 4 | `added_at` | timestamptz |  |  |
| 5 | `source` | text |  | import diwali-vips.csv · console: Anjali — where each member came from |


**Wiring**
- Static membership (CSV import): each line goes through `resolve()` (create or attach —
  people are never stored in arrays), then one member row, `ON CONFLICT DO NOTHING`.
  Skipped lines get a report (line + reason). Consent columns route to the capped import
  path — never straight to granted.
- Composite PK `(segment_id, customer_id)`; no merchant_id — a pure child row: every read
  enters through an already-scoped parent (the segment, or the customer), and uuid keys
  cannot collide across merchants.
