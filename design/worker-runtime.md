# Worker runtime — one image, role flags (ADR 0006, mechanics)

How the three CRM workers (event worker · walker · dispatcher) actually run: what the
role flag is, how code branches on it, how it deploys, and the shared scaffold every
loop runs inside. Companion to module rules §Workers and diagram 07.

## The precedent we extend, not replace

Clairvoyance already ships ONE image with ONE entrypoint (`CMD uv run python run.py`
→ uvicorn) and already runs a long-lived loop inside the app lifespan behind an env
flag — the voice dispatcher's `ENABLE_DISPATCHER`. ADR 0006 seals the CRM version of
the same move: **same image everywhere; CRM workers are separate PODS selected by a
role env flag.** Nothing new is invented at the process level; what changes is that a
worker pod's job is its loop, and no user traffic ever reaches it.

## The role flag

One env var, read once at startup, registered in the config resolver (module rules:
no loose env vars):

```
CRM_ROLE = api (default) | event-worker | walker | dispatcher
```

- **It is an env var, not a build arg and not a CLI flag** — the image is identical
  across all four; the deployment manifest is the only place a role is stated. That
  is the whole point: rebuild nothing to change what a pod does.
- **One role per pod.** A pod is its role. No pod runs two loops; no pod runs a loop
  AND serves API traffic. (The legacy `ENABLE_DISPATCHER` voice flag stays as-is
  until buddy standardizes; it folds into the role scheme then.)
- **Unset = api.** A developer running `run.py` locally gets the API server, exactly
  as today. Running a worker locally is one prefix:
  `CRM_ROLE=event-worker uv run python run.py` in a second terminal.

## How code branches on it

The entrypoint does not change. The branch lives in the app lifespan, next to where
`ENABLE_DISPATCHER` already branches — boot everything shared (env, config resolver,
logging, pools, Redis), then either serve or drain:

```python
# app/crm/worker_main.py — the ONLY registry of roles (closed dict, no discovery).
# At the crm PACKAGE ROOT beside api.py (the mount-point precedent) — NOT in
# shared/: it imports module contracts, and shared/ is leaf-only by law.
ROLES = {
    "event-worker": lambda: run_drain_loop(claim_events,   process_event,  interval=1.0, batch=100),
    "walker":       lambda: run_drain_loop(claim_due_runs, walk_run,       interval=5.0, batch=50),
    "dispatcher":   lambda: run_drain_loop(claim_sends,    dispatch_send,  interval=1.0, batch=50),
}

# app/main.py lifespan (sketch)
role = CRM_ROLE  # from the resolver
if role != "api":
    set_log_context(crm_role=role)
    app.state.worker_task = asyncio.create_task(ROLES[role]())
```

Why the worker still boots FastAPI/uvicorn: `/healthz` liveness comes free, and the
shared init (config, pools, migration-consistent code) stays one path. The event-loop
contention argument against in-process workers doesn't apply — a worker pod receives
zero requests because it is never selected by the Service.

Adding a role = one line in `ROLES` + a manifest — and a review. The dict is closed
on purpose; dynamic discovery is how frameworks are born.

**The same file is the composition root for slots** (modules/00 §11): at import it
registers `consume_attributed_event` into record's consumer slot (#1046) and
outreach's `template_references` into connectivity's retire-guard slot (#1071); the
INGRESS entries register the same way (ruled 2 Sep 2026). `app/main.py` imports
`worker_main`, so the API pod is wired identically — a route that needs a slot
(template retire) never runs unregistered. Roles as of 3 Sep 2026: still three
(event-worker · walker · dispatcher); the walker's claim callable also runs the
hourly exited-run retention sweep — versions are NOT swept (ADR 0023 §5).

## The shared scaffold (the generic module — deliberately small)

The three loops are the same machine with different contents. `app/crm/shared/worker.py`
(~100 lines) owns ONLY the loop mechanics:

- **poll cadence + jitter** — the sleep law: full batch → loop again immediately;
  empty poll → back off 1s → cap 5s, ±20% jitter so replicas never thunder together.
- **per-item error isolation** — one bad row logs loudly and never kills the batch.
- **graceful shutdown** — SIGTERM sets a stop flag; the loop finishes the current
  batch, commits, exits 0.
- **empty-queue backoff + repeated-DB-error backoff** (same curve, louder log).
- **metrics** — batch size, rows/s, queue depth, oldest-pending age. Lag is THE
  alert; CPU is not.
- **heartbeat** — touches a timestamp each iteration (probe-able).

Signature: `run_drain_loop(claim, handle, *, interval, batch)`. A worker is config
plus two callables. **Two claim styles** (sealed 28 Aug 2026, from the A2 build):
*lease-style* (walker) — claim commits immediately, `handle` processes each item
after; *txn-style* (event worker, dispatcher) — SKIP LOCKED locks live only inside
the claim transaction, so the claim callable runs the WHOLE pass and commits, and
`handle` becomes a post-commit observer. Name txn-style claims honestly
(`run_pass`/`drain_once`, not `claim_*`). **Consumers run per-row inside the row's
savepoint** — never as a batch step after the loop: a batch-level consumer failure
either rolls back the whole pass (queue stalls on one poison rule) or gets
swallowed (stamped rows lose their reactions forever). **Savepoints enter the
atomic grammar via a `savepoint(txn)` helper** on shared/db re-exported through
the doors — logic never calls driver API on the handle. **Pool floor: a worker
needs pool ≥ 2** — the pass holds one connection while contract calls
(`resolve()`, `assert_facts()`) open their own `atomically` transaction on a
second; pool=1 self-deadlocks permanently. Those contract calls commit
INDEPENDENTLY of the row's savepoint — replay-safety there is idempotency, not
atomicity, and docstrings must say which. What must stay per-module, never in the scaffold: the **claim
query** (SKIP LOCKED batch for events/sends; the wake_at lease push for the walker)
and the **handling logic** — they live in each module's `db/queries` + logic files.
The scaffold never learns what a row means. Guardrail: past ~150 lines it is becoming
a framework — stop. No plugin registries, no dynamic discovery.

## Deployment shape (four manifests, one difference)

One image, built once per commit. Four Deployments stamped from the same template —
the ONLY differences are `CRM_ROLE`, replicas, and whether a Service selects the pod:

| Deployment | CRM_ROLE | Replicas (launch) | Service/ingress | Probes |
|---|---|---|---|---|
| crm-api | api | as today | yes | liveness+readiness /healthz |
| crm-event-worker | event-worker | 1 | **none** | liveness /healthz (readiness moot — no traffic) |
| crm-walker | walker | 1 | none | liveness /healthz |
| crm-dispatcher | dispatcher | 1 | none | liveness /healthz |

- **Rollout:** the image tag updates in all four together (same commit) — API and
  workers can never disagree on schema, extractors, or contracts. Rollback is also
  all-four, atomic by tag.
- **Deploys need zero drain choreography.** Rolling update: new worker starts, old
  gets SIGTERM, finishes its batch, exits. Overlap of two replicas is SAFE BY DESIGN
  — SKIP LOCKED / the wake_at lease is exactly what makes two claimants harmless.
  `terminationGracePeriodSeconds` (30s) just has to exceed max batch time.
- **Scaling** = the replicas field, per role, independently. Gate: the connection
  budget — every replica carries a small pool (~5) through PgBouncer; the math
  api_pool + Σ(worker_replicas × 5) < PgBouncer server pool is checked BEFORE any
  replica bump (ADR 0002/0006; PgBouncer lands before A2).
- Workers arrive with their lanes: phase-1 day one has only crm-event-worker;
  walker and dispatcher are each "copy the manifest, change one env value" later.

## Env summary

| Var | Role | Where set |
|---|---|---|
| `CRM_ROLE` | which loop this pod is (or api) | Deployment manifest only |
| `CRM_WORKER_BATCH` / `CRM_WORKER_INTERVAL` | per-role overrides of the scaffold defaults | manifest, optional — defaults live in code |
| pool size | worker pools are small (~5) | shared config, per role |

All registered through the config resolver — a loose `os.environ` read is the scar
the module rules already ban.

## What this buys (and what it defers)

Buys: no version skew ever; one build; workers scale/restart/crash independently of
API traffic; a worker deploy cannot take down calls or webhooks; the boundary for a
future split into a true separate service is already the role flag (ADR 0006's exit
clause). Defers, deliberately: autoscaling on queue depth (KEDA/HPA) — not until a
real backlog exists; a dedicated worker entrypoint without uvicorn — only if the
free /healthz ever becomes a problem, which is unlikely.
