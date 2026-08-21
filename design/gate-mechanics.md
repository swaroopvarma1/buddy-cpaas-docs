# Gate mechanics — may_contact() implementation spec

**Status:** Final — unblocks task B5 · **Date:** 2026-08-21 · Decisions: ADR 0018.
Owner: Pod B (the pair). Companion: `modules/03-permission.md` (the build guide),
`diagrams/03-permission.html`.

## 1. When the verdict happens: the last responsible moment

The gate runs **at dispatch time**, not at queue time. Senders (walker, transactional
consumer, human reply, agent) write the manifest row `status='queued'` WITHOUT a
verdict; the dispatcher claims the row, calls `may_contact()`, and on yes sends
immediately. A consent withdrawal between queueing and sending is therefore always
honored. (Human replies are interactive: the reply handler runs gate+send inline —
same code path, no queue wait.)

## 2. The send token — an in-process handshake

- On ALLOW the gate mints an opaque token: Redis key `gate:tok:{uuid}`, value
  `{merchant_id, customer_id, channel, purpose_key, decision_id}`, **TTL 60s**,
  consumed by `send()` via atomic **GETDEL**. Missing, expired, or mismatched binding
  → send refuses (manifest reason `token_invalid`).
- Because verdict and send happen milliseconds apart in the same worker, the TTL is
  slack, not a scheduling budget. Redis down → token ops fail → refuse (fail closed,
  same posture as every other input).

## 3. The four inputs (all mandatory; any missing/NULL/error/unknown → NO)

**3.1 Consent-state probe** — one indexed read of `consent_state` for
(merchant, customer, channel, purpose resolved most-specific-first):
`status='granted' AND (expires_at IS NULL OR expires_at > now())`.

**3.2 Suppression** — `platform.identity.is_suppressed` probed by the customer's own
handle values against the unique (kind, value) index. One boolean. DB error → blocked.

**3.3 Frequency caps** — Redis fixed-window counters keyed
`freq:{merchant}:{customer}:{channel}:{d|w}:{local-date|iso-week}` where the date is
the **customer-local** date (their resolved timezone). `INCR` + `EXPIRE` (25h / 8d
safety TTLs). **Incremented on ALLOW** (before the provider call — overcounting on a
failed send is the conservative direction). Redis unavailable → NO, reason
`freq_unavailable`.

Defaults (per purpose root):
| Purpose | Daily | Weekly |
|---|---|---|
| marketing.* | 1 | 4 |
| transactional.* | 10 (runaway-workflow circuit breaker) | — |

Per-merchant configurable via the single resolver, under platform **ceilings**:
marketing can never exceed 3/day · 10/week however configured.

**3.4 Quiet hours** — evaluated in the customer's resolved timezone:
- `marketing.*`: default window **09:00–21:00** customer-local. Merchants may tighten;
  the platform **floor is 08:00–22:00** — no config may widen past it.
- `transactional.*`: exempt from quiet hours (still frequency-capped).

**Timezone resolution ladder** (the rung that answered is recorded in the verdict):
1. declared (customer said so)
2. observed (in-conversation / order evidence)
3. **derived from phone country code where the country has exactly one timezone**
   (+91 → Asia/Kolkata — deterministic, not a guess; multi-timezone countries skip
   this rung)
4. merchant-default timezone (explicit merchant config)
5. still unknown → **NO**, reason `tz_unknown`

## 4. Refusal semantics: defer-once for quiet hours only

- `quiet_hours` refusals return `next_allowed_at` (window open in the customer's
  timezone). The dispatcher reschedules that attempt ONCE (`not_before =
  next_allowed_at`) and re-runs the **full** gate then — a fresh verdict, so an
  overnight STOP still wins. Both verdicts land in decision_log.
- Every other reason (`no_consent`, `suppressed`, `frequency_cap`, `tz_unknown`,
  `paused_binding`, `template_not_approved`, `token_invalid`, `*_unavailable`) is
  **final for the attempt**: manifest `status='blocked'` with the reason. Callers never
  retry around the gate; a workflow decides its next move by its own rules.

## 5. The verdict record

Every call — allow AND refuse — writes one decision_log row (`decision_kind=
'send_or_hold'`) whose `chosen` jsonb dumps the working state honestly: verdict,
reason on refusal, each check's result, the timezone rung that answered, and the
config knobs as frozen at decision time (caps, window, merchant overrides) — because
they are mutable at home. `decision_id` is stamped onto the manifest row in the same
transaction.

## 6. Configurability principle

Every policy knob in this spec — windows, caps, merchant-default timezone — is
**per-merchant configurable within the platform floors/ceilings**, read exclusively
through the single config resolver, and frozen into each verdict's decision_log row.
No knob may cross a floor/ceiling regardless of configuration; there is no override
parameter, and none of these knobs can express one.

## 7. Failure matrix (the fail-closed contract, testable)

| Input failure | Verdict | Reason |
|---|---|---|
| consent_state row absent | NO | no_consent |
| consent expired (predicate) | NO | consent_expired |
| suppression probe DB error | NO | suppression_unavailable |
| Redis down (caps or token) | NO | freq_unavailable / token_invalid |
| timezone unresolved after ladder | NO | tz_unknown |
| outside window (marketing) | NO + next_allowed_at | quiet_hours (defer-once) |
| cap reached | NO | frequency_cap |
| any unrecognized input state | NO | fail_closed_default |

Acceptance for B5: property tests over this matrix — every row must hold with the
corresponding decision_log entry present.
