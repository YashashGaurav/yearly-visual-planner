# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A year-view Google Calendar planner web app. No build system — files are served directly as static HTML/CSS/JS.

- **Production URL**: `https://yashashgaurav.com/yearly-visual-planner/vp.htm`
- **Deployed via**: GitHub Actions on push to `main` (see `.github/workflows/deploy.yml`)

## Local Development

```bash
make serve
```

Opens the app at `http://localhost:8080/vp.htm`. Port 8080 is required — `vplib.js` uses `window.location.href` to construct the template URL, so it resolves correctly from whatever origin serves `vp.htm`.

## Deployment

GitHub Actions deploys to GitHub Pages automatically on every push to `main`. The live URL inherits the custom domain from `yashashgaurav.github.io` → `yashashgaurav.com`.

## Staging

**Staging URL**: `https://yashashgaurav.com/yearly-visual-planner/staging/vp.htm`

Staging exists because it shares the same Google OAuth origin (`https://yashashgaurav.com`) as production — no Google Cloud Console changes are needed to test a feature branch with real OAuth. It is also the only way to validate mobile flows (e.g., Safari popup blocking) that do not reproduce in local development.

The deploy mechanism is a single composite GitHub Pages artifact: the workflow checks out `main` at the site root and the `staging` branch under `/staging/`, then deploys them together in one step.

### Using staging for a new feature

1. Create a feature branch from `main` and make your changes.
2. To preview on staging: `git push origin <feature-branch>:staging --force`
   (force is required because the `staging` branch's history is replaced per feature).
3. Wait ~1 minute for the GitHub Pages deploy workflow to finish, then test at the staging URL.
4. When the feature is verified, open a PR from your feature branch → `main`. Merging deploys to production.

> **Note**: The `staging` branch must remain in the github-pages environment's branch policy (Settings → Environments → github-pages → Deployment branch policies). This was set up once and does not need to be repeated for new features.

## Google OAuth

The app uses Google Identity Services (GIS) with OAuth client ID from the `yearly-visual-planner-app` Google Cloud project. Authorized origins: `http://localhost:8080` and `https://yashashgaurav.com`.

Required APIs enabled in that project: **Google Calendar API** and **Google Drive API**.

The OAuth consent screen is in **Testing** mode — only `yashash.gaurav@gmail.com` is a registered test user.

Scopes requested:
- `calendar.readonly` — read calendar events
- `drive.appdata` — read/write settings file in the app's private Drive folder only

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
