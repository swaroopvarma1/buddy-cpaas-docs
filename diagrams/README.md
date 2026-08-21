# Diagrams — Buddy CPaaS

Built in the extracted visual language (`../visual-language/`). Self-contained HTML —
open in any browser. Phase badges P1/P2/P3 per ADR 0016. One file per subject; revise
the SAME file, never mint siblings.

| # | File | Shows |
|---|---|---|
| 00 | `00-master-system.html` | The entire machine: 7 lanes — touchpoints → edges → spine → bridges → contracts/engines → stores → views; sync door; receipts loop |
| 01 | `01-event-spine.html` | Record module: adapters, event_raw, consumers, contracts, facts-vs-commands |
| 02 | `02-identity.html` | Identity: callers → resolve()/assert_facts()/merge → crm.customer → readers |
| 03 | `03-permission.html` | Permission: sources → record_consent → stores → the fail-closed gate → send token |
| 04 | `04-connectivity.html` | Connectivity: onboarding → doors/pipes/templates → token→manifest→dispatcher→send → provider & receipts |
| 05 | `05-outreach.html` | Outreach: admission doors → enrollment/walker → the plan document → gate-checked sends & funnel |
| 06 | `06-conversations-inbox.html` | P3: inbound → projector → thread stores → the agent → the team (takeover/reply/handback) |
