# Decisions — Buddy CPaaS build

One file per finalized decision (ADR style): **Context → Decision → Consequences**.
Numbered, never renumbered, never deleted — a reversed decision gets a new ADR that
supersedes the old one. The build docs (`../canon/`) say *what* the system
is; these records say *why we chose*, wherever we filled a gap or deviated.

| # | Decision | Status |
|---|---|---|
| [0001](0001-crm-lands-in-clairvoyance-as-top-level-module.md) | CPaaS lands in clairvoyance as top-level `app/crm` | Accepted |
| [0002](0002-phase-0-base-fixes-precede-crm-work.md) | Phase 0 base fixes precede all CRM work (incl. PgBouncer introduction) | Accepted |
| [0003](0003-first-cut-whatsapp-outbound-only.md) | First cut = WhatsApp **outbound only**; inbound mechanics designed-in, activated in phase 2 | Accepted |
| [0004](0004-message-dispatch-simple-queue.md) | Message dispatch = simple DB queue, not the five-plane dispatcher | Accepted |
| [0005](0005-pilot-shopify-breeze-merchants.md) | Pilot = Shopify/Breeze merchants; abandoned-checkout + order-confirmation flows | Accepted |
| [0006](0006-runtime-same-image-role-flags.md) | Runtime: one image, POD_ROLE-style flags; crm workers are separate pods | Accepted |
| [0007](0007-phase-1-api-surface-ops-admin-only.md) | Phase 1 API surface: ops/admin `/crm/*` only; merchant console is a post-M2 fast-follow | Accepted |
| [0008](0008-consent-seeding-shopify-import-plus-utility.md) | Consent seeding: Shopify IMPORT + utility purpose for order-confirmation; importer pluggable | Accepted |
| [0009](0009-shopify-ingress-hybrid-relay.md) | Shopify ingress: anchor relays now, direct subscriptions later | Accepted |
| [0010](0010-voice-outside-the-gate-in-phase-1.md) | Voice stays outside the gate in phase 1; lead-machine checks govern calls | Accepted |
| [0011](0011-waba-template-apis-in-phase-1.md) | WABA template management APIs built in phase 1 (Tech Provider APIs) | Accepted |
| [0012](0012-inbox-whatsapp-first-ig-fast-follow.md) | Team inbox ships with phase-2 WhatsApp inbound; Instagram fast-follows | Accepted |
| [0013](0013-inbox-ai-first-takeover-is-assignment.md) | Inbox: AI answers first; takeover IS assignment; shared escalation queue | Accepted |
| [0014](0014-conversation-tables-not-chat-session-reuse.md) | Inbox storage: crm.conversation + conversation_message; chat tables stay the bot cockpit | Accepted (amended after adversarial review) |
| [0015](0015-message-direction-and-storage-map.md) | Direction map: crm.message outbound-only; thread always lives in crm tables; chat_session only while the AI drives | Accepted |
| [0016](0016-product-led-rephasing.md) | Product-led phases: customers e2e + UI first · static flows second · AI + inbox third; memory moves to phase 3 | Accepted |
| [0017](0017-voice-plugs-into-spine-phase-1.md) | Voice plugs into the spine in phase 1: call-lifecycle events + resolve() stamping + journey voice arm (takeover still deferred) | Accepted |
| [0018](0018-gate-mechanics.md) | Gate mechanics: dispatch-time verdicts · timezone ladder (+91→IST) · 9–21 marketing / txn exempt · 1/day·4/wk caps · defer-once | Accepted |
| [0019](0019-console-extends-loom.md) | Phase-1 console extends loom: Customers top-level + templates under Channels · per-merchant flag · loom theme contract · U1 collapses to wiring | Accepted |
