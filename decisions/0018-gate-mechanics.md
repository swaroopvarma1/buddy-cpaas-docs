# 0018 — Gate mechanics: dispatch-time verdicts, timezone ladder, TRAI-shaped defaults

**Status:** Accepted · **Date:** 2026-08-21 · Full spec: `design/gate-mechanics.md`.

## Context
The gate's implementation details were the last open design input in phase 1 (task
B5). The hard product question: the law says unknown timezone → no send, and day-1
Shopify customers have no declared timezone — strictly applied, the pilot sends
nothing.

## Decision
1. **Verdict at the last responsible moment**: the dispatcher (or inline handler)
   calls may_contact() immediately before sending — never at queue time — so consent
   changes between queue and send are always honored. The send token is a short-lived
   in-process handshake (Redis, single-use GETDEL, 60s TTL, bound to
   customer+channel+purpose; Redis down → refuse).
2. **Timezone resolution ladder**: declared > observed > derived from phone country
   code where the country has exactly one timezone (+91 → IST; deterministic, not a
   guess) > merchant-default config > refuse (`tz_unknown`). The answering rung is
   recorded in every verdict.
3. **Quiet hours**: marketing.* 09:00–21:00 customer-local by default (TRAI-shaped —
   familiar to Indian merchants); transactional.* exempt. Platform floor 08:00–22:00.
4. **Frequency caps**: marketing 1/day AND 4/week per (merchant, customer, channel);
   transactional 10/day as a runaway-workflow circuit breaker. Platform ceilings:
   marketing ≤3/day, ≤10/week. Fixed daily windows on the customer-local date,
   INCR+EXPIRE, incremented on ALLOW, Redis-down → NO.
5. **Defer-once for quiet hours only**: refused-for-window sends reschedule ONCE to
   window-open and re-run the FULL gate there (fresh verdict — an overnight STOP
   wins). Every other refusal is final for the attempt.
6. **Configurability principle**: every knob is per-merchant configurable within the
   platform floors/ceilings, via the single config resolver, frozen into each
   verdict's decision_log row. No knob can express an override.

## Consequences
- The pilot sends honestly from day one: +91 numbers resolve timezone
  deterministically; the decision trail shows exactly which rung answered.
- B5 is unblocked; its acceptance is the spec's failure matrix as property tests.
- The "guesses never decide" law is preserved: country-code derivation for
  single-timezone countries is classified as derived-deterministic, not inferred;
  ambiguous countries fall through to explicit merchant config or refusal.
