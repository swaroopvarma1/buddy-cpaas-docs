# Module build guides

One guide per app/crm module: **how to build it, and how NOT to build it** — with the
why on every rule. Engineers: read `00-module-rules.md` first (the laws every module
inherits), then your module's guide before writing code. Reviewers: these guides are
the review checklist.

| Guide | Module | Pod | Phase |
|---|---|---|---|
| [00-module-rules.md](00-module-rules.md) | The laws every module obeys | all | — |
| [01-record.md](01-record.md) | Event spine: ingest, event_raw, consumers, replay | A | P1 |
| [02-identity.md](02-identity.md) | resolve(), assert_facts(), merge, crm.customer | A | P1 |
| [03-permission.md](03-permission.md) | Consent stores, the gate, importers, suppression | B (the pair) | P1 |
| [04-connectivity.md](04-connectivity.md) | Doors, pipes, templates, manifest, send(), receipts | C | P1 |
| [05-outreach.md](05-outreach.md) | Workflows, walker, broadcasts, segments, campaigns | C | P2 |
| [06-conversations-inbox.md](06-conversations-inbox.md) | Threads, projector, takeover, the team inbox | TBD | P3 |

Companion material: `../diagrams/` (same numbering) · `../app-crm-architecture.md`
(package tree, grants, runtime) · `../task-map.md` (who builds what, in what order) ·
`../../decisions/` (ADR 0001–0017 — the why behind every rule here).

A guide is living: when an ADR changes a rule, the guide changes in the same PR.
