# 0021 — Handle lifecycle: attach if free · overwrite on the evidence ladder · staple on collision

**Status:** Accepted · **Date:** 2026-08-23 · Refines the identity guide's attach
rules (and supersedes the same-day "keep, never overwrite" interim position).
Companions: ADR 0017 (stamping), canon T05.

## Context
Handles are columns — one phone, one email per customer row — so three situations
need law: an arriving handle whose slot is empty, one whose slot holds a different
value (the customer changed their email at checkout), and one owned by a different
customer. The interim rule ("never overwrite") loses real handle changes: the row
stops following the person, and every future touch on the new handle mints a
duplicate. Swaroop's decision: overwrite when the evidence is clear (an order the
customer typed, a handle they used live) — and make it structurally impossible to
get wrong.

## Decision
The policy table (enforced inside identity, nowhere else):

| Situation | Behavior |
|---|---|
| Slot empty | Attach (any evidence ≥ imported) |
| Same value | No-op; bump last_seen_at |
| Different value, evidence `declared` or `observed` | **Overwrite**; old value appended to the attributes assertion history |
| Different value, evidence `imported` | Keep existing — bulk data never displaces live truth |
| Evidence `inferred` | Never touches a handle column |
| Value owned by ANOTHER active customer | Staple (merge — the sealed collision rule), never overwrite |

Structural enforcement — four locks:
1. **No overwrite flag exists.** `resolve(merchant_id, handles{}, *, evidence,
   source)` — callers state what kind of evidence they carry; identity applies the
   policy. Importers must pass `imported`; live traffic defaults to `observed`.
   (The gate's no-bypass principle applied to identity.)
2. **One ladder.** The same EVIDENCE_RANK `assert_facts()` uses — defined once in
   identity, imported everywhere else.
3. **One writer.** Exactly one query builder in identity's accessor writes handle
   columns; the CI ownership lint blocks other modules, grep blocks a second path
   within identity.
4. **The table keeps the history itself.** A trigger on crm_customer appends the
   old value into the attributes history whenever a handle column changes to a
   different non-NULL value — even a rogue UPDATE cannot destroy an old handle
   (one-DB-role reality: invariants live in tables).

## Consequences
- resolve()'s signature gains evidence/source; A6's implementation adds the policy
  branch + the trigger migration alongside the attach-collision fix (same code path).
- Old handles are never lost; audit and undo survive; the convergence property
  holds (an orphaned old handle that reappears alone staples back on the next
  co-occurrence).
- Number-recycling risk is contained as before: consent belongs to the address
  (T07), so an overwritten phone carries no consent with it.
