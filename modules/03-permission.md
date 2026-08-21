# Permission — build guide (the pair owns this; never split)

Owns: `crm.consent_event` (T07) · `crm.consent_state` (T08) · `crm.decision_log` (T14) ·
`platform.identity` (T02) · `record_consent()` · `may_contact()` · the importer
interface. Diagram: `../diagrams/03-permission.html`. Squad: Pod B.

## Build it like this

- **Two stores, one transaction, never disagree**: `record_consent(event)` is the single
  writer — it appends the ledger row AND rewrites the resolved state row in the same
  transaction. Ledger answers "prove she agreed"; state answers "may I send right now".
- **Append-only is enforced, not intended**: REVOKE UPDATE/DELETE on consent_event plus
  a trigger that refuses both. No hash chain — nobody forges "stop contacting me".
- **`address` is mandatory** on ledger rows — consent belongs to a way of reaching a
  person, not to the person; number recycling is why. `artifact_ref` points at ONE
  primary evidence (event_raw id, transcript span, form id, CSV row).
- **State resolver**: most-specific wins → strictest wins on tie → fallback
  `prohibited`. IMPORT resolves to granted with capped evidence class.
- **Expiry is a predicate**: `expires_at` read as `IS NULL OR > now()` at gate time.
  Its meaning follows status (granted → permission ends · withdrawn → re-ask embargo
  lifts · pending_confirm → link dies).
- **The gate** — `may_contact(merchant, customer, channel, purpose)`: consent-state
  probe + suppression boolean + frequency cap (Redis rolling counter) + quiet hours in
  the CUSTOMER's timezone. All mandatory; **any missing/NULL/error/unknown → NO with
  the honest reason.** Yes mints a **single-use send token** (short TTL, bound to
  customer+channel+purpose) that send() consumes. **Every verdict — allow AND refuse —
  writes a decision_log row** in the same act.
- **Importers are pluggable** (ADR 0008): an adapter normalizes a source into IMPORT
  consent events and hands them to record_consent(). Shopify first; new sources are new
  adapters, never new write paths.
- **Suppression** (platform.identity): every write recomputes `is_suppressed` and
  appends to `suppression_log` in one transaction. The gate reads ONE boolean. On any
  DB error the accessor returns *blocked*.
- **Blacklist migration**: one-time copy → dual-write window → flip reads. Keep the
  fail-closed-on-error behavior through every stage of the cutover.

## Do NOT

- **No override parameter. No bypass flag. No allowlist. Ever.** The day an override
  exists, it is the product. Escalations change data (record consent), not the gate.
- **No stored `expired` status** and **no cron that manufactures ledger rows** — expiry
  is arithmetic; a sweeper's stored status lies between token-death and cron-run.
- **Never write consent_state outside record_consent's transaction.** A second writer
  is how the two stores learn to disagree.
- **Never let IMPORT claim above the lowest evidence class** — the ladder is enforced
  in record_consent, not in the importer's good intentions.
- **Don't cache gate verdicts.** The probe is one indexed read; a cached YES outlives
  a withdrawal and becomes a compliance incident.
- **Don't skip decision_log on refusals** — the refusal diary is the why-didn't-it-send
  screen and the audit story; it is the one genuinely unreconstructable table.
- **Redis down ≠ allow.** Frequency-cap unavailability is a missing input → NO.
- **Don't build a purposes table yet.** Hardcoded dotted purpose_key CHECK list;
  custom purposes wait for a real need.

## Scale & future-fit

- The gate must hold at thousands of checks/minute: one state probe + one identity
  probe + one Redis op. Anything heavier belongs upstream of the gate.
- decision_log: monthly partitions; retention = partition drop; a pruned page's
  pointer on the manifest survives as a tombstone.
- A broadcast to 8,400 people is 8,400 gate checks — design for the loop, no batch
  shortcut that skips per-recipient verdicts.

Refs: 03-permission.md (corpus) · ADR 0008 · gate-mechanics spec (pending — blocks B5).
