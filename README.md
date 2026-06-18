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
| `tabular` | Tabular | Downsampled parameter table; click-to-seek; auto-scrolls to current row; event rows are colour-tinted across **all** columns by their severity level |
| `status` | Status | Own module (split out from the Tabular tab strip). Compact stat-block hero (L3/L2/L1 counts) + single scrollable list of events in time order |
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
- **Timeline zoom** (shipped):
  - Drag-select on the track → zoom into the selected interval; a teal selection rectangle previews the range. Dragging inside an already-zoomed view zooms further.
  - Click on the track → seek the playhead to that position.
  - Grab the playhead bar (or within ~8px of it) → scrub.
  - Click a phase / KTI / KPV / Event in the Finder → animated zoom-to-range with cross-fade on charts. Phases: a second click on the same phase zooms back out. Events/KTIs/KPVs that fall outside the current zoom trigger an animated zoom-out before seeking.
  - Reset-zoom button (`⤾`) sits inside the track, right-anchored; appears only while zoomed. Solid accent fill so it reads as a primary action.
  - Animation: `animateViewTo()` tweens `STATE.viewStart` / `viewEnd` over ~360ms with ease-in-out cubic. Each frame re-runs `applyZoomTimeline()` (continuous reflow of altitude profile / event pins / playhead / time readouts); charts cross-fade via `.graph-card__plot.is-zooming { opacity: 0.18 }` and `applyZoomCharts()` snaps to the final viewBox at the end. Polylines use `vector-effect="non-scaling-stroke"` so they don't visually thicken under the X-axis squeeze.

### Flight info strip

Click the flight name at the left end of the timeline bar — a slim strip slides down between the timeline and the workspace with: Flight ID, Aircraft, Operator, Flight type, Origin, Destination, Takeoff, Landing, Duration. Replaces a planned "Flight Info module" — flight metadata is read once at session open; doesn't deserve permanent module real estate.

### Sidebar

- **Finder** fills the column. Search input + scope toggles (Event / KTI / KPV / Phase / Parameters — Parameters last per latest order). Clicking a scope toggle scrolls the results to that section.
- **Layout** popover (footer, left icon) — visibility checklist for every module.
- **Templates** popover (footer, right icon) — built-in tree presets ("Default analyst layout", "Approach review", "Engine investigation") + "+ Save current as template".

Sidebar collapses via the toggle. When collapsed, the Layout / Templates popovers anchor outside the sidebar (`position: fixed`) so they don't get clipped.

### Secondary popout window

The Layout popover's "Pop out second window" button opens the same `skeleton.html` as a second window via `window.open`. The popout strips chrome and reuses the existing modules, syncing state across windows over `BroadcastChannel('brazos-sim')`.

**URL flag detection** — supports both forms:

- `?window=secondary` — clean form for direct serving (local dev, future hosted deploy).
- `#window=secondary` — hash form, required when running via `htmlpreview.github.io` because that proxy uses the query string for the source GitHub URL. The pop-out button always opens with the hash form so it works in both environments.

**Body classes**:

- `body.is-secondary` — applied at boot when the flag is present; CSS hides `.topbar`, `.flight-info-strip`, and `.sidebar`. The timeline-bar stays visible by default.
- `body.is-secondary-no-timeline` — toggled by the `×` on the secondary's timeline and persisted in `sessionStorage('brazos:sec-no-timeline')`. When set, hides the timeline-bar and shows the floating revealer strip at top center: `[+ Add module] [Timeline ▼]`.
- `body.has-sibling-window` — applied when the BroadcastChannel hello/hello-ack handshake completes. CSS-gates the per-module `→ send to other window` button in each header.

**BroadcastChannel protocol** — `var BC = new BroadcastChannel('brazos-sim')` (declared with `var` so it's hoisted; `seekTo()` and `setSelectedFinder()` can call `broadcastState()` at any point during boot without TDZ trouble).

| Message | Direction | Payload | Purpose |
|---|---|---|---|
| `hello` | either → either | `{fromRole}` | sent on boot; awaiting an ack |
| `hello-ack` | either → either | `{fromRole}` | confirms sibling presence; primary follows with `state` + `layout-modules` |
| `goodbye` | either → either | `{fromRole}` | sent in `beforeunload` |
| `reload` | primary → secondary | `{fromRole}` | sent in primary's `beforeunload` so a single Cmd-R on the primary refreshes both windows |
| `state` | either → either | `{pct, viewStart, viewEnd, playing, speed, selectedFinderKey}` | transition broadcast — sent only on play/pause toggle, speed change, seek, zoom apply, finder selection. Receiver applies under a `_suppressBroadcast = true` guard to prevent echo loops |
| `tick` | primary → secondary | `{pct}` | steady-state playback updates only, sent from the rAF loop. Receiver ignores when `STATE.playing` is locally false — this is what keeps a pause click from being undone by a queued tick. **Don't add `playing` to this payload.** |
| `layout-modules` | either → either | `{modules: string[]}` | broadcast at the end of every `applyMosaic()`. Receiver writes to `OTHER_MODULES` and re-renders the picker rows |
| `transfer` | either → either | `{module, toRole}` | sent by `sendModuleToOtherWindow()` after pruning its local LAYOUT; receiver appends via `receiveTransferredModule()` |
| `request-back` | either → either | `{module}` | sent by the secondary's add picker (and the primary Layout popover's "in window 2" row). Receiver calls `sendModuleToOtherWindow()` if it actually has the module |

Primary owns the rAF loop. Secondary's `setupPlayback()` skips the advance step (`WINDOW_ROLE === 'primary'` guard), so the two windows never double-tick `STATE.pct`.

**Per-module → button** — injected into every `.module__head-actions` at boot, hidden via CSS until `body.has-sibling-window` is set. Click → `sendModuleToOtherWindow()` removes the module from this window's LAYOUT (collapsing the tree via `pruneModuleFromTree()`, my multi-window-only helper that returns a *new* tree) and tells the other side to append it.

**Layout popover** — 3-state per row. `renderModuleToggles()` writes the same HTML into every container that has it (`#module-toggles` and the secondary's `#sec-add-toggles`):

| Visual | Meaning | Click action |
|---|---|---|
| Solid accent check + `→` button | `here` (in our LAYOUT) | `→` → send to other; row → toggle to hidden (primary popover only) |
| Dashed accent border + "in window 2" tag | `other` (in OTHER_MODULES) | row → broadcast `request-back` |
| Empty check | `hidden` (in neither) | row → add (see semantics below) |

State is **recomputed at click time** from `getVisibleModules()` and `OTHER_MODULES` — the row's `data-state` attribute can lag if `OTHER_MODULES` updates between renders. Without this recompute, a fast click landed on the stale "hidden" branch and ran the wrong path.

**Add-module path on the secondary is its own thing** (not `toggleModuleVisibility`):

- Primary's Layout popover for `hidden` → `toggleModuleVisibility` → `addModuleToTree`. That uses `LAST_POSITION` to restore a module to its analyst-layout home — correct in the primary, makes no sense in the stripped secondary.
- Secondary's `+ Add module` picker → `appendModuleToLayout(name)` (a thin extraction from `receiveTransferredModule`). Flat append, no LAST_POSITION lookup. It dedups via `getVisibleModules().includes(name)` so a `request-back` arriving back as a `transfer` after the local append is silently skipped.

**Trigger surface on the secondary** — three buttons share one `#sec-add-popover`:

1. `.timeline-bar__sec-add` (in-bar `+ Add module`, accent fill) — visible while the timeline is shown.
2. `.secondary-floating-pill--primary` (top-center pill, accent fill) — visible while the timeline is hidden, alongside the `Timeline ▼` revealer.
3. `.mosaic-empty-cta` (dashed-border tile centered in the mosaic) — visible only while `#mosaic.mosaic--empty`, gated by `body.is-secondary:has(#mosaic.mosaic--empty)`.

`openAddPop(trigger)` anchors the popover under the trigger using the bar's bottom (when the trigger lives inside the timeline-bar) so the bar doesn't clip the popover. Horizontal: right-aligns for the corner triggers (in-bar / pill), centers under the CTA. If the popover would clip the viewport bottom, it flips above the trigger.

**Templates** — the legacy `prompt("Template name:")` is replaced by an in-app modal (`#template-modal`) styled to match the rest of the chrome. `openTemplateNamePrompt()` returns a Promise that resolves to the trimmed name or `null`.

### Chart system

- **Y-axes, all on the left.** A chart renders one axis column per distinct scale (`ft`, `kt`, `%`, `deg`, `fpm`, `°C`, `hdg`), no upper bound. The 2-scale cap was removed — every parameter can be added regardless of scale.
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

### Done — recent

- **Timeline zoom**: drag-to-zoom, click-to-seek, scrub the playhead, animated phase-scope from the Finder with chart cross-fade, reset-zoom chip.
- **Status split** out into its own module (no longer a Tabular sub-tab); default layout updated to three columns: Graphs + Engines / Tabular + 3D + Map / Status + PFD.
- **Secondary popout window** with BroadcastChannel state sync, per-module `→`, `+ Add module` picker with three trigger surfaces (in-bar, floating pill, empty-state CTA tile), and a Cmd-R reload chain.
- **Finder selection state**: dots desaturated unless selected; selected non-event items turn accent blue. Events keep their severity colors as diagnostic signal.
- **Contrast lift** (`--color-text-muted` `#3A6478 → #688E9F`) and `:focus-visible` accent rings on the module head buttons, per the `/impeccable:audit` pass.
- **Custom in-app modal** for naming layout templates (replaces `window.prompt`).

### 3D View & Map — feature wishlist (Brazos)

Feedback drop from Brazos, not yet prioritised. Most need real data providers wired before they can be visualised — captured here so the design intent isn't lost.

**3D viz layers** (grouped: Flight / Terrain / Weather / Routes)

- Flight: Events · Phases · KTIs · KPVs · Approach segments
- Terrain: Terrain displacement · Ground obstacles · Offshore platforms
- Weather: Clouds, rain, snow (see [TCU-Airport-API](https://github.com/BrazosSS/TCU-Airport-API)) · METAR · Relative wind vector
- Routes: Approach plates

**2D Map layers** (grouped: Flight / Weather / Traffic / Routes)

- Flight: Events · Phases · KTIs · KPVs · Approach segments
- Weather: Wx radar (on-demand) · SIGMETs ([NWS reference](https://www.weather.gov/zse/SIGMETs)) · METAR
- Traffic: ADS-B traffic
- Routes: ARINC424 · Approach plates

**A note on "Flight timeline display"**: Brazos's original ask was "Flight timeline display (with selected time points, phases, events, etc.)" inside the 3D viz. We tried this as a flat HTML strip at the bottom of the placeholder and removed it — the topbar timeline already provides the *time* axis, so a second flat copy was redundant. The intent is captured spatially instead, by the `Flight` group toggles (Events / Phases / KTIs / KPVs / Approach segments) which will render time-stamped markers along the helicopter's path in both surfaces when the real Map and 3D viz land.

**UI principles (mariagrilo, 2026-06-18)**:

- No controls overlaid on the map or 3D viewport. Toggles live on a dedicated surface, never floating over the imagery.
- For 3D View and Map, that surface is a **horizontal strip on top** of the module body (`.module__layers`), between `.module__head` and `.module__body`. A right-side vertical rail was tried and rejected — felt heavy at the module width these usually run at.
- Layers are **grouped by domain** (Terrain / Weather / Routes for 3D; Weather / Traffic / Routes for Map) with inline eyebrow labels (`.layer-group__title`) and a thin divider between groups. Never compounded into preset "views" — analysts toggle individual layers; preset macros would hide controls they need.
- The strip is horizontally scrollable; scrollbar hidden. Groups stay intact (no internal wrapping).
- Technical names are kept verbatim — ARINC424, METAR, SIGMETs, Wx, ADS-B — see [Copy stays technical](../.claude/projects/-Users-mariagrilo-Documents-BrazosSimulator/memory/feedback_copy_for_professionals.md). Refinement happens with the analysts, not in the prototype.
- "Timeline" inside the 3D viz is a **viz mode**, not a data layer — lives as `.module__head-toggle` in the 3D module's header actions, not in the layer rail.
- Layers are visual-only right now (no real data providers wired); `is-on` class persists across `applyMosaic()` because module DOM is moved, not rebuilt.

### Pending — broader

- Real 3D visualizer (currently a photo placeholder, pending WebGL integration).
- Real map provider (Mapbox / Google / OSM not chosen yet — currently a photo with track overlay).
- Real flight data — everything is mocked.
- Backend wiring — comments / status / validity changes are session-local.
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
| Hash-based secondary URL (`#window=secondary`) | The `?` query string is taken by htmlpreview.github.io's source URL. A query param would replace the source and the popup would land on github.com. Hash fragments are client-side only. |
| Primary owns the playback rAF | If both windows advanced `STATE.pct` independently they'd drift. The secondary's `setupPlayback()` skips the advance step under `WINDOW_ROLE === 'primary'`. |
| `tick` ≠ `state` | The rAF was broadcasting full `STATE` every frame; a pause click on either side would lose to a queued tick that still claimed `playing=true`. Tick carries only `{pct}` and the receiver ignores it when local `playing` is false. **Don't add `playing` back into the tick payload.** |
| Secondary picker never uses `LAST_POSITION` | `LAST_POSITION` is the analyst-layout home for each module in the primary. Applied to the secondary's stripped tree it wraps incoming modules in nonsense split-v structures and leaves phantom empty panes. The picker uses `appendModuleToLayout()` (flat append) + `request-back` + a dedup guard on the receive. |
| 2-scale cap on charts removed | Analysts asked for "infinite" scales. Charts now render one Y-axis column per distinct scale; the plot area shrinks. The add-param menu no longer disables scale-mismatched params. |

---

## In-browser verification

Two ways the prototype gets exercised during a session:

**Local dev server**

```bash
python3 -m http.server 8765
# primary
open http://localhost:8765/skeleton.html
# secondary (for multi-window work)
open "http://localhost:8765/skeleton.html?window=secondary"
```

**Chrome MCP** — when paired (via `claude-in-chrome`), drive the browser from this session:

- `mcp__claude_in_chrome__list_connected_browsers` / `select_browser` — pair to the user's Chrome.
- `tabs_context_mcp { createIfEmpty: true }` to get a tab group; `tabs_create_mcp` for additional tabs.
- `navigate { tabId, url }`, `javascript_tool { tabId, text }`, and `computer { action: screenshot, tabId }` for verification.
- Note: the MCP-controlled tabs are scoped to a tab group separate from the user's existing tabs. To inspect the user's already-open `localhost:8765` window, ask them to paste console output instead.

The Chrome MCP is how every multi-window change gets sanity-checked: open primary + `?window=secondary` in two MCP tabs, run scripted clicks via `javascript_tool`, then read back `STATE` / `LAYOUT` / `OTHER_MODULES` to verify.

**htmlpreview cache** — `raw.githubusercontent.com` sets `Cache-Control: max-age=300`. After a push, the new file is at GitHub instantly, but a browser that already has the old version cached will keep serving it for up to 5 min. Hard-refresh (Cmd-Shift-R) bypasses, or wait. **This trips us up almost every push** — when the user says "it's not working", first check whether they're on a fresh fetch.

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
- **`pruneModuleFromTree(node, name)` vs `removeModuleFromTree(name)`** — they collided once. The multi-window helper that returns a *new* tree (pure, used by `sendModuleToOtherWindow` and `request-back`) is `pruneModuleFromTree`. The legacy single-arg `removeModuleFromTree` mutates the live LAYOUT in place and is what the Layout popover's `toggleModuleVisibility` calls. Keep the names distinct.
- **Two appends, two semantics** — `addModuleToTree(name)` respects `LAST_POSITION` (analyst-layout home) — use it ONLY from the primary's Layout popover. `appendModuleToLayout(name)` does a flat append with no LAST_POSITION lookup — use it for the secondary's `+ Add module` picker and from `receiveTransferredModule()`. Mixing these up brings back the phantom-pane bug.
- **State sync points** are explicit. Every place that mutates `STATE.pct`, `STATE.viewStart/End`, `STATE.playing`, `STATE.speed`, or `STATE.selectedFinderKey` needs `if (typeof broadcastState === 'function') broadcastState();` at the end (the `typeof` guard is defensive — the function exists once boot is done, but use it anyway). The rAF loop is the exception: it broadcasts `tick { pct }` instead.
- **Don't broadcast from animation frames.** `animateViewTo()` broadcasts once at the end of the tween, not per frame. 60 broadcasts/sec is technically cheap but the receiver re-renders heavily.
- **Empty mosaic in the secondary** is gated by `body.is-secondary:has(#mosaic.mosaic--empty) .mosaic-empty-cta`. `:has()` is widely supported (Chrome 105+, Safari 15.4+, Firefox 121+) but if it ever needs to degrade, the alternative is toggling a body class from `applyMosaic()`.
- **The Chrome MCP's tab group is isolated from the user's existing tabs.** If they have `localhost:8765` open in their normal browser, I can't read it directly — I see only tabs in my MCP group. Best paths: ask for `JSON.stringify(LAYOUT)` from their console, or open a fresh tab in the MCP group and have them rearrange that one.
- **htmlpreview popout** has bitten us twice. The hash flag (`#window=secondary`) fixes the popout URL. After a push, the user's browser may still serve a cached copy for up to 5 min (`Cache-Control: max-age=300` on raw GitHub) — if the user says it's "not working", verify the cached file first via `fetch(url, { cache: 'no-store' })` in DevTools.

When in doubt about a design direction: the user is **mariagrilo** (Subvisual, designer). She knows the analyst context, has talked to Thierry (Brazos), and will push back on bad design choices. Listen to her critiques — she's run `/impeccable:critique` on previous AI proposals and the feedback was correct.
