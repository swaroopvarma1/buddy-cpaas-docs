# Outreach — the launch button, the plan, the run

The model in one sentence: **a broadcast is ONE deliberate action (admit an audience into a
workflow, now or scheduled); a workflow is the WHOLE treatment; anything reactive enters
through the workflow's own entry rule with no broadcast at all.**

Two admission doors, honestly different:
- **Deliberate** — a human picks segment + workflow + schedule and presses go. That is a
  broadcast: one-time execution, frozen audience, funnel clocks.
- **Reactive** — the workflow's `definition.entry` watches event topics (`checkouts/create`,
  `link.clicked via_broadcast B7`), always-on, admitting one person at a time. Live/paused
  lifecycle belongs to the plan.

A broadcast may LAUNCH a workflow, never OWN one: ten broadcasts can launch the same
workflow; cancelling a blast stops future admissions while open runs resolve by their own
rules. A one-shot blast is a one-node "quick send" workflow the composer mints — the
merchant never manages it.

### crm.broadcast (T17) — 13 columns

One deliberate execution: a segment admitted into a workflow, now or on schedule, audience frozen at dispatch. The launch button — nothing else.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | IX |  |
| 3 | `name` | text |  |  |
| 4 | `campaign_key` | text | FK·IX | The badge — composite FK (merchant_id, campaign_key) into the registry (T22), so a typo is an insert error, never a split report. |
| 6 | `segment_id` | uuid | FK | → segment · the audience, NOT NULL — whose list to freeze |
| 11 | `status` | text | CK | draft · scheduled · resolving · dispatching · paused · sent · closed · cancelled · failed. resolving and dispatching stay split because they fail differently — stuck resolving is a segment problem, stuck dispatching is a pipe problem. sent = dispatch over; closed = the reporting window ended and the funnel went final |
| 13 | `scheduled_for` | timestamptz |  | Now, or the future moment |
| 14 | `resolved_at` | timestamptz |  | When the audience was frozen — deliberately LATE, just before dispatch, so Friday’s send is aimed with Friday’s aim, never Monday’s |
| 15 | `dispatched_at` | timestamptz |  |  |
| 16 | `closed_at` | timestamptz |  |  |
| 17 | `created_by` | text |  | Who built the thing that just addressed 8,400 people — the authorship fact, paired with created_at; the other clocks belong to the execution |
| 18 | `created_at` | timestamptz |  | Authored Monday, frozen Friday 09:58, fired 10:00, final Sunday — the gaps between the clocks are the design |
| 19 | `workflow_id` | uuid | FK | The action, always — NOT NULL. Every frozen recipient is enrolled into this workflow, source_broadcast_id stamped on the run. An action REFERENCE, never containment: the workflow stays independent, ten broadcasts can launch it, cancelling the blast stops future admissions while open runs resolve by their own rules. |


**Wiring**
- Dispatch at `scheduled_for`: freeze the audience ONCE (`resolved_at`, deliberately late) →
  one recipient row per person → apply the target workflow's guards → enrol everyone else
  into `workflow_id`, stamping `source_broadcast_id` on each run. The broadcast never sends.
- `campaign_key` is a composite FK `(merchant_id, campaign_key)` into the campaign registry.
- Funnel read: skips on T18; sends/refusals/receipts on the runs' manifest rows — `runs WHERE
  source_broadcast_id = B`, then their messages.

### crm.broadcast_recipient (T18) — 7 columns

The admission queue under an execution, one row per person — it dies at a guard or becomes a run. Why "the number was smaller than expected" always has an answer.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 2 | `broadcast_id` | uuid | PK·FK | → broadcast · with customer_id the composite PK — a set needs no surrogate (the T21 precedent), and the PK IS the dedupe: a crashed freeze re-runs idempotently |
| 3 | `customer_id` | uuid | PK·IX | The freeze stores WHO. Where — her current handle — is live, asked at send by the walker; the manifest records what was actually used |
| 8 | `status` | text | CK·IX | queued · claimed · skipped · enrolled · cancelled — the queue’s OWN life, nothing more. Sends, refusals and receipts live on the run’s manifest rows, reached through the join the schema already owns: runs WHERE source_broadcast_id = B. handed_off and expired died with the redesign — the handoff is now a run; staleness was a triggered concept |
| 9 | `skip_reason` | text |  | Pre-enrolment deaths only — chiefly the workflow’s guards: already_in_flow (reenter) · cooldown_active, plus resolve failures. Gate verdicts are NEVER recorded here — every send the walker proposes mints a manifest row, refused or not, with its diary page |
| 10 | `claimed_by` | text |  | Which worker holds it |
| 11 | `claimed_at` | timestamptz |  | The reaper releases anything claimed longer than N minutes |
| 15 | `updated_at` | timestamptz |  |  |


**Wiring**
- The admission queue. Composite PK `(broadcast_id, customer_id)` IS the dedupe — a crashed
  freeze re-runs idempotently. No merchant_id (child row; rides the broadcast).
- Status: queued · claimed · skipped · enrolled · cancelled. Skips are the workflow's guards:
  `already_in_flow` (reenter), `cooldown_active` — with the reason on the row. Gate verdicts
  never land here; they are manifest rows.
- Workers claim with `FOR UPDATE SKIP LOCKED`; a reaper releases stale claims.

### crm.workflow (T19) — 10 columns

The plan — ONE document a walker reads live. Many customers walk the same one; edits reach everyone not yet past them.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | IX |  |
| 3 | `name` | text |  |  |
| 6 | `definition` | jsonb |  | THE plan, whole: {entry, nodes, edges, goal, exits}. Entry carries reenter + cooldown; waits carry local-time windows — sections of one document, never columns. Node ids are minted once at creation and NEVER regenerated: current_node resolves against this live document, and a regenerated id strands every waiting walker |
| 9 | `status` | text | CK | draft · live · paused · archived. paused = no new enrolments AND the sweeper skips its rows; archived = walkers force-exited as ejected. Written down because nobody leaves it implicit twice |
| 10 | `version` | integer |  | Bumped on publish — an AUDIT stamp ("she entered under v3", the DPDP story), never an execution pin: the walker reads the live document. The publish-time validator is what makes live reads safe — it blocks the unsafe edit classes (deleting an occupied node without a strand-to-exit rule, changing entry semantics mid-flight) |
| 11 | `created_by` | text |  |  |
| 12 | `created_at` | timestamptz |  |  |
| 13 | `updated_at` | timestamptz |  |  |
| 14 | `draft` | jsonb |  | Dittofeed’s two-column trick: edits land here; publish copies to definition and bumps version. A half-finished edit can never become the document walkers read |


**Wiring**
- `definition` jsonb is THE plan, whole: `{entry, nodes, edges, goal, exits}` — read live by
  the walker; node ids are minted once and never regenerated. `draft` receives edits;
  publish copies to `definition` and bumps `version` (an audit stamp, never an execution
  pin). The publish validator blocks unsafe edit classes (deleting an occupied node without
  a strand-to-exit rule, changing entry semantics mid-flight).
- Entry carries reenter + cooldown — the admission guards enforced for BOTH doors.
  Sealed follow-up (31 Aug 2026, not yet built): entry also names its repeat policy —
  `on_repeat: ignore · refresh_latest · refresh_max(<field>) · accumulate` +
  `debounce_minutes` (a repeat entry patches the still-unfired run: winner-takes-context,
  sliding alarm) — see modules/05-outreach §Repeat entries.
- Templates live on send nodes; waits are nodes; goal checks are re-evaluated at fire time
  (never send "did you forget?" to someone who just paid).

### crm.workflow_enrollment (T20) — 16 columns

One person’s run through a workflow — the board-game token: the only stateful row in outreach. Hardened by three research passes against Oban, River, n8n, Laudspeaker and a decade of Mautic.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK | Surrogate — she can re-enrol later; each run is its own row |
| 2 | `merchant_id` | text | IX |  |
| 3 | `workflow_id` | uuid | FK | → workflow |
| 4 | `workflow_version` | integer |  | The AUDIT stamp, written at entry — what was current when she walked in, never an execution pin: the walker reads the live definition, so a long run may finish under a newer plan. Its live reader is the did-the-edit-help read — goal_met rate GROUP BY version, v3 against v2 — the entry cohort, four bytes, queryable. The per-step truth lives where it always did: each send’s diary page records what actually fired |
| 5 | `customer_id` | uuid | IX |  |
| 6 | `status` | text | CK | waiting · parked · exited. enrolled folded into waiting (it was waiting with an immediate wake); parked = attempts exhausted, held for the merchant to see and resume — errors never silently discard a run |
| 7 | `current_node` | text |  | The token’s square — a STABLE node id, resolved against the live definition at each wake |
| 8 | `wake_at` | timestamptz | IX | The durable timer AND the lease: the claim pushes it forward by the lease interval, so a dead worker’s row self-heals — comes due again with no reaper. Immutable as a delay once entered (the timing she was promised); jittered on mass writes — 8,400 rows must never come due the same second. Partial index (wake_at, id) WHERE status=waiting. This is what replaces a workflow engine |
| 9 | `entered_at` | timestamptz |  | The run’s birth — the clock timeout arithmetic reads: timed_out is entered_at + the plan’s max age, and "stuck over 7 days" is one predicate on it |
| 10 | `exited_at` | timestamptz |  | The death clock — the retention sweep reads it: exited rows age out on a partial index over exited_at, which is most of what keeps the hot table small |
| 11 | `exit_reason` | text | CK | goal_met · timed_out · withdrawn · ejected. withdrawn = she said stop, exactly that; converting IS goal_met; ejected = the workflow was archived or the merchant cancelled the run. errored is gone — errors park, never exit |
| 12 | `context` | jsonb |  | POINTERS to the spark plus the few small facts the sends will need — {chk_88412, 1850, blue kurti} — never the payload itself: e-91 already stores the cart verbatim, and a photocopy read at every wake bloats and goes stale (the artifact_ref habit). Random branch outcomes are recorded here BEFORE the send — a retry must never re-roll the dice |
| 13 | `enrollment_key` | text |  | Defaults to the customer id; the partial unique is (workflow_id, enrollment_key) WHERE status <> ’exited’. Keyed runs are how two open orders get two live WISMO flows — Dittofeed’s orderId lesson, retrofitting which means rewriting the index that guards every enrol |
| 15 | `attempts` | smallint |  | Incremented on claim BY THE CLAIM — a poison run that crashes its worker must count against itself. Backoff with jitter written into wake_at; exhausted → parked |
| 16 | `last_error` | text |  | Why the run parked, readable on the merchant’s screen — debugging a journey that silently stopped, without reading logs |
| 18 | `source_broadcast_id` | uuid | FK·IX | The spark, resolved once at enrolment — stamp at write, never a jsonb hop at read. NULL for unsparked runs: a cart flow has no blast. The broadcast-level read is WHERE source_broadcast_id = B7; the campaign-level read is one indexed join to the blast’s badge. The run wears no key of its own — a photocopy one FK away drifts the day the blast is retagged (replaced campaign_key, col 17, the same day it arrived) |


**Wiring**
- One person's run. `wake_at` is the timer AND the lease: the walker's claim pushes
  `wake_at` forward one lease window — a dead worker's claim self-heals when the clock
  passes again; no reaper. Retries are harmless because the manifest's `dedupe_key` eats the
  duplicate.
- `source_broadcast_id` stamped at enrolment for broadcast-admitted runs (NULL for reactive
  entries) — the funnel's join key. One OPEN run per (workflow, customer), partial-unique
  enforced.
- Every send node: build template+variables → `may_contact()` → `send(token, …)`. A gate
  refusal completes/parks per the plan's rules — never retries around the gate.
  Cancel-on-goal: the goal event (e.g. orders/create) resolves open enrolments.

### crm.campaign (T22) — 7 columns

The registry of badges — it names the season, never contains it. Executions wear the key; this table is why two spellings of Diwali can never split one report.

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `merchant_id` | text | PK | Tenancy — half the key |
| 2 | `key` | text | PK | The stamp executions carry — diwali-2026, chosen once, stable forever. The FK from broadcast is what turns a typo into an insert error instead of a broken report |
| 3 | `name` | text |  | The display name — renameable without touching one stamped row: key is identity, name is presentation |
| 4 | `starts_at` | date |  | The report window’s header — for eyes, never a gate: a campaign never blocks a send |
| 5 | `ends_at` | date |  | Same — the season ends in the report, not in the gate |
| 6 | `created_by` | text |  | Who minted the season |
| 7 | `created_at` | timestamptz |  |  |


**Wiring**
- A registry that names the badge — never a container. Nothing points FROM campaign to
  anything; executions wear the key. Kills tag-typo split reports: `broadcast.campaign_key`
  is a composite FK here, so a typo is an insert error.
- Campaign-level reads: broadcasts `WHERE campaign_key = K`; flow performance joins runs →
  broadcast → badge.
