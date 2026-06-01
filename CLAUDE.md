# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A year-view Google Calendar planner web app hosted at [visual-planner.github.io](https://visual-planner.github.io/). No build system — files are served directly as static HTML/CSS/JS.

## Local Development

Serve the files with any static HTTP server on **port 8080**:

```bash
python3 -m http.server 8080
# or
npx serve -p 8080
```

The `vpGrid` directive in `vplib.js` auto-detects `localhost:8080` and loads `vpgrid.htm` from the local server instead of the production URL. Without port 8080, the directive fetches the template from `visual-planner.github.io`.

Open `http://localhost:8080/vp.htm` to use the app. It will request Google OAuth2 permissions on load.

## Architecture

**No build pipeline.** All logic lives in a single file (`vplib.js`) loaded by `vp.htm`.

### File roles

| File | Role |
|---|---|
| `vp.htm` | Main app shell — toolbar, settings panel, Angular app bootstrap |
| `vpgrid.htm` | Angular directive template for the calendar grid. Contains CSS with Angular interpolation (`{{vpgrid.fontscale}}`) embedded in `<style>` tags — this is intentional |
| `vpprint.htm` | Print view — opens in a new window, reads data from `window.opener.vpprint` |
| `vplib.js` | All application logic (Angular module + services + utilities) |
| `vpmanifest.json` | PWA manifest — makes the app installable |

### Angular services in `vplib.js`

- **`vpConfiguration`** — OAuth2 auth (Google Identity Services), loads/saves settings to Google Drive `appDataFolder`, manages `localStorage` for view state (`vp-gridviewinfo`, `vp-caltoginfo`)
- **`vpGCal`** — Google Calendar API client: fetches calendar list and events, handles sync tokens, constructs `VpCalendar` and `VpEvent` objects
- **`vpDiary`** — Data model layer: builds the page of `VpMonth`/`VpDay`/`VpLabel` objects from events, handles event placement into grid slots
- **`vpGrid` directive** — View controller: manages scroll position, layout toggling (column/list, expand/collapse), print handoff, keyboard shortcuts

### Utility classes (global, not Angular)

- `VpDate` / `VpDateMonth` / `VpDateTime` — Date wrappers with YMD formatting, month/day arithmetic, Google Calendar URL generation
- `fmt()` — String interpolation using `^` as the placeholder character

### State persistence

- **Google Drive `appDataFolder`**: user settings (`settings002.json`) — title, month count, display options
- **`localStorage`**:
  - `vp-gridviewinfo`: current view mode (column/list, expand/collapse, dark mode)
  - `vp-caltoginfo`: per-calendar visibility toggles

### Data flow

1. `vpConfiguration.Load()` is called on `gapi` ready → requests OAuth2 token → loads Drive settings → fires `config:load` event
2. `vpGCal` listens for `config:load` → fetches calendar list → fires callback to populate `$rootScope.vp.calendarlist`
3. `vpGrid` directive watches `vp.calendarlist` → calls `vpDiary.makePage()` → fetches events from `vpGCal` → renders grid
