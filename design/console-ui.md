# The phase-1 console — customers, journeys, templates in loom

**Decided 2026-08-21 (ADR 0019).** The console is not a new app: phase-1 CRM UI is
new routes inside **juspay/loom**, the existing Breeze Buddy dashboard. This doc is
the design for the UI lane (U1–U4) — what each surface is, what it calls, and the
loom-specific rules that keep us from becoming the console's next CSS drift statistic.

## The frame

| Question | Answer |
|---|---|
| Where | loom `src/routes/(console)` — `/customers`, `/customers/[id]`, `/channels/whatsapp/templates` |
| Nav | `Customers` top-level in the sidebar; templates under the existing Channels surface |
| Who sees it | Per-merchant feature flag: pilot merchants + internal admin/ops (loom's flag machinery) |
| Tenancy | Every request scoped through loom's `src/lib/console/scope.ts` — never a hand-rolled merchant-id resolution |
| Styling | Loom's console theme contract — semantic tokens, no colour literals, works in both styles × both themes |
| Backend | clairvoyance `/crm/*` routers (`/crm/customers`, `/crm/consent`, `/crm/templates`, …) via one typed API client |

## U1 — CRM wiring (the collapsed shell task)

Loom already owns shell, auth, theming and scope. U1 is now exactly four things:

1. **Routes** — the three pages above, in the `(console)` group, flag-gated.
2. **API client** — one module (`src/lib/api/crm.ts` or the house pattern) wrapping
   `/crm/*`; every call takes the scoped `merchant_id` from `scope.ts`. No page
   calls `fetch` directly.
3. **Sidebar entry** — `Customers` added to the nav array in `Sidebar.svelte`,
   rendered only when the flag is on for the scoped workspace.
4. **Flag** — a `crm` feature flag per merchant, administered from the existing
   `/admin/feature-flags` page.

## U2 — Customers list (`/customers`)

A **lean ops table with segment-ready filters** — not a segment builder (that is
phase 2, ADR 0016).

- **Search:** server-side, one box — name / phone / email.
- **Filters (fixed set):** consent state (per purpose) · reachability (has a
  WhatsApp-capable identity) · source connector · last-activity window.
- **The filter contract:** filters serialize to the same predicate shape the phase-2
  segment engine will evaluate. Segments later = saved predicates + counts, not a
  second filter language. The predicate schema is agreed with the A6/B-pod owners
  before the first filter ships.
- **Columns:** name · primary phone (formatted) · consent glance (chip per purpose)
  · reachable-on chips (channel) · source · last activity · created.
- **No saved views in phase 1.** Pagination is server-side; counts are estimates
  where exact counts are expensive.
- **Empty states matter:** a merchant with a fresh connector sees "importing from
  Shopify — N so far", not a blank table.

## U3 — Customer 360 (`/customers/[id]`)

**A unified timeline with a standing-state rail.** Sequence is the debugging story
(checkout → message → no delivery → call), so sequence is the page spine.

- **Header:** name, primary handles, source, created; merchant-scope safe.
- **Timeline (main column):** every event type interleaved chronologically, one
  card per event, filterable by type. Cards, minimum anatomy:
  - *Message* — template name, purpose, delivery ticks (queued → sent → delivered
    → read), and for blocked/failed sends an **inline expansion showing the recorded
    gate verdict** (the `decision_log` reason: which rule refused, what the state
    was). Why-didn't-it-send lives where the eye already is — it is not a separate
    screen.
  - *Voice call* — direction, attempted-or-connected, outcome, duration (ADR 0017's
    journey voice arm; read from `lead_call_tracker` by stamped `customer_id`).
  - *Order / checkout event* — type, amount, source connector.
  - *Consent change* — grant/revoke, purpose, channel, provenance (import vs
    user action).
- **State rail (right):** the standing facts that are not events —
  - identities/handles with channel badges;
  - consent per purpose × channel, as it stands now;
  - **a live "can we reach them now?" check** — calls the gate's dry-run for
    (customer, purpose, channel) and shows verdict + reason (quiet hours until
    21:00 IST, cap reached 1/day, no consent…). This widget is read-only: it
    explains the gate, it never bypasses it (the gate stays fail-closed at
    dispatch time, ADR 0018).
- Timeline reads the spine (`event_raw` projections) — it never re-derives history
  by phone-matching at read (ADR 0017 rule).
- **Card dispatch is `source_kind`, never sniffing.** V01 rows carry their origin as
  a literal stamped by each union arm (`event` · `message` · `call` · `consent` ·
  later `chat`); the timeline renders `switch (source_kind)` and `(source_kind, id)`
  deep-links to the underlying row. For `event` rows, the timeline API extracts the
  card's display fields (order total, item count) from the raw payload for the
  current page only — bounded display decoding, gone in P2 when TRIGGER materializes
  the summary as columns.

## U4 — Template manager (`/channels/whatsapp/templates`)

**Full lifecycle with preview.** Meta rejections are routine; the operator iterates
here, not in curl.

- **List:** name, language, category, status badge — draft / pending / approved /
  **rejected with Meta's reason surfaced verbatim** — last synced (C7's status sync).
- **Create/edit:** guided form — category, language, header (text/media), body with
  `{{n}}` variables + required sample values, footer, buttons (quick-reply / URL).
  Validation mirrors Meta's rules client-side before submission.
- **Preview:** a WhatsApp-style bubble rendered live from the form, sample values
  substituted. The preview is the cheapest rejection-prevention we can build.
- Submission goes through the C7 Tech Provider APIs; the UI never talks to Meta
  directly.

## Build rules for the lane (read before the first component)

1. **Wrap, don't mint.** Loom's `docs/CONSOLE-COMPONENT-AUDIT.md` measured the cost
   of every page rewriting its own CSS (four `.btn-primary`s, nine paginations).
   New CRM pages use the classes/components `console.css` already defines; page CSS
   is for the genuinely new (the timeline card, the preview bubble).
2. **No colour literals, anywhere.** Semantic tokens per `docs/CONSOLE-THEMING.md`;
   the audit script fails the build otherwise. Test every screen in both styles
   (`minimal`, `breeze`) × both themes (dark, light).
3. **Scope through `scope.ts`, always.** Loom's own comment says it: a divergent
   copy of merchant resolution is a data-isolation bug, not a style issue. Same law
   as the canon's `merchant_id NOT NULL`.
4. **The API contract is the hand-off.** UI (loom) and API (clairvoyance) are
   different repos and different PRs; endpoint shapes for U2/U3/U4 are agreed with
   the A6, A12/B4 and C7 owners against this doc before either side builds.

## Dependencies (unchanged from the task map)

U2 ← A6 (customers read API) · U3 ← A12 (journey projection), B4 (decision log) ·
U4 ← C7 (template registry + Tech Provider APIs) · U1 ← A5 (auth glue, now mostly
moot) + the loom flag.
