# Scaling in three stages — traffic growth, not feature growth (PROPOSAL, 4–7 Sep 2026)

*[how-it-scales.md](how-it-scales.md) answers "what happens when we add a channel, a
connector, a vendor" — the answer being that nothing happens, by design. This document
answers the other question: **what happens when the same shape has to carry a thousand
times the letters.** Sealed nothing yet: every stage past the first is a proposal with a
named trigger and the rulings it needs listed.*

**The ruling in one sentence: not five migrations in a year — THREE.** What runs today,
one hardening pass on the same shape, and then the final architecture built ONCE, with
the log, the columnar store and client-side events landing together rather than as three
separate migrations.

Interactive version (dial any traffic level, the pod ledger recomputes):
`claude.ai/code/artifact/76fc79d1-6cac-4378-ba29-84b61a0982cd`

## Stage 1 — what runs now (release `2a00ef44`)

One Postgres primary, one image with three worker roles selected by `CRM_ROLE`, WhatsApp
out through `send()`. The tray IS the queue: `crm_event_raw`'s partial index drained
`FOR UPDATE SKIP LOCKED`.

| | |
|---|---|
| Arriving today | ~1/s (pilot) |
| Safe sustained | ~30/s, one event-worker |
| The door's ceiling | ~300–500/s per API pod |
| The pass's ceiling | 40–100 letters/s per worker replica |

**The pass is the ceiling, not the door.** Decode is microseconds (spec cached, discovery
once per topic ever); the cost is 6–12 sequential DB round trips per letter — resolve()'s
probes, assert_facts, the entry consumer's reads, enrol's atom, the stamp. Without a
pooler the whole system is capped near four worker processes, because each holds up to 15
direct connections.

**Leave when** the queue-lag line fires more than once a day, or a second event-worker is
needed. Both need the pooler first — which is the whole of stage 2.

## Stage 2 — same shape, hardened (weeks; zero new architecture)

Same tables, same contracts, same code shape. Nothing here is a design change.

- **PgBouncer** — the standing P0. `api_pool + Σ(worker replicas × 5) < server pool`,
  checked before every replica bump. #1018 makes the asyncpg pool safe behind it.
- **Read replica** — the reads that are not the queue: the 7-day GROUP BY behind
  seen-versus-matched, catalog counts, the journey, run summaries.
- **`may_contact()` live** (#1021) — consent, quiet hours, frequency caps, a decision row
  for every verdict including refusals. Until it lands, sends are gated by the global
  stop-list alone.
- **Flows editor** — the console builds on `GET /catalog`.
- **Partitions** on `crm_event_raw` (monthly RANGE) and `crm_message` (RANGE on
  created_at); retention becomes a partition drop.
- **The live-plan cache** for the entry read (pressure point 1) — the difference between
  70 and 100+ letters/s per replica.
- **Fairness lanes** in the dispatcher (pressure point 2), with the first broadcast.
- **Load test + alerts** — 15 minutes over the recorded Shopify fixtures against staging.
  Every number in this document stops being an estimate.

**Reaches ~2k–5k/s sustained** (170M–430M letters a day) on one primary. **Leave when**
letters pass ~5k/s sustained, or the day storefront tracking is switched on. Either one
means stage 3 — and stage 3 is built whole.

## Stage 3 — the last architecture (build once)

Everything that would otherwise be three separate migrations, landing together: the log,
the columnar store, client-side events, and the cell shape. **Nothing after this is a
migration; more traffic is more cells.**

- **Kafka or Redpanda** — topics `letters` (256 partitions by merchant hash, keyed by the
  dedupe key, 7-day retention = the replay window), `attributed`, a plans change feed, and
  a dead-letter topic (quarantine at scale; replay = re-produce).
- **The doors PRODUCE instead of INSERT** — one changed line each, after verify and stamp.
  Every refusal still happens before anything is produced.
- **Three streams**: sink consumers (`COPY` 5k/batch into the raw table, which becomes the
  archive), decode + identity (the same engine, the same catalog, with a handle cache in
  front of `resolve()`), and entry rules (plans in memory from the change feed; only a
  match writes).
- **ClickHouse** — every client event, 10–20× compressed, 90 days hot, Parquet after. A
  billion events a day is 50–100 GB here versus ~1.5 TB in Postgres with indexes; that
  ratio is the entire reason the component exists.
- **Collector + pixel/SDK** — Shopify's Web Pixel (clientId + consent flags) and a D2C
  snippet or the Jitsu SDK, both batched and consent-aware.
- **Derive job** — sessionises clicks into ONE letter per moment (`browse.abandoned`,
  `product.interest`), and the rollups (`spend.monthly` carrying amount + previous). Forty
  clicks become one letter; a flow reacts to it exactly like an order.
- **`anon_id`** — one merchant-scoped handle column, last in the probe order (the igsid
  precedent). A visitor is a customer row holding only that handle; the first letter
  carrying both `anon_id` and a phone staples the two cards. No new identity rule.
- **Segments** (T15/T21) — the same where-grammar compiled to SQL across the sink and the
  spine. "Everyone above 5k this month" is a segment; "she just crossed 5k" is a derived
  letter.
- **Sharding** — the raw table and identity by merchant hash, 4–8 primaries.

**Untouched by all of it**: the catalog (stays cold), the permission gate (fail-closed,
never cached), the walker, the dispatcher, `send()`, the console, and every contract —
`resolve()`, `may_contact()`, `send()`, `enrol()`. None of them can tell where the queue
lives. **A cell** is this whole stack for a slice of merchants behind a router; the only
thing that stays global is suppression, because a person who said stop is stopped
everywhere.

## The pod arithmetic (and the correction)

**An earlier sketch of this said ~100 door pods and ~200 decoders at 50k/s. That was
wrong** — it costed unbatched requests and one-row-at-a-time work. Producers batch (a
beacon or a Meta callback carries dozens of letters), so 50k letters/s is only *hundreds*
of HTTP calls a second; and a stream consumer with a handle cache does 5–10× what the
current pass does per process, because most letters are for a customer we already know.

| | Stage 1 · 1/s | Stage 2 · 2k/s | Stage 3 · 50k/s + tracking |
|---|---|---|---|
| API / door pods | 2 | 5 | 7 |
| Pass / streams | 1 | 20 | 26 decode + 21 entry + 9 sink |
| Walker | 1 | 2 | 7 |
| Dispatcher | 1 | 2 | 13 |
| Collector + derive | — | — | 13 |
| **Application pods** | **5** | **29** | **96** |
| Stateful nodes | 2 | 6 | 28 |

Model: doors at 400/s unbatched and ~8,000/s batched, the current pass at 70–100
letters/s, a stream consumer at 2,000–2,500/s, sink COPY at 6,000/s, sends capped by
provider throughput.

**Is that how big systems run?** One stateless API service is typically 3–20 pods, set by
redundancy and burst headroom rather than throughput. One stream consumer group is 10–60,
hard-capped by partition count (a 257th consumer on 256 partitions does nothing). A large
company runs thousands of pods, but that is hundreds of *services* × 3–10 replicas, not
one pipeline with a thousand workers. **One pipeline needing 300+ is a smell**: it is
almost always cheaper to delete a round trip than to run the pods — our own handle cache
is worth more than a hundred extra processes. And pod count is the number people quote;
the brokers, the primaries and ClickHouse are the number finance sees.

## Rulings owed before stage 3 starts

1. **`anon_id`** as a merchant-scoped handle, and the shape of tracking consent (the
   consent ledger keys on a way of reaching a person; a cookie is not one).
2. **The clickstream sink** — a partitioned Postgres table for the first tracked store,
   ClickHouse from ~50M events/day.
3. **Jitsu versus our own collector** — Jitsu ships the SDK, a Shopify pixel and the
   batching endpoint; ours is ~a week and one fewer component to run. Either way Jitsu is
   never the identity authority, the flow engine, or the consent authority.
4. **Whether client-side tracking is a new lane** with its own owner.

## Refs

[how-it-scales.md](how-it-scales.md) (feature growth, the four pressure points) ·
[worker-runtime.md](worker-runtime.md) (the connection budget) ·
[entry-points-and-the-event-spine.md](entry-points-and-the-event-spine.md) ·
[event-catalog.md](event-catalog.md) (the decode engine the streams reuse unchanged) ·
[modules/01-record.md](../modules/01-record.md) §capacity.
