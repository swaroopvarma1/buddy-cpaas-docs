# How this system scales — growth is boring by design (sealed 1 Sep 2026)

*A teaching document. Read it when you're about to add a connector, an event, a node
type, or a channel — or when you're worried that we can't. The thesis in one
sentence: **every axis of "more" terminates in a registry with a pin test, so
growing the product never changes its shape — only its vocabulary.***

## The idea everyone must hold

Systems die in two ways as they grow: **scatter** (the fifth connector is wired in
eleven places, and place nine gets forgotten) or **cleverness** (an engine so
general nobody can predict it). We refuse both with one pattern, used five times:

> A closed vocabulary lives in ONE dict. Everything that needs per-entry behavior
> iterates or dispatches through that dict. A pin test makes a half-added entry
> fail CI. Adding an entry is therefore boring — and boring is the goal.

| Registry | Vocabulary | Lives in | Guards |
|---|---|---|---|
| `EXTRACTORS` (#1020) | event sources → decode | record | fixture conformance |
| `NODE_TYPES` (#1029) | workflow node words | outreach | Literal pin + `is_wait == (execute is None)` |
| `INGRESS` (owed, C6) | provider webhook doors | record | one verifier per bay |
| `CATALOG` (owed) | events + their fields | record | extractor↔catalog square, fields-in-fixtures |
| `CONSUMERS` (committed) | spine subscribers | worker_main registers | record imports nobody |

If you are about to add per-entry behavior with an `if/elif` in a second file —
stop. That is scatter beginning. Find the registry or make one (ask first: the
[ask-timing principle](#the-two-timing-rules) says guarantees land NOW).

## The worked example: adding Meta as a connector, end to end

The ritual every connector follows — four artifacts, ONE PR, roughly a day:

1. **Ingress entry** (`record/ingress.py`): how Meta proves it's Meta —
   `verify` (X-Hub-Signature-256 over raw bytes), `envelope` (topic/external_id
   from the body), `challenge` (the GET handshake). The door is three verbs:
   verify → store raw → 200.
2. **Extractor** (`EXTRACTORS["meta"]`): a pure function, payload in →
   `Extracted(handles, facts)` out. No I/O, no DB.
3. **Fixtures**: a folder of REAL recorded Meta payloads. These are simultaneously
   the extractor's tests AND the catalog's conformance proof.
4. **Catalog entry** (`CATALOG`): what the events are called, which fields they
   carry, their types, which ops each type allows, which fields are keyable and
   which ride into templates.

And then — this is the point — **you stop.** You do not touch the workflow editor
(its trigger picker reads the catalog). You do not touch the entry rules, the
walker, the dispatcher, the journey view, or a single UI file. On the next console
load, "Meta" appears in the trigger picker with its fields and operators, flows can
filter on them, and the walker executes those flows with zero new code. Ten
connectors cost ten of this same PR — not ten integrations.

## The five growth axes, and why each is safe

**More connectors** — the four-artifact ritual above. Constant cost, no downstream
change. The doors design (design/ingest-doors.md) keeps trust boundaries exact:
one envelope door for callers who speak our language, one registry-backed bay per
provider who speaks their own.

**More events per connector** — one catalog entry + one branch inside that source's
extractor. The UI grows itself.

**More node types in workflows** — one `NODE_TYPES` entry + one word in the
schema's Literal. The pin test fails CI if you add one without the other. (This is
not hypothetical: at only FOUR types, the pre-registry code had already half-wired
one — a `wait_event` first node that never waited. The registry made that bug
unwritable.)

**More channels on send nodes** — a channel value + an adapter behind `send()`.
The manifest (T16) and dispatcher are channel-agnostic; ONE send path absorbs
email and SMS the way it carries WhatsApp.

**More merchants** — everything tenant-scoped by law (merchant_id first in every
unique index; per-merchant plan reads). The deliberately GLOBAL work queues
(walker, dispatcher) scale by adding replicas, because correctness rides leases
and dedupe keys, never worker uniqueness. Add pods, not code.

## The four pressure points — each named, each wired to its trigger

Scaling honesty means saying where it WILL hurt and pre-deciding the fix, so the
person who hits the wall finds a door already drawn on it:

1. **Entry matching at event volume.** Every attributed event evaluates its
   merchant's live plans, in Python, inside the spine's pass — heavy event flow
   taxes the drain. Relief (ledger trigger): the **live-plan cache** — validated
   definitions cached by (workflow_id, version); plans are authored, not
   generated, so seconds of staleness is free. Detection: the queue-lag alert
   already fires.
2. **One global dispatch queue meets its first broadcast.** Correct today; but a
   10k-recipient blast queues beside COD confirmations with no priority. Relief
   (trigger: the W8/broadcast PR): **fairness lanes** — claim ordering by purpose
   root (transactional/utility before marketing) + per-merchant round-robin. A
   blast must never starve a confirmation call.
3. **Consumers multiply on the spine.** Entry rules today; segments tomorrow.
   Each adds per-event latency inside the pass. The consumer registry keeps them
   organized; replicas keep throughput; the discipline is *consumers stay cheap
   and indexed* — the review rules police per-row consumer cost.
4. **PgBouncer before replica scaling** (the standing P0). Every worker role
   multiplies pool connections. This is the one gate that is ops, not design —
   and it is the oldest unstarted line on the ledger. It lands BEFORE we
   multiply replicas, not after the incident.

## The two timing rules

Everything above obeys two rules worth internalizing, because they answer "should
I build this now?" without a meeting:

- **Guarantees land NOW.** If a small structure makes a bug class impossible
  (a registry + pin test, a unique index, a fail-closed default) it is built at
  the first opportunity, however small the current surface. Deferred guarantees
  decay silently — we have the scar to prove it.
- **Hygiene lands at its NAMED trigger.** File splits, caches, fairness lanes,
  per-table query files — these cannot decay silently (size and latency are
  visible), and their seams only become real when the material arrives. Each is
  written next to the exact PR that will trip it, and the review protocol's
  trigger sweep (skill Phase 0.5) makes hitting a trigger without delivering the
  committed change a MAJOR.

The result: adding to this system is deliberately unexciting. The exciting parts —
what to say to Priya, when to call, what a flow should do — stay in the plan
documents and the catalog, where product lives. The machine underneath grows by
vocabulary, never by surgery.

Refs: design/ingest-doors.md · design/event-catalog.md · design/worker-runtime.md ·
modules/01-record.md · modules/05-outreach.md · design/execution-ledger.md
(§follow-ups holds every named trigger).
