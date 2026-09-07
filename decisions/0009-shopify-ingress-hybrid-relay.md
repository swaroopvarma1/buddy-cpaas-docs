# 0009 — Shopify ingress: anchor relays now, direct subscriptions later

**Status:** Accepted · **Date:** 2026-08-19

## Context
The Breeze Shopify app (OAuth, shop tokens, webhook subscriptions) lives in **anchor**.
Clairvoyance has WooCommerce/Breeze ingress but nothing from Shopify. Both pilot flows
and the consent import ride on Shopify events reaching `event_raw`.

## Decision
**Hybrid.** Phase 1: anchor forwards the needed topics — `checkouts/create`,
`orders/create`, customer/consent payloads — to clairvoyance's webhook endpoint with s2s
auth (the existing Breeze-webhook pattern). `connector_installation` abstracts the hop:
the shopify door row's transport is "via anchor relay". Cutting over to clairvoyance
registering its own subscriptions later is a connector change, not a redesign.

## Consequences
- Anchor team gets one bounded task: forward three topic families with s2s signing.
  This is a cross-team dependency on the phase-1 critical path — track it explicitly.
- `event_raw.source='shopify'` regardless of transport; `external_id` = Shopify webhook
  id so the relay can retry safely (dedupe absorbs duplicates).
- Clairvoyance holds no Shopify tokens in phase 1.

## Trail — 7 Sep 2026 (#1057 / nautilus#201)
The relay now carries WRITES back as well: order tags and notes travel as the `shopify`
connector's action transport (`providers/shopify/via_nautilus.py`, the only file that knows
nautilus exists in that direction). Clairvoyance still holds no Shopify token — the T11 door
row exists with a NULL `credential_id`, and that NULL is the per-shop migration switch.
"Cutting over" remains a connector change: the transport file and the fork below `perform`.

