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
| `app.js` | All frontend behavior (~419 lines) |
| `styles.css` | Mobile-first responsive design (~277 lines) |

**Backend:** External REST API at `https://jaundice-server.onrender.com`. The frontend POSTs JSON to `/api/chat` with a `clientId` header.

### `app.js` internals

- **Session identity:** UUID stored in `localStorage` as `clientId`; no login required.
- **Retry logic:** Three distinct error paths — empty response, incomplete-marker response, and HTTP/network error — each with its own retry count and message.
- **Markdown rendering:** Uses the bundled `marked.umd.js` (no CDN dependency).
- **XSS prevention:** User input is HTML-escaped before insertion into the DOM.

### CSS conventions

- Uses CSS custom properties (`--color-primary`, etc.) for theming.
- Warm healthcare palette: cream `#FFFBF2`, amber `#F59E0B`.
- Mobile-first; uses `safe-area-inset` and dynamic `vh` units for keyboard-aware layout.

## Static assets

- `assets/` — logo and avatar images (logo, user, bot).
- `pic/` — educational images embedded in chat responses (stool color chart, bile duct diagram).
- `marked.umd.js` — bundled markdown parser; update by replacing this file with a newer UMD build from the marked project.
