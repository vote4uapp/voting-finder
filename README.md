# Vote4U

A nonpartisan civic-engagement web app built for the 2028 U.S. presidential election cycle. It gives visitors three tools in one place: a live electoral-map tally, a polling-place finder that resolves any ZIP code to real voting locations, and a curated feed of 2028-race news.

## What it does

- **Electoral map & tally** (`/tools`, "Electoral Map" tab) — a state-by-state map with running Democrat / Republican / swing-state electoral vote counts, built from a static state dataset (`client/src/data/stateData.js`).
- **Polling place finder** (`/tools`, "Find My Polling Booth" tab) — enter a 5-digit ZIP code and get nearby voting locations. The server tries three data sources in order and always returns *something*:
  1. **Google Civic Information API** — official polling/early-voting locations for the address, tried across the nearest upcoming election IDs.
  2. **OpenStreetMap Nominatim** — if no official data is available, falls back to nearby libraries, schools, community centers, and town halls within 10 km, geocoded and distance-ranked.
  3. **Sample data** — if both fail (e.g. no API key configured, or the network is unreachable), generates clearly-labeled placeholder locations so the UI never breaks.
  Results are cached in Postgres (`polling_cache`, 7-day TTL) keyed by ZIP.
- **Election news feed** (`/news`) — pulls 2028-cycle articles from NewsAPI, tags each one with a candidate (from a fixed watchlist: Newsom, Whitmer, AOC, Buttigieg, Vance, Youngkin, Hawley, DeSantis, Scott, etc.) and an inferred party, then dedupes and caches the result in Postgres (`news_cache`, 14-day TTL).

## Architecture

```
voting-finder/
├── client/          React 19 + Vite SPA (React Router, Tailwind)
│   └── src/
│       ├── pages/       HomePage, AboutPage, ToolsPage, NewsPage
│       ├── components/  ElectoralMap, PollingLocationCard, NewsCard, NavBar
│       ├── hooks/        usePolling, useNews (fetch + loading/error state)
│       └── data/         stateData.js — static electoral-vote/party data
├── server/          Express 5 API
│   ├── routes/       elections.js, polling.js, news.js
│   ├── services/     civicService (Google Civic), geocodeService (Zippopotam + Nominatim), newsService (NewsAPI)
│   ├── middleware/   cors.js, rateLimit.js
│   └── db/           schema.sql (Postgres cache tables), client.js
├── vercel.json      Deploys the client as a static Vite build
└── package.json     Root scripts to run client+server together in dev
```

The client and server are two independent Node projects (`client/package.json`, `server/package.json`) orchestrated by the root `package.json` via `concurrently`.

## Setup & running locally

Requires Node ≥18 and a Postgres instance (for the two cache tables — the app degrades gracefully without one, just without caching).

```bash
npm run install:all          # installs root, client, and server deps

cp .env.example server/.env  # fill in the values below
cp .env.example client/.env  # only needed for VITE_API_URL in production

# apply the schema once against your Postgres instance
psql "$DATABASE_URL" -f server/db/schema.sql

npm run dev                  # runs client (Vite, :5173) and server (:3001) together
```

**Server environment variables** (`server/.env`):

| Variable | Required | Purpose |
|---|---|---|
| `GOOGLE_CIVIC_API_KEY` | optional | Enables official polling-location + election-ID lookups. Without it, the app falls back to OSM/sample data. |
| `NEWS_API_KEY` | optional | Enables the `/news` feed. Without it, the endpoint returns a 503. |
| `DATABASE_URL` | optional | Postgres connection string for response caching. Without it, routes still work but hit the live APIs every request. |
| `CLIENT_URL` | yes | Client origin, for CORS. |
| `PORT` | no (default 3001) | Server port. |

No API keys or secrets are committed to the repo — `.env` files are gitignored, and `.env.example` only ships placeholders.

## Deployment

- **Client**: `vercel.json` builds `client/` with Vite and serves the static output; SPA routes are rewritten to `index.html`.
- **Server**: any Node host (Railway, Render, etc.) — `server/package.json`'s `start` script runs `node index.js`.

## Current status

Functional full-stack app with three working tools (electoral map, polling finder, news feed) and a three-tier fallback strategy for polling data so the UI has something to show even without API keys configured. No automated test suite yet. The news feed's candidate/party tagging is a fixed keyword watchlist (`server/services/newsService.js`), not a general NER model, so it only recognizes candidates named in that list.
