# Outreach — build guide

Owns: `crm.workflow` (T19) · `crm.workflow_enrollment` (T20) · `crm.broadcast` (T17) ·
`crm.broadcast_recipient` (T18) · `crm.segment` (T15) · `crm.segment_member` (T21) ·
`crm.campaign` (T22) · `enrol()` + `tick()` · `broadcast.schedule()`. Diagram:
`../diagrams/05-outreach.html`. Squad: Pod C (phase 2).

## Build it like this

- **The workflow is ONE live-read document**: `definition` jsonb `{entry, nodes,
  edges, goal, exits}`. Edits land in `draft`; publish copies to definition and bumps
  `version` (an AUDIT stamp — "she entered under v3" — never an execution pin). The
  publish validator blocks unsafe edit classes (deleting an occupied node without a
  strand-to-exit rule; changing entry semantics mid-flight). **Node ids are minted once
  and NEVER regenerated** — current_node resolves against the live document.
- **wake_at is the timer AND the lease**: the walker's claim pushes wake_at forward one
  lease window, so a dead worker's row self-heals when the clock passes — **no reaper**.
  Immutable as a delay once entered; jittered on mass writes (8,400 rows must never
  come due the same second). Partial index `(wake_at, id) WHERE status='waiting'`.
- **Runs park, never vanish**: attempts++ ON the claim (poison runs count against
  themselves); exhausted → `parked` with `last_error` readable on the merchant's
  screen. `enrollment_key` (partial unique WHERE not exited) allows keyed concurrent
  runs (two orders = two WISMO flows).
- **Per send node**: build template+variables → `may_contact()` → `send(token, …)`.
  Goal re-checked AT FIRE TIME (never "did you forget?" to someone who just paid);
  the goal event (orders/create) cancels open enrollments. **Random branch outcomes are
  recorded BEFORE the send** — a retry must never re-roll the dice.
- **Voice nodes** (ADR 0010): enqueue a lead into the existing machine with
  enrollment_id stamped — no gate, no manifest row, until the takeover phase.
- **Broadcast = the launch button only**: freeze the audience LATE (at dispatch), one
  recipient row per person (composite PK = the dedupe; a crashed freeze re-runs
  idempotently), apply the workflow's own guards (reenter/cooldown) as skips WITH
  reasons, enrol the rest stamping `source_broadcast_id`. The broadcast never sends.
- **Segments**: definition jsonb compiles to ONE SQL query; **the compiler rejects any
  inferred attribute** — enforcement in the compiler so no caller can bypass. Counts
  always carry `computed_at`. Static lists are member rows via resolve().
- **Campaign is a badge registry**: composite FK from broadcast turns a tag typo into
  an insert error. Nothing points FROM campaign to anything.

## Repeat entries — debounce and refresh (sealed position 31 Aug 2026 · follow-up, NOT in #1029)

Humans act in sessions; event streams arrive in bursts. Any flow that talks to a
human about a burst needs exactly one of: **dedupe** (built — the open-run unique),
**debounce + refresh** (this position), or **accumulate**. Today a repeat entry event
for an open run is silently absorbed by the unique — correct for *one message per
burst*, wrong for *which* message.

**The example that seals it.** Priya abandons six carts between 10:00 and 10:09; the
plan says "message 10 minutes after abandonment." As built, the run is born at 10:00
with cart #1 (Rs 800) and fires at 10:10 — the wrong cart, while she is still
shopping. The sealed behavior: each repeat entry **patches the unfired run** —
context keeps the winner (the Rs 4,500 cart), the alarm slides to now+10 — so at
10:19 she gets ONE message, about the biggest cart, sent only after she has actually
gone quiet.

**The vocabulary** (entry section of the document, beside reenter + cooldown —
per-plan words, never walker behavior):

- `on_repeat`: `ignore` (default — today's behavior) · `refresh_latest` ·
  `refresh_max(<payload field>)` · `accumulate`
- `debounce_minutes`: every matching repeat slides the entry wait's alarm

**Mechanics when built**: one idempotent UPDATE in the entry processor (the
`resume_run_on_event` shape): `WHERE status='waiting' AND current_node = <entry
node>` — a run past its first square is NEVER patched; what it already said was true
when it said it. Source-event idempotency unchanged (each event still marks itself
used).

**Where each policy earns its keep**: `refresh_max` — abandonment (biggest cart) ·
`refresh_latest` — failed-payment bursts (quote the LATEST error), order edits before
the COD call (confirm the final order, not the first snapshot), disruption comms
(four revised ETAs in 30 minutes — speak the latest, never ETA #2 of 4) ·
`accumulate` — two COD orders in an hour = one call confirming both; back-in-stock
batches = one message listing all four items. **Where it is deliberately wrong**:
WISMO (keyed runs, never merged) and transactional flows (no debounce — act now),
which is exactly why this is plan vocabulary, not engine behavior.

**Fire-time alternative, noted for phase 2**: freeze nothing — the send node re-reads
the winner from the spine at fire time via a record contract (the goal-re-check
pattern, e.g. "her highest-value abandonment since entered_at"). More general and
always fresh, but needs a new cross-module contract and a per-fire read; the
entry-patch version needs neither. Revisit when send nodes want live facts for other
reasons.

## Do NOT

- **Don't build a workflow engine.** No BPMN, no Temporal, no state-machine framework.
  wake_at + the live document IS the engine; everything else is a liability with a
  roadmap.
- **Don't pin execution to a version.** Long runs finish under newer plans by design;
  the per-step truth lives in each send's decision page.
- **Don't regenerate node ids on edit/save** — a regenerated id strands every waiting
  walker at a square that no longer exists.
- **Don't record gate refusals as recipient skips.** Skips are pre-enrolment deaths
  (guards, resolve failures). Every PROPOSED send mints a manifest row, refused or not.
- **Don't let a broadcast own a workflow.** Ten broadcasts can launch the same one;
  cancelling a blast stops admissions while open runs resolve by their own rules.
- **Don't evaluate the segment at authoring time** — Friday's send is aimed with
  Friday's aim. Freeze late, once, at resolved_at.
- **Don't add an `errored` exit** — errors park for a human, never silently exit.
- **A segment is never a permission list.** Membership answers "who to ask the gate
  about", nothing more.

## Scale & future-fit

- The walker scales by replicas; correctness comes from the lease + dedupe_key, not
  from worker uniqueness.
- Exited rows age out on a partial index over exited_at — that sweep is most of what
  keeps the hot table small.
- Click/branch consumers (button.reply, link.clicked) advance runs by topic — adding
  a branch type must be a node-vocabulary change in the document, not a walker rewrite.

Refs: 05-audiences.md + 06-outreach.md (corpus) · ADR 0004 / 0010 / 0016.
