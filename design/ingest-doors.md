# Ingest doors — how facts enter the spine (A9)

*Sealed 1 Sep 2026 (Swaroop + review discussion around clairvoyance#1025/#1029 and
nautilus#195). The rule in one sentence: **one mailroom, two kinds of entrances —
and the split between them is exactly as wide as the trust boundary, not one line
wider.***

Everything after a door is ONE system: the same `crm_event_raw` table, the same
event worker, the same extractors, the same entry rules. A Shopify order that
arrived via nautilus and one that arrived via a direct webhook are
indistinguishable one line past their doors. The doors differ only in the three
things that happen *before* storage: who is trusted, how they prove it, and what
ritual their reply requires.

## The two door kinds

| | **Envelope door** | **Provider doors** |
|---|---|---|
| Route | `POST /ingest/events` — ONE, forever | `POST /ingest/webhooks/{provider}` — one bay per provider (`GET` too, only where the provider demands a challenge) |
| Who | Callers who speak OUR envelope: nautilus relay, future internal services (loyalty, billing) | Providers who speak their own: Shopify direct, Meta (receipts + inbound), payment gateways |
| Auth | Our s2s credential, verified as a route `Depends` (`verify_s2s_caller`) | THE PROVIDER'S ritual: Shopify = HMAC-SHA256 over the raw bytes with the shop secret; Meta = `X-Hub-Signature-256` + a GET `hub.challenge` handshake at subscribe time |
| Body | IS our envelope: `{merchant_id, source, topic, external_id, payload, occurred_at}` | IS the payload, in their schema — the envelope is ASSEMBLED from headers (merchant from shop-domain / phone-number-id, topic from `X-Shopify-Topic` or the body, external_id from the webhook id) |
| Reply | Our receipt contract: `{id, duplicate}`; 503 = retry (dedupe makes it safe) | Whatever they demand: bare fast 200 (Shopify disables slow webhooks), challenge echo (Meta) |
| Never | — | never `crm` in the path (ADR 0022) |

### The side-by-side that explains everything

```
POST /ingest/events                        POST /ingest/webhooks/shopify
x-s2s-token: <our token>                   X-Shopify-Hmac-Sha256: kJx9f...
{ "merchant_id": "shop.myshopify.com",     X-Shopify-Topic: orders/create
  "source": "shopify",                     X-Shopify-Shop-Domain: shop.myshopify.com
  "topic": "orders/create",
  "external_id": "ord-42:created",         { ...the entire body IS the payload,
  "payload": { ...the order... } }           in Shopify's own schema... }
```

### Why they can never merge (each reason alone is disqualifying)

1. **Auth dispatch is the bug.** A combined route must first GUESS which verifier
   to run from whichever headers are present. Ingestion is permission-adjacent —
   a forged `orders/create` ("COD, Rs 50,000, phone = the victim's") that lands on
   the spine causes the entry rule to enrol and the walker to CALL the victim.
   Fail-closed law: permission-adjacent code never guesses. Two doors = one
   verifier per route, declared as the route's dependency (rules: a route without
   its auth dependency is a BLOCKER). One door = hand-rolled auth dispatch, the
   exact code that fails open.
2. **The bodies differ in kind.** One door's handler still forks into two complete
   code paths on line one (envelope-as-body vs envelope-from-headers) — nothing
   merges; the seam just becomes invisible and untestable.
3. **The reply rituals differ** — receipts vs bare-200 vs challenge echo.

## The provider-door mechanics: the INGRESS registry

Per-provider differences are table-shaped, so they live in ONE registry — the
house pattern, third instance (`EXTRACTORS` #1020, `NODE_TYPES` #1029):

```python
# app/crm/record/ingress.py (concern-named logic file, to build)
INGRESS: Dict[str, IngressSpec] = {
    "shopify": IngressSpec(verify=_verify_shopify_hmac, envelope=_shopify_envelope),
    "meta":    IngressSpec(verify=_verify_meta_sig, envelope=_meta_envelope,
                           challenge=_meta_challenge),
}
```

- `verify(request) -> merchant_id | raise` — the provider's signature over the RAW
  bytes; secrets via the config resolver, never per-route hardcoding.
- `envelope(headers, body) -> (source, topic, external_id, occurred_at)` —
  ENVELOPE FIELDS ONLY. The door never parses payload contents; a semantic
  problem is quarantine's job, not the door's.
- `challenge` — optional, only for providers with a subscription handshake.
- The door's whole job, any provider: **verify signature → store the letter raw →
  200 fast.** Unknown `{provider}` = 404. Adding a provider = one registry entry +
  one secret. A provider door that parses payloads or answers non-200 on semantic
  failure is drift (the provider's retry storm is the punishment).

## When and who decodes: the event worker — the SAME one, already running

Doors never decode; storage is the whole door job. Decoding happens seconds later
on the **`CRM_ROLE=event-worker` pod** running record's pass (`run_pass`, merged
#1020) — one worker for BOTH doors' letters; doors never get their own workers.

```
door stores (processed_at NULL)
  └─ event-worker pod claims a batch (SKIP LOCKED)
       └─ per row, inside the row's savepoint:
            EXTRACTORS[source](payload) -> Extracted(handles, facts)
            resolve()        -> customer found-or-created
            assert_facts()   -> profile claims (drift-only)
            consumers        -> entry rules: workflows enrol / goals cancel
                                (consumer REGISTRY — committed next record PR)
            stamp            -> customer_id + processed_at on the row
       no handle found -> QUARANTINE (kept forever, marked, queue never blocks)
       decoder fixed   -> replay() re-drives quarantined rows (raw is immutable)
```

Quarantine + replay is what makes "decode later" safe rather than lazy: a decoder
bug is never data loss — it is a backlog with a repair tool.

## Where provider mappings live: the EXTRACTORS registry

The mapping "Shopify's `customer.default_address.phone` → our `phone` handle" is a
**pure function**: payload in, `Extracted(handles={phone,email}, facts={name,…})`
out. No I/O — tested against a folder of RECORDED real payloads as fixtures, the
only honest test for provider schemas. It registers under its source key in
record's `EXTRACTORS` (built, #1020; voice sources registered; default flat shape
for unknown sources). Everything downstream is provider-blind.

- Adding Meta end-to-end = one ingress entry + one extractor + fixtures. Nothing
  else learns Meta exists.
- Provider API-version drift: the extractor grows a tolerant branch; `replay()`
  heals whatever quarantined in the gap.
- Many topics in one source dispatch INSIDE that source's extractor — the registry
  stays keyed by source.

## Folder structure (record module, exact — current code + what's owed)

```
app/crm/record/
  api.py         journey_router · ingest_router (envelope door, #1025)
                 · webhook_router /ingest/webhooks/{provider}   [TO BUILD]
  ingest.py      record_event() / ingest_event() store logic     [built]
  ingress.py     INGRESS registry (verify/envelope/challenge)    [TO BUILD]
  events.py      cross-module reads (customer_has_event)         [built #1029]
  workers.py     the pass: run_pass + EXTRACTORS                 [built #1020]
                 (consumer registry: committed next record PR)
  extractors/    shopify.py · meta.py — the split lands when a   [at trigger]
                 second real source's extractor arrives; the
                 registry stays the single assembly point
  db/            door · accessor · queries · decoder             [built]
```

**The decoder's obligations grew (1 Sep 2026):** a new source/topic ships FOUR
things in one PR — ingress entry · extractor · fixtures · **catalog entry** (see
[design/event-catalog.md](event-catalog.md): the schema registry that drives the
workflow editor's pickers, the where-grammar, and phase 2's segment builder).

## Build order — exists vs owed

| Piece | State | Trigger |
|---|---|---|
| Envelope door `/ingest/events` | #1025 in review | — |
| Shopify extractor + fixtures | **owed** | the moment nautilus#195 shadow goes live (else its letters quarantine — harmlessly, but pointlessly) |
| `webhook_router` + `ingress.py` INGRESS registry | **owed** | first direct provider — Meta delivery receipts (C6) |
| Consumer registry in the pass | committed | next record-touching PR |
| `/crm` prefix removal | ruled (ADR 0022) | with #1025 |

Refs: modules/01-record.md · design/entry-points-and-the-event-spine.md ·
design/worker-runtime.md · ADR 0020 (stamp) · ADR 0022 (paths) · canon T13.
