# Outreach — build guide

Owns: `crm.workflow` (T19) · `crm.workflow_enrollment` (T20) · `crm.broadcast` (T17) ·
`crm.broadcast_recipient` (T18) · `crm.segment` (T15) · `crm.segment_member` (T21) ·
`crm.campaign` (T22) · `enrol()` + `tick()` · `broadcast.schedule()`. Diagram:
`../diagrams/05-outreach.html`. Squad: Pod C (phase 2).

## Build it like this

- **The workflow is ONE document; a run executes the VERSION it entered under** (ADR
  0023, built 3 Sep 2026 — reverses the earlier "never an execution pin"): `definition`
  jsonb `{entry, nodes, edges, goals, exits, purpose_key, on_publish, stages?}` is the
  LATEST document (entries, console); every publish writes an immutable
  `crm_workflow_version` row (T25) and `enrollment.workflow_version` pins the run to it.
  Edits land in `draft`; publish validates, checks every send template is approved in the
  T23 registry (#1065), copies, bumps `version`, writes the version row, and under
  `on_publish: migrate` re-pins every open run — one atom. Under `pin` (default) the
  stranding refusals do not apply; under `migrate` the validator still blocks deleting an
  occupied node and changing `entry` while runs are open. The walker resolves the pin
  through `definitions.py` (one point read, process-local LRU; a missing version row is an
  honest park, never a fallback to the live document) and reads the live row for STATUS
  only. Fixes reach pinned runs through `POST /workflows/{id}/versions/{from}/migrate?to=N`
  (the stranding laws as a pure function + the target's template check). **Node ids are
  minted once and NEVER regenerated.** The decision rule per plan: runs of minutes-to-hours
  → `migrate` (a template fix should reach every waiting run); runs of days-to-weeks →
  `pin`.
- **wake_at is the timer AND the lease**: the walker's claim pushes wake_at forward one
  lease window, so a dead worker's row self-heals when the clock passes — **no reaper**.
  Immutable as a delay once entered; jittered on mass writes (8,400 rows must never
  come due the same second). Partial index `(wake_at, id) WHERE status='waiting'`.
- **Runs park, never vanish**: attempts++ ON the claim (poison runs count against
  themselves); exhausted → `parked` with `last_error` readable on the merchant's
  screen. `enrollment_key` (partial unique WHERE not exited) allows keyed concurrent
  runs (two orders = two WISMO flows). **RULED 31 Aug 2026 (Swaroop): the key is
  plan-level document vocabulary — `entry.key: "<payload field>"`, omitted = the
  customer id.** The author declares what a run is ABOUT: abandonment omits it
  (bursts coalesce), WISMO writes `"order_id"` (parallel threads). The entry
  processor reads `payload[entry.key]` and passes it to `enrol()`; goal-match-by-key
  (a keyed goal closes only its own run) AND reply-match-by-key (a reply wakes only
  its own run) — BOTH BUILT 3 Sep 2026: goal tiers carry `key {event, run}` (#1063), every
  consumer write names the run it is about (#1070), a listening square may say WHOSE
  letter it hears via `match {payload, run}` (#1076), and the walker CONSUMES `reply_<node>`
  when the token leaves the square (#1072) so a revisited square never resolves on a stale
  answer. Admission guards on a keyed plan judge that key's history (#1060).
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

**Mechanics — BUILT 3 Sep 2026 (#1058, rollout phase 00; carried manas-narra's #1041
with two guards folded in)**: `outreach/repeat.py` (vocabulary, PURE `repeat_plan`,
`apply_repeat`) + ONE idempotent UPDATE (`patch_open_run_query`): `WHERE status='waiting'
AND current_node = <the door's start square> AND enrollment_key = $key AND NOT
(repeat_event_ids ? event_id) AND context->>'source_event_id' IS DISTINCT FROM event_id`
— a run past its first square is never patched (unless the door says `restart_on_repeat`,
#1074: then the repeat re-arms whatever square the run stands on, "KYC retried, the timer
restarts"); the founding event is never a repeat (P9); the debounce may only EXTEND the
window — `GREATEST(wake_at, now() + N)` (P10); `refresh_max` refuses NaN/inf; the words
judged are the OPEN RUN'S version's door (#1070). A letter the run's current square
answers is its wake, never also its repeat (#1075). `repeat_count` is a fact templates may
read; `repeat_event_ids` / `repeat_items` are bookkeeping.

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

## Positions sealed by the first implementation (#1029 — folded from its review-phase decisions catalog, 31 Aug 2026)

Where canon named the mechanism, #1029 chose the numbers and shapes. Sealed as built:

- **Walker step mechanics**: a node's time is written when the token ARRIVES on it
  (arrival scheduling); one claim executes consecutive immediate nodes, bounded at 10
  (runaway-document guard); `attempts` resets on a successful step — only CONSECUTIVE
  failures park; the claim is one `UPDATE … SKIP LOCKED … RETURNING` (lock + lease +
  attempts in one statement — no transaction spans node work); transient backoff =
  lease × 2^(attempts−1), capped 1h, ±20% jitter.
- **Document shapes** (T19 names the sections; these are the words): `nodes[0]` is the
  start square; an edge is `[from, to]` (+ `on` only out of a `wait_event`); node
  vocabulary `wait · send · call · wait_event`, registry-backed (ruling above)
  — **`wait_event` folded into `wait` on 17 Sep 2026, see "One wait" below**;
  `entry.where` = typed conditions `[{field, op, value}]` ANDed with the topic (#1047, 4 Sep
  2026; the equality map retired by migration 069). `wait_event` = topics +
  key + minutes (event OR timer, whichever first); the consumer writes
  `context.reply_<node> = payload[key]` + `wake_at = now()` on the run still standing
  on that node (late/repeat replies change nothing); the walker takes the edge whose
  `on` matches, `timeout` when the alarm fired first, exits `completed` when no edge
  matches.
- **Goal match is customer-level** (keyed goal-match lands with the first keyed flow)
  and counts only events with `occurred_at` after the run's `entered_at`. **Cooldown
  anchors on the latest run's `entered_at`.**
- **Call idempotency** = deterministic lead id `uuid5(run_id, node_id)`; the PK absorbs
  a lease retry. Leads skip the Redis dispatch nudge (`schedule_lead` lives in
  `app.ai`; the reconciler heals within 60s — outreach importing app.ai is a coupling
  not worth one minute on a 30-minute flow). `enrollment_id` stamped by a separate
  UPDATE through the legacy layer's own accessor (the table's owner writes it,
  ADR 0010).
- **`purpose_key` lives on the plan document root** (one flow, one purpose; required by
  the publish validator once a send node exists; copied onto every manifest row).
  Per-node override can come later without breaking root-level plans. Template
  language (T23 keys name+language; T16 stores name) decided with T23.
- **Run ops numbers**: retention 90d · sweep DELETE 500/pass, hourly, run by the
  walker's claim callable (one loop per pod; the owner keeps its table small) · lease
  300s · max attempts 3 · poll/batch from the shared worker knobs. **Resume** =
  parked-only, merchant-scoped, `attempts=0`, `wake_at=now()`, `last_error` survives
  until the next successful step clears it.
- **Merchant payload contract**: push producers send `customer_mobile_number` +
  `customer_name` (one generic extractor; missing phone → quarantine `no_handle`);
  every other scalar payload key ≤256 chars rides run context → lead payload →
  `{placeholder}` resolution. `external_id` = the checkout id for all of a checkout's
  events, topic prefix keeping keys unique. Workflow call templates hand-configured
  with `initial_offset=0` (phase 1).
- **Composition-root exception**: record's `contracts.py` does NOT export the event
  worker's pass (`run_pass`/`observe_processed_event`) — record's workers call
  outreach's consumer while outreach imports record's contracts, so exporting the pass
  there closes an import cycle; `app/crm/worker_main.py`, the one composition root,
  takes the pass from `record/workers.py` directly.

Still open after #1029: per-node `purpose_key` override (with T23) · W5 branch-limit lift
owner. (Keyed goal-match and reply-match landed 3 Sep 2026 — see the rollout section.)

## The workflow rollout as built (2–3 Sep 2026, phases 00–18 · #1056–#1077)

Twenty ordered, PR-sized phases dispatched from `clairvoyance/docs/crm/workflow-rollout/`
(README, PIPELINE, `context/reading-notes.md` = the intent, `context/nits.md`); every phase
one PR, one commit, red tests first. The vocabulary the corpus now carries, by phase:

| Phase / PR | Delivered |
|---|---|
| 00 #1058 | Repeat entries (`on_repeat`, `debounce_minutes`) with the P9/P10 guards — above |
| 01 #1059 | A `wait_event` letter without the key does NOT end the window (B1); entry compare by meaning (B3); `live` on an unpublished draft is a 422, not a 500 (B4) |
| 02 #1060 | Admission guards scope to the enrollment key on keyed plans (B2) |
| 03 #1061 | Walker writes are compare-and-set on the leased `wake_at`; event-side writes win (P1) |
| 04 #1062 | `crm_event_raw.attempts` spent by the claim; poison rows quarantine after `CRM_EVENT_MAX_ATTEMPTS` (P2; record's) |
| 06 #1063 | Goal TIERS `{topics, key?, exit_reason}`, keyed-first; `converted_elsewhere` (063); time-aware on the founding letter's `occurred_at` (`context.entered_event_at`, G7); `customer_has_event(where=)` on record's contract |
| 07 #1064 | Plan templates validated in CI (`docs/crm/plans/*.json`) + runbooks |
| 08 #1065 | Publish refuses a send node whose template the registry does not hold approved (G12) — `template_status` / `registers_templates_for` on connectivity's contract |
| 09 #1066 | `GET /workflows/{id}/summary` (counts by exit reason, open, median minutes, recovered amount from `context.goal.amount`) · `GET /customers/{id}/runs` (G9) |
| 10 #1067 | ADR 0023 |
| 11 #1068 | T25 `crm_workflow_version` (064) · `on_publish` · publish writes the row, migrate re-pins |
| 12 #1069 | The walker executes the pin (`definitions.py` LRU); missing row = park |
| 13 #1070 | The consumer's two reads: her open runs (goals, listening, per pinned version; `cancel_run` / `resume_run_by_id` by run id) then the live plans' latest (entries) |
| 14 #1071 | Migrate-forward route · versions list · template-retirement guard (409) under the advisory lock (`shared/locks.py`) registered from the composition root · retention DROPPED (versions kept) |
| 15 #1072 | `wait_event.key: "$topic"` · `entry` as a list of doors `{topic, start, …}` (shared admission words at the top level fold into every door) · `reply_<node>` cleared on advance |
| 16 #1074 | Letters' scalar facts ride into the run under `context.facts.<square>` (namespaced, replaced per square); `current_node` / `current_stage` reach templates; a PARKED run moves when its square hears a letter; `restart_on_repeat` re-arms any square (G8); send variables drop bool/None |
| 17 #1075 | `stages` ladder → `at-/act-/after-<slug>` squares + one labelled arrow per later stage, expanded PURE and idempotent by `ladder.py` at create/draft/publish (the walker never learns the word); loan-dropoff is ONE pinned board (`docs/crm/plans/loan-dropoff.json`, `key: application_id`, cooldown 1h); `latest_letter` wins the call's facts |
| 18 #1076 (call half) | `call.completed` mirrors `enrollment_id`; `wait_event.match {payload, run}`; edge label `else` (catch-all out of a listening square — a connected call's outcome is the buddy template's own word); keyed ladders match on the key; `cart-recovery-fallback.json`. The MESSAGE half (receipts, STOP → suppression) waits for #1040/#1052 |
| 18 #1140 (message half) | **A reply is ADDRESSED, not matched** (11 Sep 2026): a letter's `replied_to` → `crm_message_provider_id_uq` → the row → `source_id` = the run (canon T16 col 7/8/14). `connectivity.contracts.send_behind()` reads it; `outreach/reply_attribution.py` turns it into a run; `entry.py` resolves ONCE per letter and narrows to that run. Nothing declared by author, validator or console, and no plan republished. Addresses the RUN, never the square — the manifest names the SENDER and the listener is another node; which square resolves stays the resume statement's own WHERE. Narrows only: an unthreaded letter, a reply to a send that was not ours or not a run's, a contradicting dedupe_key, a source with no `replied_to`, or an answered run that has EXITED all behave exactly as before. One behavioural change: a receive-first "any inbound" square in another plan no longer hears a letter answering a different OPEN run's send |
| catalog #1047 (4 Sep) | Typed `entry.where` + `entry.key` as catalog paths (dot-walks + code-layer derived fields through record's `field_value`) · send `variables: {blank: fact}` = EXACTLY what is posted (positional or named, one style; empty = nothing) · publish gathers the merged catalog → pure `catalog_laws.py` (undeclared field / ill-fitting op / unknown choice value / unkeyable key refused; an unknown topic may trigger but not filter) · seen-vs-matched on the flow list, computed on read · a topic two sources declare refuses publish (`AmbiguousTopic`) until `entry.source` is ruled · `lead_request_id` = the enrollment key on a keyed plan |
| 19 | Permission-gate wiring (G1) — DEFERRED until #1021 lands `may_contact()`; see the B-lane |

Reserved edge/branch words: `timeout` · `else` · `$topic` (all in `outreach/nodes.py`).
Files as built: `entry.py` (the consumer) · `enrol.py` · `walker.py` · `nodes.py`
(NODE_TYPES + the bookkeeping keys) · `plans.py` (validator + publish atom) ·
`definitions.py` (the pin read + LRU) · `versions.py` (migrate-forward, list, the
retire guard's count) · `repeat.py` · `ladder.py` · `runs.py` (summary, journey, resume,
retention sweep) · `api.py` (12 routes; `customer_router` under `/customers`).

**Hygiene triggers that FIRED in this rollout and are OWED (structure PR, not a
feature PR)**: `outreach/db/queries.py` is 814 lines over THREE tables (the ruled
subfolder trigger — `db/queries/<table>.py` · `accessors/<table>.py` ·
`decoders/<table>.py` — fired at #1068 and did not land); `outreach/db/accessor.py` 502.
Named triggers still ahead: `entry.py` (427) splits the consumer's three reads when the
segments consumer lands; `schemas.py` (388) becomes a package at the next document
family.

## Do NOT

- **Don't build a workflow engine.** No BPMN, no Temporal, no state-machine framework.
  wake_at + the document IS the engine; everything else is a liability with a
  roadmap.
- **Don't fall back to the live document for a pinned run** (ADR 0023): a run whose
  version row is missing parks honestly. And **don't delete a version row** — an exited
  run's pin must keep answering "what did this run execute"; the DELETE-refusing trigger
  is a named follow-up migration.
- **Don't let the walker overwrite an event-side write.** Walker writes are conditional
  on the leased `wake_at`; goal-cancel, replies and repeat patches are unconditional and
  win (#1061).
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
- **The node vocabulary is registry-backed from the start (RULED 31 Aug 2026 — Swaroop,
  #1029 review):** one dict in an `outreach/nodes.py` concern file — `NODE_TYPES:
  {type: (validate, execute, is_wait)}` — the publish validator iterates it, the walker
  dispatches through it, and adding a type is ONE entry (half-adding a type — validator
  knows it, walker doesn't — becomes structurally impossible). The schema's `Literal`
  stays in schemas.py (leaf shapes import nothing), pinned to the registry keys by a
  test. Precedent: record's `EXTRACTORS` registry (#1020) — same shape, same reason.
- **The fifth word: `action` (ruled by Swaroop in Slack — "Approach 3"; recorded 7 Sep 2026
  with #1057 in review).** A square says `{type: "action", connector, action, args}` and
  NOTHING else — no URL, no credential, no transport: connectivity holds the merchant's door
  and decides how the write travels (a relay today, the provider directly once we hold its
  token), so a published plan never changes when the carrier does. `args` are the ACTION's own
  contract (the connector's pydantic model, validated at publish); values may be
  `{placeholder}` and resolve from the run's facts at fire time — the send node's explicit
  `variables` law, applied to the fourth verb: **what the run is about (an order id) is an ARG
  the author names as `{id}`, never something a provider face digs out of the run's context**.
  Two failures, two classes: a DEFECT (unknown connector/action, args that do not fit, a 4xx)
  parks; a BAD MOMENT (timeout, 5xx, 429) raises as itself and the lease ladder re-sends.
  Supersedes #1078 A/03's `http` node with an author-supplied URL.
- **`outreach/nodes.py` becomes the `outreach/nodes/` package (ruled by Swaroop 7 Sep 2026 at
  #1057).** One file per word — `wait.py · wait_event.py · send.py · call.py · action.py`, each
  holding its validate + execute — the shared run-context helpers in one sibling
  (`context.py`: bookkeeping keys, `run_facts`, placeholders), and `NODE_TYPES` ASSEMBLED in
  the package `__init__` from the word modules (the `SPEC_MODULES` precedent: the assembly
  point is the package's `__init__`, not a re-export hub). Trigger fired by #1057: the file
  crossed ~500 lines (431 → 596) as the fifth word landed. The Literal pin test is unchanged. **As built (#1057 merged 7 Sep; #1101 trims the hub)**: `nodes/` = `__init__` (the
  registry: `NODE_TYPES` · `NodeSpec` · `is_wait` — and NOTHING else, pinned) · `spec.py`
  (`NodeSpec`, `NodeParked`) · `context.py` (`run_facts`, `send_variables`, `lead_request_id`,
  `reply_key`, `without_reply`, `is_bookkeeping`) · one file per word with its `validate` +
  `execute`. Importers name the file they mean.

## One `wait` — the vocabulary folded (ruled by Swaroop 17 Sep 2026, built in #1151)

`wait` was `wait_event` with no topics. Its whole validator was "minutes > 0"; the only other
difference was two registry flags. That is a CONFIGURATION, not a word, and the cost showed the
moment a wait-shaped feature arrived: the calling window had to be declared once and reasoned
about twice, and its publish law was written against the `is_wait` FLAG rather than the word —
the code already treated wait-ness as a property. Every future wait feature would have paid the
same tax, and the half-retired string match in `unenumerable_squares` (left by #1114's N1
retirement) was the trap waiting for the second listening word.

**One word, three forms.** `topics` is the discriminant; everything else follows from it.

| Field | Required | Meaning |
|---|---|---|
| `minutes` | see below | how long the alarm runs (> 0 when given) |
| `topics` | optional | absent = a plain timer; present = it also listens |
| `key` | when `topics` | how the letter answers (`$topic`, or a payload field) |
| `match` | optional, only with `topics` | which letter is about THIS run |
| `window` | optional | the hours the timer may fire in |

**`minutes` is optional, and what fills it composes.** The duration and the hours answer
different questions — `minutes`/`topics` decide HOW LONG, `window` decides WHEN THAT MAY ACT —
so the window applies to whatever the duration resolved to rather than replacing it:

    duration = now + minutes        if minutes
             = run's end of life    elif topics     ("listen as long as this run may live")
             = now                  else            (a bare window: "wait for the hours")
    alarm    = opens_at(duration, window) if window else duration

A wait with none of minutes, window or topics is refused at publish: it waits for nothing.
`wake_at` is ALWAYS set — it is the timer AND the lease (canon T20), migration 058 CHECKs it,
and the claim is `wake_at <= now()`, so a run with no alarm would never be claimed and the
self-healing that means there is NO REAPER would be gone. The listening form's alarm is the
run's own `max_age` plus a minute: the claim runs on the database clock and the max-age check on
the worker's, and an alarm exactly at end-of-life could be claimed a moment before the worker
agrees it expired — the run would then take a timeout arrow, or exit `completed` instead of
`timed_out`. This replaces the workaround the first lending board shipped with, where `listen`
carried `minutes: 43200` — the plan's own `max_age_days` copied by hand, which would have gone
silently wrong the day that number moved.

**The window holds the TIMER, never a letter.** A letter moves the run the moment it lands, so
the night's newest event is what the morning acts on and a goal ends the run before any call
exists. That asymmetry has a publish law behind it: a `call` reached from a windowed square by
any arrow but its timer's — directly, or through squares that act in the same visit — is
REFUSED, because a letter is never held and that call would be queued in the shut hours. The
guarantee is the law's, not the comment's.

**It reads the PLAN's clock, not the customer's.** This is a scheduling window, NOT the
quiet-hours control: quiet hours for messages are permission's at send time (ADR 0018), voice
sits outside that gate (ADR 0010) so its hours live in the plan, and the dialler's own calling
hours remain the last check on when a phone actually rings. The customer's own timezone (the
ADR 0018 ladder), days of the week, and a window on the `stages` ladder are NOT built.

**`wait_event` stays readable forever.** T25 version rows are immutable and thousands of them
carry the old word, so the `Literal` keeps it and a `mode="before"` validator reads it as a
`wait` with topics — lossless, it always carried them. Publish REFUSES it in a new document, the
shape the retired equality map already uses. `listens(node)` and `branches(node)` become
node-derived (`condition` and `split` keep their static `branches`), which is what closes the
`unenumerable_squares` string match in the same pass.


Refs: 05-audiences.md + 06-outreach.md (corpus) · ADR 0004 / 0010 / 0016.

## The playbook — the plan fills the agent's holes (ruled by Swaroop 18 Sep 2026)

**One agent template, edited only to change how the agent behaves.** Every word it says that
depends on the customer — the opening hook, the step-by-step walk, the lender's screen notes —
is chosen by the PLAN from the event and handed over finished. The agent never picks and never
function-calls for it. Brief: `visuals/playbook.html`.

**Why not the two obvious places.** Not the vendor's schema registration (`item_lookup`): the
words are the merchant's, the registration keys on one field, `item_format` caps at 160 chars,
and one registration is shared by every plan on the topic — the exact failure §The `list` ruling
removed. Not a transformer node: a square that transforms anything is a DSL in jsonb with no
validator, and a box that neither sends nor waits nor calls is clutter. Not one agent template
per lender either: the core agent gets copied N times and a turn-taking fix becomes N edits.

**The shape** — one key on the document, beside `stages`:

```json
"playbook": {
  "lines":  { "hook_offered_free": "जी आपने {product_name} के लिए …", "step_open_app": "सबसे पहले Flipkart app खोलिए…", "fibe_notes": "…" },
  "blocks": {
    "hook_line": [ { "when": [ {"field":"context.loan_application_status","op":"is","value":"OFFERED"},
                               {"field":"context.no_cost_emi_applicable","op":"is","value":"yes"} ], "say": "hook_offered_free" },
                   { "say": "hook_default" } ],
    "walk":      [ { "when": [ {"field":"context.lender_name","op":"is","value":"Fibe"} ],
                     "say": ["step_open_app","step_cart_order","step_fibe_verify_bank", "…"] },
                   { "say": ["…generic…"] } ]
  }
}
```

Two rules hold it: **every piece of text lives in `lines`, exactly once** (a block names lines and
never holds text — reword `step_open_app` once and every walk changes); **`say` is a line name or an
ordered list of names** (one name renders as its text; a list renders as `- name: "text"` per line,
in order — the ORDER/LINES sections of a prompt, generated). Rows are judged in document order;
the first whose `when` holds wins; **the last row has no `when`** (the condition square's `else`).
`when` is the sealed where-grammar over the condition square's field grammar (`context.*`,
`facts.<node>.*`, `customer.*`) — no new vocabulary beyond the four words `playbook` · `lines` ·
`blocks` · `say`.

**A block is a fact.** It is never sent to the agent as such: at fire time the node that builds
outward data merges the rendered blocks into what it already builds — the call node's lead
payload (`{hook_line}` lands in the prompt like any variable), the send node's `variables`
right-hand side, the action node's `args`. No node learns the playbook exists. The one limit is
the existing one: a WhatsApp blank may carry no line break, so a list-block (`walk`) mapped into a
send is refused at publish and a single-line block (`hook_line`) goes anywhere.

**Laws.** Publish: every block ends in a row with no `when`; every name in `say` exists in
`lines`; every `{hole}` in a line is a declared fact (the send-variables right-hand-side law:
a catalog variable for the plan's topics or a run fact); every `when` field is one
`predicates.field_problems` accepts; a list-block is never mapped into a WhatsApp blank; no line
break inside a single line. Fire time: **a leftover `{hole}` parks the run** naming the hole —
the agent's substitution is one pass (`template/utils.py`) and will not catch it; a line fills
from facts only, never from another line; **blocks render into the payload, never into context**
(T20 col 12: the row stays small); the chosen line name of each single-line block is recorded as
`playbook_<node>` (bookkeeping, beside `split_<node>`) for the funnel.

**Deliberately not built.** Trimming the walk to the steps still pending: the prompt already
reads `pending_steps` from the event and says "never re-walk what the profile shows as done";
if server-side trimming is ever wanted it is more `when` rows keyed on the state, no code. UPI
versus bank mid-call stays with the LLM. Sharing `lines` across plans: promote to a
merchant-owned store WHEN a second plan needs the same walk, not before. The console needs a
playbook editor (a lines list, blocks with `when` rows) — the same gap as `window` and `match`.

**As fitted to `flipkart-checkout-nudge-new` (Sharifa, v3, paused).** The board is untouched:
`call-1` · `call-2` · `call-3` already share one template (the constraint, already in practice),
every wait already listens on fifteen topics with `match`, and `on_repeat: refresh_latest` keeps
`context.loan_application_status` / `line_status` / `lender_name` current — the same keys
`retarget-route` already reads, so the `when`s read facts proven present. With the playbook, the
three call squares say different hooks as she progresses — `call-2` after a `KYC_COMPLETED`
letter speaks the KYC hook — with zero edges added.

**Build.** `outreach/playbook.py` (PURE: `laws()`, `resolve()`, `HOLE`); `schemas.py`
`PlaybookBlock{when, say}` · `Playbook{lines, blocks}` · `WorkflowDefinition.playbook`;
`plans.validate_definition` extends with `playbook.laws`; three merge sites — `nodes/call.py`,
`nodes/send.py` (via `send_variables(..., extra=)`), `nodes/action.py` — pinned by one test
(a precondition honoured at one call site and not the others is the #1143 finding); `playbook_`
joins `_BOOKKEEPING_PREFIXES` in the same PR; a fixture plan under `docs/crm/plans/` and its README
row. Tests prove each law by injecting what it forbids, and the red check.
