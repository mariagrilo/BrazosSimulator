# Brazos Simulator — Analyst Tool Redesign

A standalone HTML prototype of a redesigned flight-data analysis tool for **Brazos safety analysts**. Built to validate UX direction with stakeholders before any production engineering work.

The original tool we're redesigning lives at `https://truthdata.flightdataservices.com/flight/46309524/graph/` (private, login-walled). Our prototype is a single self-contained `skeleton.html` — vanilla HTML/CSS/JS, no build step, no framework. Open the file, see the tool.

---

## Quick start

```bash
# from this directory
python3 -m http.server 8765
# then open http://localhost:8765/skeleton.html
```

Shareable preview link (the `mariagrilo` repo is public, `subvisual` is private):

**https://htmlpreview.github.io/?https://github.com/mariagrilo/BrazosSimulator/blob/main/skeleton.html**

A single `git push origin main` writes to both remotes — see [Git workflow](#git-workflow).

---

## Who this is for

**Brazos flight safety analysts.** Their workflow drove every architectural decision. If a screen, control, or animation doesn't help an analyst do their job, it shouldn't be there.

**The workflow we built around** (validate with analysts before shipping):

1. Open a flight already knowing it has events (≤ ~6 events per flight, never triple digits).
2. Immediately see *where* those events fall on the timeline.
3. Scrub to an event timestamp to understand what justified the maneuver.
4. Use the 3D visualizer (sometimes Map too) for a video call with the pilot — spatial common ground.
5. Validate / comment / change status of each event without leaving the simulator.

This is the assumption set the whole prototype rests on.

---

## Design system

Source: Figma file `ol9sqOkx1bq7dKwqvxXtQT` (`Brazos-UI-WIP`). Chart palette specifically: node `4161:15602`.

CSS custom properties on `:root` (all verified against the current file):

```css
--color-bg:          #021C25;   /* page background (Dark Blue 100) */
--color-surface:     #031F2C;   /* panel background */
--color-surface-hi:  #052535;   /* raised surfaces, cards */
--color-border:      #074058;   /* dividers / borders (Blue 100) */
--color-accent:      #28C8B9;   /* primary interactive (Light Blue 100) */
--color-accent-dim:  #1a8a80;
--color-level-3:     #FF6019;   /* L3 events (Signal Orange) */
--color-level-2:     #F9F46E;   /* L2 events (Signal Yellow) */
--color-level-1:     #6DEB5D;   /* L1 events (Signal Green) */
--color-text:        #FFFFFF;
--color-text-dim:    #7FA8B8;
--color-text-muted:  #3A6478;
--font-body:         'Acid Grotesk', 'Inter', sans-serif;  /* body weight 350 (Book) */
--font-label:        'Space Grotesk', sans-serif;
/* Acid Grotesk (FFF / Studio Feixen) — local @font-face from assets/fonts/.
   Weights wired: 300 Light, 350 Book, 400 Normal, 450 Regular, 500 Medium, 700 Bold (+ italics). */
--topbar-height:     56px;
--timeline-height:   70px;   /* track 48px + 11px padding each side */
--sidebar-width:     260px;
--sidebar-collapsed: 52px;
```

**Severity colors are sacred.** Orange / yellow / green appear *only* on event markers, the severity pill, and the count badges in the Status hero. They never appear on a chart line, never on a status pill that's not actually severity-related, never on a non-event UI affordance. This avoids the most common misread in safety tooling.

**Chart palette** is a separate 15-color set defined in `PARAM_COLORS`, pulled from the Figma legend node. Every parameter has a *stable* color across every chart it appears in, so analysts learn the mapping (Airspeed is always red, Alt AAL is always pink). Engine 1/2 pairs share adjacent hues:

- Eng 1 / Eng 2 Torque → violet `#9B65FF` / lavender `#E6C2FF`
- Eng 1 / Eng 2 N1 → spring green `#00FF80` / mint `#00FFAE`
- Gas Temp 1 / Gas Temp 2 → coral `#FF6D56` / peach `#FFAD9D`

---

## Architecture

Everything is in `skeleton.html`. Three sections:

| Section | Approx. lines | Contains |
|---|---|---|
| `<style>` | ~9 – 2398 | All design tokens, layout, components, modules, popovers, drawers |
| `<body>` | ~2400 – 2806 | Topbar, timeline, flight-info strip, workspace (sidebar + mosaic), event-popover/drawer mount points |
| `<script>` | ~2815 – 4833 | Data, render functions, interactions, layout engine |

(Line ranges drift; treat them as orientation, not pins.)

### Mosaic — nested split-pane tree

The workspace is **not** a CSS grid. It's a recursive tree of horizontal/vertical splits. Each divider belongs to *one* split node and resizes only its two adjacent children. Hiding a module makes its parent split collapse cleanly so siblings expand to fill.

```js
type Node =
  | { type: 'leaf', module: string }
  | { type: 'split', dir: 'h'|'v', weights: number[], children: Node[] }
```

State lives in `LAYOUT` (the live tree), with `BUILTIN_TEMPLATES` for presets and `SAVED_LAYOUTS` for user templates.

Key functions:

- `applyMosaic()` — wipes the mosaic DOM, moves all module sections into a hidden pool, rebuilds nested flex containers from the tree, reinserts modules at their leaves.
- `renderTreeNode()` — recursive flex builder; creates `.mosaic-split`, `.mosaic-pane`, `.mosaic-divider` elements.
- `removeModuleFromTree()` / `addModuleToTree()` / `swapModulesInTree()` — tree mutations driven by the × button, the Layout visibility checklist, and drag-to-swap respectively.
- `collapseSingletonSplits()` — after removal, any split left with a single child is replaced by that child, so the sibling expands.
- `setupMosaicResize()` — delegated `mousedown` on `.mosaic-divider`; mutates the divider's parent split's `weights[idx]` and `weights[idx+1]`.
- `setupMosaicReorder()` — delegated `mousedown` on `[data-module-drag]`; drop intent is decided by *where* on the target the mouse releases (Path B): hovering an edge of a pane = split that pane on that edge; hovering the gap between sibling panes = insert there; hovering near the workspace perimeter = wrap the whole tree. A translucent **phantom pane** previews the landing slot live while dragging. There is no swap mode — every gesture is "insert beside".
- `applyFitContentSizing()` — after each `applyMosaic`, walks panes and pins the PFD (400px) and Engines (240px) to fixed pixel heights when they sit inside a vertical split, so they always render at their intrinsic content height. Horizontal splits keep weighted share.
- `pushLayoutUndo()` / cmd-Z — 20-deep layout undo stack restores `LAYOUT` after any mutation.

This replaced a CSS-Grid layout. Grid tracks are shared across all cells, so adjusting one cell affected every other cell in the same row/column. The tree fixes that.

### Modules

Seven static `<section class="module" data-module="…">` elements live in the HTML and get moved around the tree by `applyMosaic`:

| `data-module` | Title | Body content |
|---|---|---|
| `graphs` | Graphs | Dynamic chart cards + "Add a graph" button (bolder teal-tinted CTA, lifts on hover) |
| `tabular` | Tabular | Tabular + Status as sub-tabs. Tabular: downsampled parameter table; click-to-seek; auto-scrolls to current row; event rows are colour-tinted across **all** columns by their severity level |
| `status` | Status (tab inside Tabular slot) | Compact stat-block hero (L3/L2/L1 counts) + single scrollable list of events in time order |
| `view3d` | 3D View | Helicopter photo placeholder (`assets/helicopter-placeholder.png`) + live readout (ALT, IAS, VSI, HDG, P, R) |
| `map` | Map | Real map photo (`assets/map.png`) with zoom + pan, flight-track polyline overlay, directional aircraft arrow rotated to path tangent |
| `pfd` | Primary Flight Display | Speed/alt tapes, attitude, VSI, heading strip — fits to 400px in vertical splits |
| `engines` | Engines | TRQ / N1 / EGT bars per engine + COL + digital cells — fits to 240px in vertical splits |

Each module has a header with: title, ⋮⋮ drag handle, × hide button. **No sub-tabs** — PFD/Engines used to be tabs of one Instruments module, and 3D/Map used to be tabs of one View module. Both were split per analyst-side feedback that tabs hide functionality.

### Event interaction model

Three commitment levels, increasing in weight:

1. **Pin click on the timeline** (or graph event-line click, or Finder event click) → `openEventPopover()` shows a small 340px popover anchored to the pin: code, value, threshold, phase, fixed-width status + validity pills (one row, never wraps), "Open full detail" link. Closes on ×, outside-click, Esc, or another pin click. The simulator stays visible.
2. **"Open full detail"** or **Status row click** → `openEventDrawer()` shows the heavy side drawer: large severity pill + level under title, line-broken event ID / name, flight info strip mirrored at the top, editable description, full metadata (with a **Specifications** chip that hovers a threshold tooltip anchored to the chip's centre), comments thread, status + validity dropdowns. No "Jump to moment" — the underlying timeline is already at the event.
3. **Status + validity changes** route through `openInlinePicker()` (a dropdown anchored to whichever pill was clicked). All state changes are **session-local** — they live on `FLIGHT.events[*]` and reset on reload. No backend wired.

`STATUS_LIST` = `["open", "in_progress", "closed"]`. `VALIDITY_LIST` = `["auto", "manual"]`.

### Timeline

- Altitude profile drawn as an SVG path *behind* the 48px track — gives flight shape at a glance (helicopter vs jet, where each phase sits).
- Event markers are map-pin glyphs (per Figma 5221:13079) bottom-anchored inside the track, tip on the exact event time, inner severity-coloured dot (r=3).
- Current-time chip sits below the track, with a teal caret pointing up at the playhead.
- Click / drag the track to seek. Buttons: prev event, play/pause, next event. Speed selector from 0.25× to 8×.
- **Timeline zoom is not yet built** — it's the next thing to ship (see [Roadmap](#status--roadmap)).

### Flight info strip

Click the flight name at the left end of the timeline bar — a slim strip slides down between the timeline and the workspace with: Flight ID, Aircraft, Operator, Flight type, Origin, Destination, Takeoff, Landing, Duration. Replaces a planned "Flight Info module" — flight metadata is read once at session open; doesn't deserve permanent module real estate.

### Sidebar

- **Finder** fills the column. Search input + scope toggles (Event / KTI / KPV / Phase / Parameters — Parameters last per latest order). Clicking a scope toggle scrolls the results to that section.
- **Layout** popover (footer, left icon) — visibility checklist for every module.
- **Templates** popover (footer, right icon) — built-in tree presets ("Default analyst layout", "Approach review", "Engine investigation") + "+ Save current as template".

Sidebar collapses via the toggle. When collapsed, the Layout / Templates popovers anchor outside the sidebar (`position: fixed`) so they don't get clipped.

### Chart system

- **Dual Y-axes, both on the left.** A chart accepts up to 2 distinct scales: `ft`, `kt`, `%`, `deg`, `fpm`, `°C`, `hdg`. The `+` button on a chart header opens a picker that disables params whose scale would push the chart over 2 scales.
- **Hover-isolate** — hovering any legend item dims every other line and the axis that doesn't apply to the focused param.
- **Per-legend `×`** removes a single param from the chart.
- **Picker is a toggle list** — clicking an already-added param removes it; the menu stays open across toggles for batch editing.
- **Empty chart card** shows an "Add parameters" CTA pill instead of an empty plot.
- "Add a graph" button at the bottom of the graphs module creates a new empty card.

---

## Mock data

`FLIGHT` is one helicopter flight modeled on flight #46309524:

- AW139 G-BJOP, Bristow Helicopters
- 39 min 23 s
- Aberdeen (EGPD) → ETAP-1 Platform
- 9 phases (Engine Start through Taxi In)
- 6 events spanning all three severity levels
- 1 approach

`PARAMETERS` holds 15 time-series (altitudes, airspeed, engine torques, N1s, gas temps, attitude, VSI, heading, collective). Each has `name`, `units`, `group`, `decimals`, `scale`, and a `series` array generated by `gen()` — piecewise interpolation through phase keyframes with light noise plus Gaussian spikes at event times.

`PARAM_COLORS` holds the stable hex per parameter.

Comments / validity / status changes live on `FLIGHT.events[*]` in memory and survive a session, but reset on reload. There is no backend.

---

## Status & roadmap

### Done

- Brazos design tokens + visual identity
- **Acid Grotesk wired locally** (`assets/fonts/`), body weight 350 Book per Figma
- Timeline with altitude profile, Figma map-pin event markers, scrubber, playback
- Mosaic nested-split-pane layout: drag with **phantom-pane preview** (edge / gap / perimeter drop), resize, hide, layout undo (cmd-Z), built-in + user templates
- **Fit-content sizing** for PFD (400px) and Engines (240px) in vertical splits
- 7 modules: graphs, tabular (Tabular + Status as sub-tabs), view3d, map, pfd, engines
- **Tabular interactivity**: click-to-seek, auto-scroll-to-current, severity-tinted event rows, in-row event markers
- Event interaction: pin popover (340px, one-row pills) + drawer (heavy, with flight strip + Specifications tooltip) + inline picker
- Comments thread per event (session-local)
- Finder with scope toggles (Parameters last), scroll-to-section, stable per-param colors
- Per-graph add/remove parameters with dual-Y scale enforcement
- Hover-isolate on chart lines + matching axis
- Bolder "Add a graph" CTA with teal hover lift
- **Status module compacted**: stat-block hero (no verdict text), single scroll, 3-col compact event list
- 3D View with helicopter photo placeholder; Map with real photo + zoom/pan + directional aircraft arrow
- Flight info slide-down strip — one line at any width via flexbox
- Sidebar Layout & Templates popovers
- Working dual-remote git push, assets tracked in `assets/`

### Pending — Thierry's latest feedback

- **Timeline zoom.** Drag-select a range on the timeline to zoom; click a phase or KTI to set a span; persistent context strip (mini-map of full flight) above the zoomed range; zoom affects all charts. This is the next thing to build.

### Pending — broader

- Real 3D visualizer (currently a photo placeholder, pending WebGL integration).
- Real map provider (Mapbox / Google / OSM not chosen yet — currently a photo with track overlay).
- Real flight data — everything is mocked.
- Backend wiring — comments / status / validity changes are session-local.
- Multi-window sync model — TBD.
- Persistent saved layouts (`SAVED_LAYOUTS` is in-memory only).

### Open design questions for the analyst meeting

- Is "timeline as primary nav surface" the right anchor?
- Which instruments matter at first glance? How much does priority vary between analysts?
- Single screen with scrolling vs. multi-monitor as the default working surface?
- Anything expected that isn't there yet?

---

## Load-bearing design decisions

These have been argued out — don't relitigate without evidence.

| Decision | Why |
|---|---|
| Severity colors only on event markers | A yellow chart line gets misread as a moderate-severity indicator. Reserve those hues. |
| Stable per-parameter color across all charts | Builds analyst muscle memory. Engine pairs use adjacent hues so they read as a pair. |
| Dual-Y both on the left, max 2 scales | Left+right dual-axis charts get misread constantly. Two left axes + hover-isolate is clearer. |
| Pin click = popover, not drawer | The pin is 16px; a 520px overlay was disproportionate and covered the chart the analyst came to look at. Drawer is reserved for "deep" reveals. |
| Mosaic via nested split tree, not CSS Grid | Grid tracks are shared across cells — adjusting one cell affected others. Splits keep every divider local to two children. |
| All modules visible by default (no module tabs) | Analyst feedback (Thierry): "everything visible, not hidden in tabs". |
| Sub-tabs eliminated (PFD/Engines, 3D/Map split) | Same reason — tabs hide functionality. |
| Flight info as slide-down, not a module | Flight metadata is read-once. A permanent module would compete with active work surfaces. |
| Templates popover, not a full template manager modal | Original tool had a heavy template manager. Keep it light. |
| Add-param menu stays open across toggles | Analysts toggle multiple params at once; closing after each click was friction. |
| Status hero kept, flight ID line removed | Status is for judgment ("severe events present"). Identification belongs in Flight Info. |

---

## Git workflow

```bash
$ git remote -v
origin  https://github.com/subvisual/brazos-simulator.git (fetch)
origin  https://github.com/subvisual/brazos-simulator.git (push)
origin  https://github.com/mariagrilo/BrazosSimulator.git (push)
```

`origin` has **two push URLs** — a single `git push origin main` writes to both. Fetch is from `subvisual` only.

- **subvisual/brazos-simulator** — private, team viewing.
- **mariagrilo/BrazosSimulator** — public, so `htmlpreview.github.io` can render the prototype.

---

## Notes for the next session

Terse, on purpose:

- **Don't redesign the layout system again.** The nested-split-tree refactor was substantial. It works. Build on it.
- **Severity colors are sacred.** Never on a chart line. Never on a non-status pill. Never on a non-event affordance.
- **Single file by design.** No build step. A colleague can open `skeleton.html` locally with zero setup. Don't break that without a strong reason.
- **Drag / resize / hide use document-level delegated handlers** because modules move around the tree on every render. Per-element handlers would be stale immediately. Keep delegation.
- **Empty cells were a bug** in the old grid layout. The tree collapses single-child splits so siblings expand. Don't bring back static placement.
- **State mutations**: anything touching `LAYOUT` (the tree), `FLIGHT.events[*]`, or `GRAPHS` needs to be followed by `applyMosaic()` or the corresponding render function. The render functions write directly to DOM; there's no virtual DOM.
- **Dead code worth knowing about**: `_legacy_findResizeHandles`, `_legacy_positionMosaicResizeHandles`, `_legacy_setupRightTabs`, the empty stubs `setupRightTabs`, `setupInstrumentsTabs`, `setupLayoutOptions`, `setupPanelResize`, and the now-noop `renderMap` (its original body lives next to it as `__removed_renderMap_unused`). Safe to delete in a future pass — left in for now to keep the boot section unchanged.
- **Assets live in `assets/` (lowercase).** macOS is case-insensitive but GitHub isn't — keep the directory lowercase or htmlpreview will 404. Fonts in `assets/fonts/` (FFF Acid Grotesk OTFs + Space Grotesk variable TTF). Photos: `helicopter-placeholder.png`, `map.png`.

When in doubt about a design direction: the user is **mariagrilo** (Subvisual, designer). She knows the analyst context, has talked to Thierry (Brazos), and will push back on bad design choices. Listen to her critiques — she's run `/impeccable:critique` on previous AI proposals and the feedback was correct.
