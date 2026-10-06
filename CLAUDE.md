# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`chgeo` ("Geografia svizzera") is a static, dependency-free geography quiz, available in **Italian, German, French, Romansh and English**, aimed at learning Swiss cantons, capitals, rivers and lakes, plus Ticino's districts, towns, rivers and lakes. The player picks a card, taps the matching place on an SVG map, then presses **Verifica** to score. It is hosted on GitHub Pages from `main` / root (`.nojekyll` present) at the custom domain **chgeo.giar.dev**, which is set by the `CNAME` file. Don't remove or rename `CNAME`, or the custom domain stops working. It is also an installable PWA via `manifest.webmanifest` (no service worker).

The footer carries the required data attributions (swisstopo, "© contributori OpenStreetMap" linked to openstreetmap.org/copyright) and a plain Ko-fi donation link (`ko-fi.com/miguelgila`). Keep the link plain: no third-party widgets, scripts, analytics or cookies, since the audience is schoolchildren and the site currently needs no consent banner. Code is MIT (`LICENSE`); `data.js` is ODbL/swisstopo (`DATA_LICENSE`).

There is no build step, no package manager, no tests and no linter. Never hard-code user-facing text in `index.html`: add a key to every language in `i18n.js`.

## Run locally

```sh
python3 -m http.server 8000   # then open http://localhost:8000
```

## Architecture

There are three source files:

- **`i18n.js`**: `window.I18N`, with three parts:
  - `langs`: the list of languages.
  - `ui[lang][key]`: flat UI strings with `{0}`/`{1}` placeholders. Mode strings are keyed `mode.<id>.<label|tray|subtitle|what|help>`, and maps are keyed `map.<id>` and `map.<id>.title`.
  - `places[targetId][lang]`: place names keyed by the same target IDs as the game (`ZH`, `cap:ZH`, `r:Reno`, `l:Lemano`, `d:…`, `c:…`).

  Missing UI keys fall back to Italian, and missing places fall back to the name in `data.js`. Every language must define the same UI keys and placeholders. Romansh strings still need review by a native speaker. In German and French, cards are "Kärtchen" and "fiches", because "Karte"/"carte" already means the map.

- **`data.js`**: generated geometry, assigned to `window.GEO`. Do not hand-edit the large path strings. It was produced once by Claude Design, and that generator was not kept, so `data.js` cannot currently be regenerated. A reproducible pipeline is planned in issue #3 (swisstopo LV95 data → SVG, with simplification that keeps shared borders intact). Its shape:
  - `GEO.ch`: `w:900, h:600`, `cantons[{name,abbr,d,cx,cy}]` (26), `capitals{abbr→name}`, `rivers[{name,d,cx,cy}]`, `lakes[{name,d,cx,cy}]`
  - `GEO.ti`: `w:600, h:800` (portrait), `districts[{name,abbr,d,cx,cy}]`, `cities[{name,cx,cy}]` (pins only, no path), `rivers`, `lakes`
  - `d` is an SVG path in map coordinates; `(cx, cy)` is the label/chip anchor.
- **`index.html`**: markup, CSS and one inline IIFE script. Its main parts:
  - **`MAPS`**: one entry per map (`ch`, `ti`). Each entry normalises the `GEO` data into `zones`, `lakes`, `rivers` and `pins`, and lists the `modes` (decks) for that map.
  - **Mode/deck**: `{id, label, tray, subtitle, what, help, kind, targets(), hints?}`. `kind` is one of `zone | pin | river | lake | water` and controls which SVG layers are clickable and coloured. `targets()` returns `{id, name, cx, cy}`.
  - **Target ID convention**: target IDs must match the IDs that `buildSvg()` binds to click handlers. Canton zones use `abbr` (e.g. `ZH`), and capitals reuse the canton `abbr`. The other prefixes are `d:<name>` for districts, `c:<name>` for cities, `r:<name>` for rivers and `l:<name>` for lakes.
  - **State `S`**: `{map, mode, order, selected, placed{targetId→cardId}, checked, hints, names, view{z,cx,cy}}`. `fresh()` reshuffles the deck and turns `names` off. `setMap()` rebuilds the SVG. `setMode()` only re-renders.
  - **`buildSvg()`** runs once per map and creates the DOM nodes. Rivers get a second, invisible, wider `riverHit` path, and pins a transparent `pinHit` circle, as tap targets. `applyView()` sizes both in screen pixels.
  - **Order matters in `setMap()`**: call `buildSvg()`, then `render()`, then `applyView()`. `applyView()` reads `box.clientWidth`, which forces a style pass. If that runs before `render()` has set the fills, the new shapes get computed with no fill (black), and `.zone`'s `transition: fill` animates from black.
  - **`render()`** runs on every state change and recolours the existing nodes through `status()` (idle, placed, ok, bad or unplaced colours). It also rebuilds the card tray, progress, results and the "Da ripassare" error list. `renderChips()` draws HTML overlay labels at each target's `(cx, cy)`. After a check, wrong placements get red chips with the correct name. With `S.names` (the "Mostra nomi" toggle, available in every deck), every target gets a neutral `.chip.name` label, and the red chips stay on top.
  - **Zoom/pan** works by rewriting the SVG `viewBox` (`S.view`, zoom range 1–8). It supports pointer drag, two-finger pinch and ctrl/wheel. A drag sets `dragged` so that the pointer-up does not count as a pick.
  - The last selected map is persisted in `localStorage['chgeo.map']`.
  - **i18n**: `t(key, ...args)` returns UI strings, `pn(id, fallback)` returns place names, and `mt(k)` returns strings for the current mode. Static markup is translated through `data-i18n` (text) and `data-i18n-aria` (aria-label) attributes. The language is chosen by `?lang=xx`, then `localStorage['chgeo.lang']`, then Italian. The browser language is deliberately ignored, because the default is Italian by the owner's choice. `setLang()` re-translates without reshuffling, so a game in progress survives a language switch: `names()` rebuilds the targets, while `fresh()` also reshuffles.
  - Target IDs are derived from the **Italian names in `data.js`**, so they are stable keys, not display text. Display names always go through `pn()`.

## Adding a deck

Add a mode to the relevant `MAPS[...].modes` array in `index.html`. Any new geometry must first exist in `data.js` and must be wired into that map's `zones`/`lakes`/`rivers`/`pins`, otherwise there will be no clickable element whose ID matches the target.

## Data sources (keep attribution in README)

- Cantons: `click_that_hood` (OSM, ODbL).
- Ticino districts: `swiss-maps` (swissBOUNDARIES3D, swisstopo).
- Hydrography: swissTLMRegio (swisstopo).
- Town positions: hand-placed from LV95 coordinates.
