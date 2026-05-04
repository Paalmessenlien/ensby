# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained static page, `Ensby26_map.html`, that renders one hardcoded GPS track on a Leaflet map with a custom SVG elevation profile, plus numbered stop markers. No build step, no package manager, no tests, no server — open the file directly in a browser. Published to GitHub Pages at https://paalmessenlien.github.io/ensby/ (root-level `index.html` redirects to the map).

The track location is in Innlandet, Norway (~61.26°N, 10.41°E), which is why Kartverket topo tiles are the default layer.

## Track data

The `TRACK` constant (a single long line in the `<script>` block; find with `grep -n 'const TRACK' Ensby26_map.html`) is the source of truth: an array of `[lat, lon, elevation_m]` tuples. The current track is derived from `tur1.gpx` (Norgeskart export). To swap tracks, replace the array; everything downstream (header stats, polyline, elevation SVG, hover sync, stop markers) recomputes from it on load.

Stops in the `STOPS` array reference TRACK by index (`{n, i}`). They were detected from `tur1.gpx`'s timestamps as low-velocity pause segments (speed < 0.5 m/s for ≥ 5 s, near-duplicates merged within 30 m). If you replace TRACK, recompute STOPS or the indices will point at the wrong points.

## Architecture

Linear, top-to-bottom in the `<script>` block:

1. Compute `cumDist` (haversine), `eles`, total distance, ascent/descent, min/max — populate the header stats.
2. Build the Leaflet map with three swappable tile layers (Kartverket topo default, OSM, Esri satellite).
3. Draw the track as a red polyline, fit bounds, add start/end circle markers.
4. Hand-roll the elevation profile as SVG (no charting library) using `cumDist` for x and `eles` for y.
5. A `mousemove` handler on the SVG positions a dashed cursor + label and syncs a `hoverMarker` on the map; `mouseleave` tears them down.

Coordinates are `[lat, lon, ele]` throughout. Leaflet expects `[lat, lon]`, so `latlngs` is built by stripping the elevation.

## Gotchas

- **Leaflet is pinned with SRI hashes.** Bumping the version requires updating both the URL and the `integrity=` attribute on the `<link>` (CSS) and `<script>` (JS) tags. Mismatched hashes cause the browser to refuse to load the asset silently-ish.
- **Kartverket tiles are Norway-only** — outside Norway they return blank. Switch the default tile layer if porting the viewer elsewhere.
- **SVG cursor math is tied to fixed constants.** The `<svg>` uses `viewBox="0 0 1000 140"` with `preserveAspectRatio="none"`, and the `mousemove` handler maps client coords back into that viewBox space using `W`, `H`, `padL`, `padR`, `padT`, `padB`. Change those and the cursor positioning needs re-deriving.
- **Tile servers and CDN need internet.** There is no offline fallback.
