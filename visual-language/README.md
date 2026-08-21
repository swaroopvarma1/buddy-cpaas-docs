# Buddy CPaaS visual language

Extracted 2026-08-19 from the four reference images the team chose (Semantix-style
platform diagram · dark gradient roadmap poster · node-card lineage board). Every
visual we produce — system designs, data flows, roadmaps — uses this language.
Not fancy; developer-friendly; consistent.

## Files

| File | Purpose |
|---|---|
| `tokens.css` | Canonical design tokens (colors, type scale, shape). Update here first. |
| `template.html` | The reusable page: component gallery + copy-paste scaffold. Open in a browser. |
| `../diagrams/*.html` | Actual diagrams. Each is SELF-CONTAINED (tokens inlined) so a single file can be shared/committed/rendered anywhere. |

## Two modes, fixed purposes

1. **System flow (light)** — refs 1 & 4. Default for every developer diagram:
   pale blue-gray canvas, swimlane panels one step darker, white cards with soft
   shadows, module tag pills, thin navy elbow connectors with pill labels.
2. **Roadmap (dark)** — refs 2 & 3. ONLY for phase timelines/posters: near-black
   canvas, thick snake path stepping purple → violet → green, circular icon
   badges, uppercase white step titles with small gray body text.

## The module color law (never re-assign)

| Color | Meaning |
|---|---|
| purple | sources · edges · adapters |
| blue | the spine · record · core platform |
| green | outputs · sends · success |
| teal | storage · tables |
| amber | external providers (Meta, Shopify, telephony) |
| red | refusals · fail-closed · errors |

A card's **tag pill** carries its module color; card bodies stay white — color is
information, not decoration (the references never flood-fill nodes).

## Composition rules

- **Swimlanes are stages**: translate → record → bridge → write → store/exit.
  Left-to-right reading order; one concern per lane.
- **Cards**: title (13px/700) + at most 3 short body lines (11px). If a card
  needs more, it wants to be two cards or a footnote.
- **Connectors**: thin (1.5px) navy elbow lines; label only when the edge isn't
  self-evident, as a navy pill (white 9.5px text). Dashed = command/sync-door or
  deferred path. Solid = fact/data flow.
- **Legend strip** top-right of every diagram; footer carries doc refs (ADR
  numbers, design doc names) so a diagram is always traceable to prose.
- Titles sentence-case; lane titles UPPERCASE 11px; ASCII only.

## QA loop

Open the HTML file in the chrome-devtools MCP browser (`file://` URL),
screenshot, fix overlaps/spacing, repeat. A diagram ships only after it has
been LOOKED at.
