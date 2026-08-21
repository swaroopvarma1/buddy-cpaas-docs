# 0005 — Pilot: Shopify/Breeze merchants, abandoned-checkout + order-confirmation

**Status:** Accepted · **Date:** 2026-08-19

## Context
Designing channels and workflows in the abstract invites speculative scope. A concrete
pilot defines what "done" means for phase 1.

## Decision
Pilot merchants are **Shopify merchants on Breeze**. Two flows define the first cut:

1. **Abandoned checkout** — Shopify `checkouts/create` arrives (webhook →
   `event_raw`), customer created/attached via `resolve()`, a workflow entry rule
   enrolls them, the walker wakes after the configured delay, re-checks the goal
   (`orders/create` cancels), passes the gate, and triggers **a call or a WhatsApp
   message per the workflow's nodes** — voice via the existing lead machine, WhatsApp
   via the new send path.
2. **Order confirmation** — `orders/create` (COD focus, as today's webhook ingress
   already does) → transactional send through the same gate + manifest.

**Broadcast** (deliberate launches into workflows) follows as the next increment after
the two reactive flows work — it reuses the same walker and send path.

## Consequences
- The existing WooCommerce/Breeze webhook → lead path stays untouched for voice; the new
  Shopify → event_raw path is the CPaaS front door from day one.
- Success criteria are per-flow funnels readable from the manifest + decision log
  ("why didn't it send" answerable for every skipped customer).
- Mixed-channel treatments (call + WhatsApp in one workflow) are in scope for the pilot —
  the workflow walker must treat voice sends as first-class nodes from the start.
