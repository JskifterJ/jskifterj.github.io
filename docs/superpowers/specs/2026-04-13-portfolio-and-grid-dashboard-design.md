# Design Spec: Personal Portfolio + EU Grid Dashboard

**Date:** 2026-04-13  
**Author:** Johannes Skifter  
**Status:** Approved

---

## Overview

Two linked GitHub repositories:

1. **`jskifterj/portfolio`** — personal portfolio site, hosted on GitHub Pages at `jskifterj.github.io`
2. **`jskifterj/eu-grid-dashboard`** — live European electricity grid dashboard with AI briefing, hosted on Render (free tier)

The portfolio introduces Johannes and links to the dashboard as its featured project. Together they serve as a general professional portfolio demonstrating the intersection of business/finance, data engineering, and software — targeted at roles like Hitachi Energy's Power+ Graduate Program but broadly applicable.

---

## 1. Portfolio Site (`jskifterj/portfolio`)

### Visual Design

- **Theme:** Dark technical, GitHub-inspired (`#0d1117` background, `#7ee787` green accent)
- **Typography:** System UI for body, `SF Mono`/`Fira Code` for monospace accents
- **Tone:** Technically credible but playful and self-deprecating — not a brag sheet

### Layout

Single-page scroll with expandable deep-dives (modals or anchor-linked sections for project detail). Five sections:

1. **Hero** — Name, tagline ("Technically not an engineer — but I've been taking things apart since before that was a career path."), role/institution line, tag pills (Energy systems, Finance & analytics, Python/ML/SQL, Hardware + software), metrics strip, venture log, CTAs
2. **Experience** — Timeline of professional roles (Impossible Cloud, FIH Partners, Orbitvu Nordic)
3. **Projects** — Featured card: EU Grid Dashboard (live embed/link). Secondary cards for GitHub projects and academic work
4. **Education & Skills** — EPFL/IMD/HEC, CBS, Wharton; languages and tools
5. **Contact** — Email, GitHub, LinkedIn

### Metrics Strip (Hero)

Five stats rendered in monospace:

| Stat | Value | Label |
|------|-------|-------|
| GPA | 5.7/6 | GPA |
| IB screening | 40+ | Co's screened |
| Revenue | 3.5M DKK | Revenue grown |
| Electronics | €30k+ | Flipped |
| First biz | age 16 | First import biz |

### Venture Log (Hero)

Terminal-style `git log --oneline --ventures` block:

- `16 —` Imported stickers from China. *Margin was great. Scale was not.*
- `17 —` Mini ad agency: websites + Google Ads. *Closed it. The clients weren't ready. (Neither was I.)*
- `15→` Bought, fixed & resold €30k+ of electronics. *Still my most reliable income source.*
- `20→` Co-founded Orbitvu Nordic. DKK 500k → 3.5M. *Learned more than any class.*

### Hosting

GitHub Pages, branch `gh-pages` or root of `main`. No build step — pure HTML/CSS/JS. Custom domain optional (out of scope for v1).

---

## 2. EU Grid Dashboard (`jskifterj/eu-grid-dashboard`)

### Architecture

```
ENTSO-E Transparency Platform API (free, requires registration)
    └── Python FastAPI backend
            ├── GET /api/generation?country=DE    — current generation mix by source
            ├── GET /api/prices?country=DE         — day-ahead prices, 24h
            ├── GET /api/flows?country=DE          — cross-border import/export flows
            ├── GET /api/overview                  — all-countries CO₂ + renewable summary (for map)
            └── GET /api/summary?country=DE        — Claude API briefing (cached 15 min)
    └── Vanilla JS frontend + Chart.js + D3.js (v7 + world-atlas)
            ├── Interactive Europe map
            ├── Stats strip
            ├── AI briefing card
            └── Four charts
```

Backend deployed on **Render** (free tier, Python). Frontend served as static files from the same Render service or from the portfolio's GitHub Pages.

### Frontend — Map

- Real geographic country shapes via D3.js + `world-atlas@2/countries-50m.json` (TopoJSON)
- Mercator projection centred on Europe (`center: [14, 54]`)
- Countries coloured by CO₂ intensity: green (`#1e4a2a`) → amber → red (`#4a1a1a`)
- Flow arrows (Bezier curves) between major country pairs; blue = import into selected country, orange = export
- **Denmark default selection** with pulsing green ring pin and callout bubble: *"🏠 that's where I'm from 😄"*
- Click any country → updates side panel

### Frontend — Side Panel (per country)

- Country name + flag
- 4 mini-stats: renewable %, day-ahead price €/MWh, net import/export GW, CO₂ g/kWh
- Horizontal bar chart of generation by source (Wind, Solar, Nuclear, Hydro, Gas, Coal, Biomass, Other) with source-specific colours
- CO₂ intensity gauge (gradient bar, pointer at current value)
- Cross-border flows list (import = blue, export = orange)

### Frontend — Charts (main panel)

| Chart | Type | Data |
|-------|------|------|
| Generation mix — now | Donut (Chart.js) | Current gen by source for selected country |
| Day-ahead price — 24h | Line (Chart.js) | Hourly prices, current day |
| Renewable share — 7 day | Line (Chart.js) | Daily renewable % for past 7 days |
| Cross-border flows | Horizontal bar (Chart.js) | Neighbouring country flows, GW |

### Frontend — Stats Strip

Four KPIs above charts: renewable share %, day-ahead price €/MWh, CO₂ intensity g/kWh, grid frequency Hz. Deltas vs yesterday shown in green (improvement) or red.

### AI Briefing Card

- Sits at top of dashboard, below nav
- Green left border, ✦ icon, label "AI Grid Briefing · {Country} · {time}"
- One paragraph plain English summary of current grid conditions
- Generated by Claude API (`claude-haiku-4-5-20251001` for cost efficiency)
- Prompt bundles: current generation mix, price, net flows, CO₂ intensity, 7-day trend direction
- Cached server-side for 15 minutes per country; refreshed on cache expiry, not on every page load
- Falls back gracefully if API key absent or rate limited

### Backend — ENTSO-E Integration

- Register for free ENTSO-E API key at `transparency.entsoe.eu`
- Use `entsoe-py` Python library (wraps the XML API cleanly)
- Cache responses in-memory (15 min TTL) to avoid rate limits and keep Render free tier alive
- Key data points fetched:
  - `query_generation()` — actual generation per type
  - `query_day_ahead_prices()` — DA prices
  - `query_crossborder_flows()` — physical flows between bidding zones
- CO₂ intensity derived from generation mix using standard emission factors (g/kWh per source)
- Grid frequency from ENTSO-E or hardcoded as 50.01 Hz nominal (real-time frequency requires separate feed; use nominal + small simulated jitter for v1)

### Countries Supported (v1)

DE, FR, GB, NO, SE, DK, FI, CH, AT, NL, BE, PL, ES, IT, CZ, PT, RO, GR, IE, HU

### Environment Variables

```
ENTSO_API_KEY=...
ANTHROPIC_API_KEY=...
```

### Deployment

- Render: free web service, Python 3.11, `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
- `render.yaml` for one-click deploy
- Frontend static files served from FastAPI's `StaticFiles` mount

---

## 3. Repo Structure

### `jskifterj/portfolio`
```
index.html          # single-page portfolio
style.css           # all styles
script.js           # smooth scroll, modal deep-dives
assets/             # profile photo, favicon
README.md
```

### `jskifterj/eu-grid-dashboard`
```
app/
  main.py           # FastAPI app, routes
  entso.py          # ENTSO-E data fetching + caching
  ai.py             # Claude API briefing generation
  models.py         # Pydantic response models
frontend/
  index.html        # dashboard shell
  map.js            # D3 Europe map
  charts.js         # Chart.js charts
  api.js            # fetch wrappers
  style.css
requirements.txt
render.yaml
.env.example
README.md
```

---

## 4. Out of Scope (v1)

- Real-time grid frequency (nominal + jitter used instead)
- Historical data beyond 7 days
- Mobile-optimised layout (desktop-first for now)
- Authentication or user accounts
- Custom domain

---

## 5. Success Criteria

- Portfolio site loads at `jskifterj.github.io` with no build step
- Dashboard is live on Render with real ENTSO-E data for at least DE, FR, DK, NO
- AI briefing generates a coherent paragraph for any supported country
- Denmark easter egg visible on map load
- Both repos public on GitHub
- Can be explained and demoed in a 5-minute interview slot
