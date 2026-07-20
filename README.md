# EROD Decoder

A browser-based viewer for FMCSA ELD (Electronic Logging Device) output files — also known as **EROD** (Electronic Records of Duty). No backend, no dependencies, no data ever leaves your machine.

**Live:** https://operations-analytics-zonar.github.io/erod-decoder/

---

## Background

ELD devices generate plain-text output files in the FMCSA-defined format that carriers submit to safety officials. The format is readable but not human-friendly: raw CSV segments with numeric status codes, UTC timestamps, and coordinate pairs. During development, validating the exact data being shared with roadside inspectors required either reading the raw file directly or using specialized hardware — neither option was fast.

This tool was built to close that gap: load an EROD file and immediately see a structured, human-readable view of what's inside.

---

## Usage

1. Open https://operations-analytics-zonar.github.io/erod-decoder/
2. Drag and drop an EROD file onto the page (or click **Browse files**)
3. The file is parsed entirely in your browser — nothing is uploaded anywhere

That's it. No login, no install, no configuration.

---

## What It Shows

### Header
Driver identity, vehicle (VIN, truck/trailer number), carrier (USDOT), ELD provider/firmware, license info, co-driver, shipping document ID, and file checksum.

### Summary
Quick counts: total events, hours by duty status (Driving, On Duty, Off Duty, Sleeper Berth), certifications, and malfunction/diagnostic events.

### Timeline
Visual 24-hour duty status bars, one row per day.

- **Total view** — all days at a glance, colored by status
- **Daily view** — drill into a single day; shows each event, its duration, and a status breakdown table

Hovering a segment shows location coordinates, local time, and duration.

### Tabs

| Tab | Contents |
|---|---|
| Timeline | Duty status visualization (see above) |
| Certifications | Daily certification and recertification actions |
| Events | Full event log — all 16 fields per record |
| Annotations | Driver-added comments tied to event sequence IDs |
| Malf / Diag | Malfunction and data diagnostic events |
| Login / Logout | ELD user session activity |
| Engine | CMV power-up and shutdown events |
| Users | Authorized user list with roles |

---

## EROD File Format

EROD files are plain text, structured as labeled segments. Each segment begins with a header line and contains one or more CSV rows.

```
*ELD File Header Segment*
...header fields...
*ELD Event List*
...one CSV row per event...
*End of File*
checksum
```

### Segments parsed

| # | Segment | Key fields |
|---|---|---|
| 1 | ELD File Header | Driver name, DL, VIN, carrier USDOT, timezone offset, ELD ID/provider/firmware |
| 2 | User List | userId, role, name |
| 3 | CMV List | USDOT, VIN, event count |
| 4 | ELD Event List | Duty status events (see below) |
| 5 | Event Annotations | Sequence-linked driver comments |
| 6 | Driver Certification Actions | Daily log certifications |
| 7 | Malfunctions & Data Diagnostics | System integrity events |
| 8 | Login/Logout Report | Session open/close events |
| 9 | CMV Engine Power-Up/Shutdown | Engine on/off with location |
| 10 | Unidentified Driver Profile Records | Unmatched activity records |
| 11 | End of File | File-level checksum |

### Duty status event fields (Segment 4)

| Field | Values |
|---|---|
| `eventType` | `1` = duty status change |
| `eventCode` | `1` Off Duty · `2` Sleeper Berth · `3` Driving · `4` On Duty |
| `recordStatus` | `1` = active · `0` = inactive |
| `recordOrigin` | `1` = automatic · `2` = manual · `3` = edited |
| `date` / `time` | UTC — displayed in driver's local timezone |
| `accumMiles` / `accumHours` | Odometer and engine hours |
| `lat` / `lon` | GPS coordinates |
| `malfIndicator` | Malfunction flag |
| `dataDiagIndicator` | Data diagnostic flag |

### Timezone handling

The header field `timeZoneOffset` (e.g. `"04"` for UTC-4 Eastern Time) is used to convert all UTC timestamps before display. No timezone database required — the offset is embedded in the file itself.

---

## Architecture

Single-file application: everything lives in `index.html`.

```
index.html
├── <style>          CSS variables, layout, timeline, tables (dark theme)
├── <body>           Drop zone → header card → summary → tabbed content
└── <script>
    ├── parseEROD()      Splits raw text into structured data object
    ├── render*()        One renderer per UI section
    ├── buildTabs()      Wires up all tab panels after parse
    ├── utcToLocal()     UTC → driver local time (uses TZ_OFFSET_HOURS)
    └── durationStr()    Minutes → h:mm display
```

**No build step.** No bundler, no framework. Runtime dependencies loaded via CDN: Roboto and Roboto Mono from Google Fonts, and [Lucide](https://lucide.dev) icons from unpkg. Everything else is inline in `index.html`.

**Data flow:**

```
File drop / input
  → FileReader (reads as text)
    → parseEROD() (segments → arrays of objects)
      → render*() functions (DOM updates per tab)
```

Parsed data is held in a module-level variable. Switching tabs re-renders from that in-memory object — no re-parsing.

---

## Deployment

Hosted on GitHub Pages. Deploys automatically on every push to `master` via `.github/workflows/pages.yml`.

```yaml
on:
  push:
    branches: [master]
```

No build step — the workflow uploads the repo directory as-is and serves it through the Pages CDN.

To run locally, just open `index.html` in a browser. File I/O uses the Web File API, which works without a server.

---

## Development

There is nothing to install. Edit `index.html` and open it in a browser.

Common touch points:

| Task | Where |
|---|---|
| Parse a new segment | `parseEROD()` in the `<script>` block |
| Add a UI section / tab | Add a `<div class="tab-content">` and a matching button in the tab bar, then a `render*()` function |
| Change duty status colors | CSS variables at the top of the `<style>` block (`--driving`, `--off-duty`, `--sleeper`, `--on-duty`) — and keep the `DUTY_HEX` map in the `<script>` block in sync |
| Change theme tokens | Ignition tokens (`--ig-bg`, `--ig-border`, `--ig-font`, …) at the top of the `<style>` block |
| Change timezone logic | `utcToLocal(date, time)` — reads the global `TZ_OFFSET_HOURS` set at parse time from the file header |

---

## Status Codes Reference

### Duty Status (`eventCode`)
| Code | Status | Color |
|---|---|---|
| 1 | Off Duty | Emerald |
| 2 | Sleeper Berth | Blue |
| 3 | Driving | Red |
| 4 | On Duty (Not Driving) | Amber |

### Record Origin (`recordOrigin`)
| Code | Meaning |
|---|---|
| 1 | Automatic (ELD system) |
| 2 | Manual (driver input) |
| 3 | Edited (logged-in user) |

---

## Related

- [FMCSA ELD Technical Standard (49 CFR Part 395)](https://www.fmcsa.dot.gov/hours-service/elds/electronic-logging-devices) — the spec this file format comes from
- [Zonar ELD product](https://www.zonarsystems.com) — the ELD generating the files this tool was built to inspect
