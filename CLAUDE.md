# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Single-file browser tool (`index.html`) for parsing and visualizing FMCSA EROD (Electronic Records of Duty) files — the plain-text output that Zonar ELD devices submit to roadside safety officials. Everything runs client-side; no backend, no build step, no dependencies at runtime except Roboto/Roboto Mono from Google Fonts and Lucide icons via CDN.

Live: https://operations-analytics-zonar.github.io/erod-decoder/

## Development

No install, no build. Open `index.html` directly in a browser. The File API works without a local server.

Deploy is automatic — every push to `master` triggers `.github/workflows/pages.yml`, which uploads the repo as-is to GitHub Pages.

## Architecture

Everything lives in `index.html` in three logical blocks:

**CSS (`<style>`)** — Zonar Ignition design system tokens as CSS variables (`--ig-*`), then component styles that consume them. Dark theme is the default (`--ig-bg: #0F141C`). Duty status colors use the brand palette (`--off-duty`, `--sleeper`, `--driving`, `--on-duty`) defined separately from the Ignition tokens.

**HTML (`<body>`)** — Two top-level states: `#drop-zone` (initial, file picker) and `#content` (post-parse, hidden until a file loads). Content is a topbar, a header card, a summary row, and a tab system — all populated by JS after parsing.

**JavaScript (`<script>`)** — Linear, no framework:
- `parseEROD(rawText)` — splits raw text on the 11 FMCSA segment labels, returns a structured data object
- `render*()` functions — one per UI section, each returns an HTML string injected into the DOM
- `buildTabs(data)` — wires up all tab panels at once after a file loads
- `renderDailyNav(idx)` / `showTotalView()` / `showDailyView()` — timeline view switching (state lives in `_tlData`)
- `utcToLocal(d, t)` — converts UTC date+time strings using `TZ_OFFSET_HOURS`, which is set from the file's `timeZoneOffset` header field on every parse

## Key data facts

- All timestamps in EROD files are UTC. `TZ_OFFSET_HOURS` is a global updated at parse time from the file header.
- `eventType === 1` filters for duty status changes. Other event types (intermediate, driver indication) exist but are not plotted on the timeline.
- `recordStatus === 1` means active. Inactive records are displayed in the Events tab but excluded from the timeline.
- Duty status `eventCode`: 1 = Off Duty, 2 = Sleeper Berth, 3 = Driving, 4 = On Duty (Not Driving).
- `DUTY_HEX` must stay in sync with the CSS `--off-duty / --sleeper / --driving / --on-duty` values — it's used for tooltip dot colors where CSS vars can't be read directly.

## Design system rules (Ignition)

- **Roboto only** — no secondary typeface.
- **No `backdrop-filter`**, no gradients on surfaces, no transform lifts on hover.
- **Lucide icons** via `https://unpkg.com/lucide@latest` — call `lucide.createIcons()` after any DOM update that introduces new `data-lucide` attributes.
- All tokens used must exist in the `--ig-*` list in the CSS block. No invented `var()` names.
- Sentence case for all labels, buttons, and headings.
