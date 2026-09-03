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
| 6 | `definition` | jsonb |  | The LATEST published version's document, whole: {entry, nodes, edges, goals, exits, purpose_key, on_publish, stages?} — read by the entry consumer (new runs start here) and the console; **never what a run executes** (that is its pinned T25 row — ADR 0023, 3 Sep 2026). `entry` is one door or a LIST of doors `{topic, start, reenter, cooldown_hours, key, on_repeat, debounce_minutes, restart_on_repeat}` (a journey first seen at KYC starts on the KYC square); `goals` is a list of tiers `{topics, key?: {event, run}, exit_reason}`; `stages` is the ladder sugar stored beside the board it expanded to. Node ids are minted once at creation and NEVER regenerated |
| 9 | `status` | text | CK | draft · live · paused · archived. paused = no new enrolments AND the sweeper skips its rows; archived = walkers force-exited as ejected. Written down because nobody leaves it implicit twice |
| 10 | `version` | integer |  | Bumped on publish; every publish writes an immutable `crm_workflow_version` row (T25) under this number, and runs PIN to it — **ADR 0023 (3 Sep 2026) reversed the earlier "audit stamp, never an execution pin"**. Under `on_publish: pin` (default) the stranding refusals do not apply — a new version cannot strand anyone; under `migrate` the validator still blocks deleting an occupied node and changing `entry` while runs are open, because every open run is re-pinned inside the publish atom |
| 11 | `created_by` | text |  |  |
| 12 | `created_at` | timestamptz |  |  |
| 13 | `updated_at` | timestamptz |  |  |
| 14 | `draft` | jsonb |  | Dittofeed’s two-column trick: edits land here; publish copies to definition and bumps version. A half-finished edit can never become the document walkers read |


**Wiring**
- `definition` jsonb is the LATEST plan document; the walker executes each run's PINNED
  version (T25 by `(workflow_id, version)`, ADR 0023). Node ids are minted once and never
  regenerated. `draft` receives edits; publish validates, copies `draft` → `definition`,
  bumps `version`, writes the T25 row, and (migrate mode only) re-pins open runs — one atom.
  Publish also refuses a send node whose template the T23 registry does not hold approved
  (#1065, via connectivity's `template_status` contract). A ladder draft is stored beside the
  board it expands to; publish re-expands and must find the same board (#1075).
- Words as built 2–3 Sep 2026 (rollout 00–18): doors (`entry` as a list, #1072) · goal
  tiers with a payload key and their own `exit_reason` (#1063) · `on_repeat` /
  `debounce_minutes` / `restart_on_repeat` per door (#1058, #1074) · `wait_event.key:
  "$topic"` (branch on the letter's name), edge labels `timeout` and `else` (catch-all),
  `wait_event.match {payload, run}` (WHOSE letter a square hears — a call's outcome names
  its `enrollment_id`) (#1072, #1076) · `node.stage` labels (#1074) · `stages` ladder
  `{order, idle_minutes, on_idle, after_action_minutes, restart_on_repeat, overrides}`
  expanded by `outreach/ladder.py` into `at-`/`act-`/`after-` squares per stage (#1075) ·
  `on_publish: pin | migrate` (#1068).
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
| 4 | `workflow_version` | integer |  | **The EXECUTION PIN (ADR 0023, 3 Sep 2026)**: names the T25 row this run executes — written at entry as the latest version, changed only by a `migrate` publish or a migrate-forward (`POST /workflows/{id}/versions/{from}/migrate`), kept on exit as the audit fact ("what did this run execute"). The walker parks a run whose version row is missing rather than fall back to the live document. Still the did-the-edit-help read — goal_met rate GROUP BY version |
| 5 | `customer_id` | uuid | IX |  |
| 6 | `status` | text | CK | waiting · parked · exited. enrolled folded into waiting (it was waiting with an immediate wake); parked = attempts exhausted, held for the merchant to see and resume — errors never silently discard a run. Trail (3 Sep 2026, #1074): a letter the parked run's CURRENT square listens for also resumes it (waiting, attempts reset, last_error cleared) — an event is evidence the customer moved, so a parked journey is never unwatched |
| 7 | `current_node` | text |  | The token’s square — a STABLE node id, resolved against the live definition at each wake |
| 8 | `wake_at` | timestamptz | IX | The durable timer AND the lease: the claim pushes it forward by the lease interval, so a dead worker’s row self-heals — comes due again with no reaper. Immutable as a delay once entered (the timing she was promised); jittered on mass writes — 8,400 rows must never come due the same second. Partial index (wake_at, id) WHERE status=waiting. This is what replaces a workflow engine. **Trail (3 Sep 2026, #1061): the lease is also the GENERATION** — every walker write (advance, exit, park, record error) is compare-and-set on `wake_at = <the leased value>`; event-side writes (goal-cancel, a reply, a repeat patch) are unconditional and move `wake_at`, so a reply landing mid-visit makes the walker's write match zero rows and defer — the answer is never clobbered by the timeout path |
| 9 | `entered_at` | timestamptz |  | The run’s birth — the clock timeout arithmetic reads: timed_out is entered_at + the plan’s max age, and "stuck over 7 days" is one predicate on it |
| 10 | `exited_at` | timestamptz |  | The death clock — the retention sweep reads it: exited rows age out on a partial index over exited_at, which is most of what keeps the hot table small |
| 11 | `exit_reason` | text | CK | goal_met · timed_out · withdrawn · ejected · completed · **converted_elsewhere** (063, 3 Sep 2026: a goal TIER keyed to the run's own cart says "THIS cart recovered" = goal_met; any other order by the customer still ends the run — never nudge someone who just bought — but distinguishably; the CHECK was re-created under the explicit name `crm_workflow_enrollment_exit_reason_ck`, and a new reason is a migration that re-creates it). Tiers may end a run with goal_met · converted_elsewhere · withdrawn; timed_out · ejected · completed are the walker's own. (amended 31 Aug 2026, ratifying #1029). withdrawn = she said stop, exactly that; converting IS goal_met; ejected = the workflow was archived or the merchant cancelled the run; completed = the run executed its last node WITHOUT the goal firing — finished-without-converting, the funnel's denominator (goal_met ÷ (goal_met + completed) is the flow's conversion rate). NEVER presented as success in any UI — the label is "finished without converting". Late conversions are a REPORTING join (exited completed + goal event within the attribution window), never a reason to hold runs open: completion frees the open-run key for her next journey. errored is gone — errors park, never exit |
| 12 | `context` | jsonb |  | POINTERS to the spark plus the few small facts the sends will need — {chk_88412, 1850, blue kurti} — never the payload itself: e-91 already stores the cart verbatim, and a photocopy read at every wake bloats and goes stale (the artifact_ref habit). Random branch outcomes are recorded here BEFORE the send — a retry must never re-roll the dice. **Bookkeeping keys as built (one list, `outreach/nodes.py`, never template variables):** `source_event_id` · `entered_event_at` (the founding letter's own time — goals measure from it, G7) · `goal` (the letter that ended the run + its amount, for recovered revenue) · `phone` · `repeat_event_ids` / `repeat_items` (a repeat marks itself used; accumulate's list) · `facts.<square>` (each listened letter's scalar facts, namespaced per square, replaced not appended) · `latest_letter` (which square heard the most recent letter) · `reply_<node>` (cleared when the token leaves the square) · `lead_<node>` / `message_<node>`; `current_node` / `current_stage` are computed at fire time. Scalars ≤256 chars only, so the row stays small |
| 13 | `enrollment_key` | text |  | Defaults to the customer id; the partial unique is (merchant_id, workflow_id, enrollment_key) WHERE status <> ’exited’ (merchant-first per tenancy law — amended 31 Aug 2026 ratifying #1029). Keyed runs are how two open orders get two live WISMO flows — Dittofeed’s orderId lesson. RULED 31 Aug 2026: the key is chosen by the plan document (`entry.key` names the payload field; omitted = customer id) — the author declares what a run is about. As built (#1060, #1076): on a keyed plan the reenter/cooldown guards judge THAT key's history, not the customer's; a keyed ladder gives every listening square `match {payload: key, run: key}` so two applications never move on each other's letters |
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
  Cancel-on-goal: the goal event (e.g. orders/create) resolves open enrolments — per open
  run, judged by ITS pinned version's tiers, keyed-first, time-aware on the founding
  letter's `occurred_at` (#1063, #1070).
- Reads built 3 Sep 2026 (#1066): `GET /workflows/{id}/summary?since&until` (runs, by
  exit reason, open by status, median minutes to exit, recovered amount — one grouping-sets
  statement) and `GET /customers/{id}/runs` (her runs across every plan, in entry order —
  the journey while a funnel runs). Both admin-only per ADR 0007.

### crm.workflow_version (T25) — 8 columns *(sealed 3 Sep 2026 — ADR 0023; migration 064; owner outreach)*

One immutable row per publish: the document a run is pinned to. Rows only — never edits (BEFORE UPDATE trigger `crm_workflow_version_immutable_guard`), never deletes (ADR 0023 §5 as amended; the DELETE-refusing trigger is a named follow-up migration).

| # | column | type | keys | notes |
|---|---|---|---|---|
| 1 | `id` | uuid | PK |  |
| 2 | `merchant_id` | text | UQ·FK | Tenancy — first in the unique; half of the tenant-pinned FK |
| 3 | `workflow_id` | uuid | UQ·FK | Composite FK `(merchant_id, workflow_id) → crm_workflow (merchant_id, id)` — the 057 tenant-pin precedent |
| 4 | `version` | integer | UQ | `UNIQUE (merchant_id, workflow_id, version)`; the number T19 col 10 bumped and T20 col 4 pins to |
| 5 | `definition` | jsonb |  | NOT NULL — the document exactly as it became live (the draft, copied verbatim) |
| 6 | `on_publish` | text | CK | `pin` · `migrate` — a closed status enum, so a CHECK (law 11) |
| 7 | `published_by` | text |  | The publish route threads the caller's email |
| 8 | `published_at` | timestamptz |  | DEFAULT now() |

**Wiring**
- Written ONLY inside the publish atom (`plans._publish_in_txn`): validate → template check → copy → version row → (migrate) re-pin open runs. No ON CONFLICT: a second row for one version is a bug the unique must surface.
- Read by the walker and the entry consumer through `outreach/definitions.py` (one point read per `(workflow_id, version)`, process-local LRU of 512 — safe because rows are immutable and kept), by `GET /workflows/{id}/versions` (each with its open-run count), by migrate-forward (both documents read inside its atom), and by the template-retirement guard (`runs_referencing_template_query` joins open runs to their pinned document's send nodes).
- Backfilled at 064 from every live plan's current `definition`; open runs on unrecoverable older versions were re-pinned to the current one (exactly what they executed under 057).
- No partitioning, no `updated_at` (nothing updates): a plan publishes tens of versions, not millions of rows.

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
