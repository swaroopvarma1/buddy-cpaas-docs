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
