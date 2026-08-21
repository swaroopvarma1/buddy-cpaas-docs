# 0019 — The phase-1 console extends loom, not a new app

**Status:** Accepted · **Date:** 2026-08-21 · Refines ADR 0007 (which deferred a
"merchant console" as post-M2) and delivers the UI half of ADR 0016 (customers
end-to-end WITH UI in phase 1).

## Context
Phase 1 ships four UI surfaces (U1–U4): console shell/auth, customers list, customer
360 with journey + why-didn't-it-send, and the WABA template manager. Clairvoyance is
a pure Python backend with no frontend anywhere in the repo, so the console needed a
home. Building one from scratch was the working assumption when the task map was cut.

Meanwhile **juspay/loom already exists**: the Breeze Buddy merchant dashboard
(SvelteKit + Tailwind 4, static-adapter build) with a full console shell — sidebar,
topbar, login + token auth, dark/light theming behind a two-axis token contract
(`data-bb-style` × `data-bb-theme`), a feature-flags admin surface, and — decisive —
tenancy scoping in `src/lib/console/scope.ts` whose hierarchy (admin All-workspaces →
reseller umbrella → merchant workspace) is exactly the reseller model the CRM canon
assumes. Merchants and internal ops already log into it.

## Decision
The phase-1 CRM console is **new routes inside loom's existing `(console)` group**,
not a new app and not a new repo.

1. **Placement.** `Customers` becomes a top-level sidebar item: `/customers` (list)
   → `/customers/[id]` (the 360). The template manager is channel-scoped:
   `/channels/whatsapp/templates`. No "CRM" grouping — the CRM is the console growing,
   not a wing bolted onto it.
2. **U1 collapses.** There is no shell or auth to build. U1 is re-scoped to wiring:
   the CRM routes, a typed API client for clairvoyance's `/crm/*` routers, tenancy via
   the existing `scope.ts` (every CRM request carries the scoped `merchant_id` — a
   divergent copy of that resolution is a data-isolation bug, per loom's own note),
   and the exposure flag.
3. **Exposure is a per-merchant feature flag.** CRM nav renders only for flagged
   merchants (the pilot) and internal admin/ops, using loom's existing feature-flag
   machinery. Rollout later is flipping flags, not shipping code. This supersedes
   ADR 0007's "post-M2 fast-follow" sequencing for the console while keeping its
   spirit: the `/crm/*` API surface remains ops/admin-shaped, and the console is its
   first client.
4. **Loom's design system governs.** CRM pages follow the console theme contract
   (`docs/CONSOLE-THEMING.md`): tier-2 semantic tokens / tier-1 `--bb-*` ramps, no
   colour literals (the CSS audit script fails the build on them), both styles ×
   both themes. The corpus visual language stays where it lives today — docs and
   diagrams — it does not style product UI.
5. **No new CSS dialect.** Loom's component audit documents real drift (four
   `.btn-primary` definitions, nine pagination implementations) and an extraction
   spec whose rule is *components wrap the class names `console.css` already
   defines*. CRM pages build with those classes/components and add page CSS only
   for what is genuinely new. We are the first new surface to land after that
   audit; we do not become its next data point.

## Consequences
- U1 shrinks from "console shell + auth" to "CRM wiring in loom" (routes, API
  client, scope, flag) — roughly a task of days, not weeks.
- The UI lane spans two repos: loom (U1–U4) and clairvoyance (`/crm/*` APIs). An
  API/UI change is two PRs; the API contract in `design/console-ui.md` is the
  hand-off between them.
- CRM inherits loom's auth model wholesale — reseller umbrella views of CRM data
  come for free structurally, but phase 1 flags only merchant workspaces.
- Phase-2/3 surfaces (segments, broadcasts, flow-builder, inbox) get their homes
  in the same nav; the inbox will sit beside the existing voice `/conversations`
  page, and that adjacency is a design question for phase 3, noted now.
- Design deliverables for U2–U4 are specified in `design/console-ui.md`.
