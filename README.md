# chgeo — Geografia svizzera

A small, dependency-free geography quiz (in Italian, German, French, Romansh and English) for learning the Swiss cantons, capitals, rivers and lakes, plus the districts, towns, rivers and lakes of Ticino. Pick a card, tap the matching place on the map, then press **Verifica** to see what matched.

It is a single static page: `index.html` (markup, styles, logic), `i18n.js` (UI strings and place names per language) and `data.js` (map geometry). The default language is Italian; it can be changed in the header (the choice is remembered) or forced with `?lang=it|de|fr|rm|en`. No build step, no backend.

## Run locally

Any static file server works, e.g.

```sh
python3 -m http.server 8000
```

then open <http://localhost:8000>.

## Hosting on GitHub Pages

Settings → Pages → *Deploy from a branch* → `main` / root. The site is then served at `https://<user>.github.io/chgeo/`.

To use a custom domain such as `chgeo.giar.dev`: add a `CNAME` file containing `chgeo.giar.dev` to the repo root, and at the DNS provider add a `CNAME` record `chgeo` → `<user>.github.io`. GitHub provisions the TLS certificate automatically; tick *Enforce HTTPS* once it is issued.

## Data

- Canton outlines: `click_that_hood` (OpenStreetMap-derived, ODbL).
- Ticino districts: `swiss-maps` npm package (swissBOUNDARIES3D, swisstopo).
- Rivers and lakes: swissTLMRegio hydrography (swisstopo, open government data).
- Town positions: hand-placed from LV95 coordinates.

Geometry was simplified and projected to SVG coordinates by a one-off generator that is not preserved; each entry carries its path (`d`) and a label anchor (`cx`, `cy`). A reproducible pipeline is planned in [#3](https://github.com/miguelgila/chgeo/issues/3).

## License

- **Code** (`index.html`, icons, manifest): [MIT](LICENSE).
- **Map data** (`data.js`): not MIT. The canton outlines are derived from OpenStreetMap (© OpenStreetMap contributors) and distributed under the [ODbL 1.0](https://opendatacommons.org/licenses/odbl/1-0/); the remaining geometry is derived from swisstopo open government data (source: Federal Office of Topography swisstopo). See [DATA_LICENSE](DATA_LICENSE).

## Adding a deck

Decks live in the `MAPS` object at the top of the script in `index.html`. A deck is a `mode` with a `kind` (`zone`, `pin`, `river`, `lake` or `water`) and a `targets()` function returning `{id, name, cx, cy}` items; the geometry it refers to must exist in `data.js`.
