# ddchart — CLAUDE.md

## Project
Brand positioning scatter chart — D3 7.8.5, Tableau extension, export-ready SVG.
Single file: `index.html`. No build step. Deploy = `git push` to GitHub Pages.

## Live URL
https://bengtcj.github.io/ddchart/index.html

## Test in browser
Open the URL directly — falls back to sample data when no Tableau host is
present. Use the ⚙ settings dialog to switch theme / mode / guides.

## What this chart draws
Brands on a fixed 0–5 × 0–5 plane (LLM × Survey). Six zone bands (Drift,
Inflection, Overreach, Plateau, Peak, Frontier) drawn as subtle alternating
grey tints with grey zone labels. A piecewise parabolic frontier curve through
the apex. Variable-size bubbles from score_metrics_overall. The client brand
is the only element in colour — `#e994a2` (pink) fill and dashed ring.
Everything else is black, white, or grey. Clicking a brand dims the rest
and selects marks on other Tableau worksheets.

## Palette rule
The entire chart uses ONLY: black, white, shades of grey, and `#e994a2`.
No other colours. This applies to both themes. The `brandColorMap` / tab10
system from scorer_viz is deliberately NOT used here — all non-client brands
are the same grey. The client brand is the only pink element.

## Themes
| Token | Light | Dark |
|---|---|---|
| ground | white | #111111 |
| ink | black | #e6e6e6 |
| brand (non-client) | #7d7d7d | #888888 |
| client | #e994a2 | #e994a2 |
| zone tint | black @ 0.04–0.08 | white @ 0.04–0.08 |

All guide lines, labels, axes, and legends derive from grey tones that
flip with the theme. The pink is the same in both.

## Modes
| Mode | Draws |
|---|---|
| **simple** | Brand centroids, curve, apex, zone tints + labels, guides, bubble legend |
| **advanced** | + G/S/F category cloud dots + letter annotations + triangle lines per brand (grey for non-client, pink for client) |

## Guides
| Setting | Draws |
|---|---|
| **zones** (default) | Three zone boundaries (Q25x, Q75x, meanY) — solid, no annotations |
| **all** | Six stats with text annotations (analyst view) |
| **none** | Nothing |

## Sync rule — the two renderers
The scatter chart exists in two independent renderers:

- **scorer_viz.py** — matplotlib, the Colab/notebook design surface
- **ddchart/index.html** — D3, Tableau extension + export

Same data contract (SCORER_RANKING columns). Same visual target. When changing
the chart in either, update the other to match. Close out by running the
fixture data against both and flagging any visual divergence for user
inspection.

The shared data shape (one row per brand):
  brand_name, display_name, brand_role,
  score_x_overall (→ x), score_y_overall (→ y),
  zone_name, rank_overall, score_metrics_overall (→ bubble size),
  target_x, target_y (→ apex),
  apex_dist, score_rank

scorer_viz.py is the source of truth for chart geometry decisions: zone cuts,
bubble sizing formula, curve equation. ddchart ports those decisions to JS.
Changes to geometry start in scorer_viz.py and are ported here.

Note: scorer_viz uses tab10 brand colours; ddchart uses grey for all
non-client brands and pink for the client. This is a deliberate divergence —
the renderers match on GEOMETRY and POSITION, not on colour.

## Architecture
- Single file app — all JS/CSS/HTML in index.html
- Config stored in Tableau Extensions Settings API (persists in .twb)
- `renderScatter(el, data, clientBrand, opts)` is the chart entry point
- ResizeObserver triggers re-render on container resize
- `curveY()` mirrors `scorer_core.curve_y` (piecewise parabola)
- Zone bands from Q25/Q75 on x, mean on y (same as `_zone_bands`)
- `bubbleRadius()` mirrors `_bubble_size` (min-max normalised, r=5..16)

## Data source (Tableau)
Reads from a worksheet via `getSummaryDataAsync`, same pattern as
b_tableau_extensions_2. The worksheet must carry SCORER_RANKING columns
(one row per brand per run, filtered to a single run). The `bulletproof_target`
row supplies the apex coordinates and is excluded from the chart.

## Settings (persisted in .twb via Extensions Settings API)
worksheet, brandField, xField, yField, zoneField, rankField,
apexXField, apexYField, roleField, scoreField, displayField,
parameter, theme, mode, guides

All have sensible defaults matching the SCORER_RANKING column names.

## Code style rules (from b_tableau_extensions_2)
- No `?.` optional chaining
- No `??` nullish coalescing
- No arrow functions in D3 event handlers that need `this`
- All chart sizing uses `el.getBoundingClientRect()` (not vw/vh)
- Inline styles on chart elements (D3 `.style()` pattern)

## Deploy
Before every commit, bump `VERSION` at the top of index.html.
Format: `'YYYY-MM-DD.N'` where N increments from 1 each day.

```
git add -A && git commit -m "..." && git push
```

## Export path (future)
`renderScatter` takes a plain data object and a DOM element — no Tableau
dependency at render time. For export:
1. Create an off-screen container, call renderScatter with the data
2. Serialise the SVG via XMLSerializer
3. Convert SVG → PNG (cairosvg or headless Chrome screenshot)
4. Place on a PPTX slide via python-pptx (scorer_deck.py)
5. Or render a full HTML page to PDF via headless Chrome --print-to-pdf

## Known issues / future work
- Apex marker: star to be replaced with a small image (user will provide later — "the apex image")
- Category cloud data not yet populated from Tableau — needs components layer or second worksheet
- Tracking overlay (baseline ghost bubbles) not yet ported
- Gap arrows to target brand not yet ported
- Bubble size legend placement needs polish at small container sizes
- display_name not yet wired — falls back to brand_name until the ACTIONS.md display_name chain is closed
