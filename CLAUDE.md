# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`chgeo` ("Geografia svizzera") is a static, dependency-free geography quiz **in Italian**, aimed at learning Swiss cantons, capitals, rivers and lakes, plus Ticino's districts, towns, rivers and lakes. The player picks a card, taps the matching place on an SVG map, then presses **Verifica** to score. It is hosted on GitHub Pages from `main` / root (`.nojekyll` present); it is also an installable PWA via `manifest.webmanifest` (no service worker).

There is no build step, no package manager, no tests and no linter. All user-facing text is Italian; keep it that way.

## Run locally

```sh
python3 -m http.server 8000   # then open http://localhost:8000
```

## Architecture

There are only two source files:

- **`data.js`**: generated geometry, assigned to `window.GEO`. Do not hand-edit the large path strings. It comes from an external Python pipeline that is not in this repo (simplification plus projection to SVG coordinates). Its shape:
  - `GEO.ch`: `w:900, h:600`, `cantons[{name,abbr,d,cx,cy}]` (26), `capitals{abbr→name}`, `rivers[{name,d,cx,cy}]`, `lakes[{name,d,cx,cy}]`
  - `GEO.ti`: `w:600, h:800` (portrait), `districts[{name,abbr,d,cx,cy}]`, `cities[{name,cx,cy}]` (pins only, no path), `rivers`, `lakes`
  - `d` is an SVG path in map coordinates; `(cx, cy)` is the label/chip anchor.
- **`index.html`**: markup, CSS and one inline IIFE script. Its main parts:
  - **`MAPS`**: one entry per map (`ch`, `ti`). Each entry normalises the `GEO` data into `zones`, `lakes`, `rivers` and `pins`, and lists the `modes` (decks) for that map.
  - **Mode/deck**: `{id, label, tray, subtitle, what, help, kind, targets(), hints?}`. `kind` is one of `zone | pin | river | lake | water` and controls which SVG layers are clickable and coloured. `targets()` returns `{id, name, cx, cy}`.
  - **Target ID convention**: target IDs must match the IDs that `buildSvg()` binds to click handlers. Canton zones use `abbr` (e.g. `ZH`), and capitals reuse the canton `abbr`. The other prefixes are `d:<name>` for districts, `c:<name>` for cities, `r:<name>` for rivers and `l:<name>` for lakes.
  - **State `S`**: `{map, mode, order, selected, placed{targetId→cardId}, checked, hints, view{z,cx,cy}}`. `fresh()` reshuffles the deck. `setMap()` rebuilds the SVG. `setMode()` only re-renders.
  - **`buildSvg()`** runs once per map and creates the DOM nodes. Rivers get a second, invisible, wider `riverHit` path that serves as the tap target.
  - **`render()`** runs on every state change and recolours the existing nodes through `status()` (idle, placed, ok, bad or unplaced colours). It also rebuilds the card tray, progress, results and the "Da ripassare" error list. After a check, `renderChips()` positions the correct-name labels over wrong placements as HTML overlays.
  - **Zoom/pan** works by rewriting the SVG `viewBox` (`S.view`, zoom range 1–8). It supports pointer drag, two-finger pinch and ctrl/wheel. A drag sets `dragged` so that the pointer-up does not count as a pick.
  - The last selected map is persisted in `localStorage['chgeo.map']`.

## Adding a deck

Add a mode to the relevant `MAPS[...].modes` array in `index.html`. Any new geometry must first exist in `data.js` and must be wired into that map's `zones`/`lakes`/`rivers`/`pins`, otherwise there will be no clickable element whose ID matches the target.

## Data sources (keep attribution in README)

- Cantons: `click_that_hood` (OSM, ODbL).
- Ticino districts: `swiss-maps` (swissBOUNDARIES3D, swisstopo).
- Hydrography: swissTLMRegio (swisstopo).
- Town positions: hand-placed from LV95 coordinates.
