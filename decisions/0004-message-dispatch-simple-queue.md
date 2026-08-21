# 0004 — Message dispatch uses a simple DB queue, not the five-plane dispatcher

**Status:** Accepted · **Date:** 2026-08-19

## Context
Voice dispatch earned its Redis five-plane pipeline (ZSET schedule, leader-elected
promoter, channel semaphores) from hard per-number concurrency limits and ring-time SLOs.
Messages have neither: WhatsApp throughput is governed by Meta rate/quality tiers, not
channel counts, and the manifest already provides the primitives.

## Decision
Start with the simplest thing that is honestly scalable:
- Dispatcher drains `crm.message WHERE status='queued'` (partial index) with
  `FOR UPDATE SKIP LOCKED`; provider 429s back off; no transaction held across HTTP.
- Exactly-once effect via the manifest's `dedupe_key` partial unique index — not locks.
- Workflow timing lives on `workflow_enrollment.wake_at` (timer + lease in one column) —
  no workflow engine, no separate scheduler.

Re-evaluate only against measured evidence (queue latency at pilot volume), not
anticipation. Scale-out path if needed later: shard the drain by merchant, or promote
hot merchants to a Redis ready-list — without changing the manifest contract.

## Consequences
- No new Redis topology for phase 1; fewer moving parts for 10 engineers to integrate.
- Sub-second send latency is NOT a phase-1 goal; campaign/workflow sends tolerate
  seconds of queue delay.
