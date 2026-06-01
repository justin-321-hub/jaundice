# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Traditional Chinese chatbot frontend for postpartum newborn jaundice home care education ("新生兒黃疸居家照護衛教智慧客服小幫手"). Vanilla JS single-page app — no build step, no npm, no framework.

## Running Locally

Serve the static files with any HTTP server, e.g.:

```bash
python -m http.server 8080
# or
npx http-server .
```

Then open `http://localhost:8080` in a browser.

## Deployment

GitHub Actions (`.github/workflows/static.yml`) auto-deploys to GitHub Pages on every push to `main`. No manual deploy step.

## Architecture

All logic lives in three files at the root:

| File | Role |
|---|---|
| `index.html` | Shell — semantic layout (header / main / footer), no inline logic |
| `app.js` | All frontend behavior (~435 lines) |
| `styles.css` | Mobile-first responsive design (~277 lines) |

**Backend:** External REST API at `https://jaundice-server.onrender.com`. The frontend POSTs JSON to `/api/chat` with these fields:

```json
{ "text": "...", "clientId": "...", "babyId": null, "parentId": null, "language": "繁體中文", "role": "user" }
```

The `clientId` is also sent as an `X-Client-Id` request header.

### `app.js` internals

**Session identity:** `clientId` is a UUID stored in `localStorage` under the key `fourleaf_client_id`. `babyId` is stored under `babyID`.

**WebView bridge:** Native mobile apps inject identifiers after the page loads via two functions:
- `window.setBabyId(id)` — injects the patient's baby ID; persisted under `babyID` in localStorage.
- `window.setParentId(id)` — injects the logged-in parent's Firebase UID; persisted under `parentID` in localStorage.

Both values are sent to the API on every request.

**Input preprocessing:** `processQuestionMarks()` strips trailing `?`/`？` and converts mid-sentence question marks to newlines before the text is sent to the API.

**Content rendering pipeline (bot messages only):**
1. `isHtmlFormat()` — if the response contains HTML tags, inject it directly (no escaping).
2. `processContent()` — detects Markdown patterns; if found and `marked` is available, runs `marked.parse()`.
3. Fallback — `escapeHtml()` + replace `\n` with `<br>`. User messages always use this fallback.

**Temp messages:** During a pending request, progress messages (`{ isTemp: true }`) are pushed to the `messages` array at 4 s and 8 s. `clearAllTempMessages()` removes all of them before the final reply or error is rendered.

**Concurrency guard:** `window.isChatFetching` blocks re-entry on double-click. `window.globalReqId` (incremented per request) lets stale timeout callbacks detect they've been superseded and abort their `updateTempMsg()` call.

**Retry logic:** Three independent retry counters in `retryCounts`:
- `emptyResponse` — HTTP 200 with empty/null `text` field (max 1 retry).
- `incompleteMarkers` — response body contains both `"search results"` and `"html"` (max 1 retry).
- `httpErrors` — HTTP 500/502/503/504/401/404 (max 1 retry).

Each path inserts an interim status message, waits 1 s, then calls `sendText()` recursively. Second failure throws and lands in the catch block.

**Avatar images:** Loaded from `raw.githubusercontent.com` CDN URLs (not the local `assets/` folder), so they require an internet connection even when running locally.

### CSS conventions

- Uses CSS custom properties (`--color-primary`, etc.) for theming.
- Warm healthcare palette: cream `#FFFBF2`, amber `#F59E0B`.
- Mobile-first; uses `safe-area-inset` and dynamic `dvh` units for keyboard-aware layout on iOS.
- `.hidden { display: none !important }` is the sole visibility toggle used throughout.

## Static assets

- `assets/` — logo and avatar images (logo, user, bot). The bot/user avatars in `assets/` are not used at runtime; `app.js` loads them from GitHub CDN.
- `pic/` — educational images embedded in chat responses (stool color chart, bile duct diagram).
- `marked.umd.js` — bundled markdown parser; update by replacing this file with a newer UMD build from the marked project.
