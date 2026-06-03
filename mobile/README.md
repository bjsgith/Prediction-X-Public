# PredictionX Mobile Dashboard

**TL;DR** — A read-only, iPhone-first PWA (Vercel) that surfaces the live Kalshi Kelly book from a FastAPI service on the VPS. Six tabs — Portfolio, Positions, Activity, Analytics, Model, Agent. The browser holds no secrets: a Vercel proxy injects the API key and the API reads `pricer_live.db` in `mode=ro`.

**Live deployment:** [predictionx-zeta.vercel.app](https://predictionx-zeta.vercel.app)

A read-only, iPhone-first PWA for tracking the **live Kalshi trading portfolio**, backed by a FastAPI service on the VPS.

The dashboard presents the **live Kelly book exclusively** (`live_mode = 1`). The paper Fixed shadow and pre-live shadow-Kelly history are not surfaced. Six tabs — **Portfolio**, **Positions**, **Activity**, **Analytics**, **Model**, **Agent** — are accessible, together with a persistent agent-health pill in the app bar that navigates to the Agent tab.

The client device does not contact the VPS directly. Every request is routed through a Vercel serverless proxy that injects the shared API key before forwarding to the FastAPI service, which queries `pricer_live.db` in read-only mode. No credentials are retained by the PWA.

---

## System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│  iPhone — Safari "Add to Home Screen"                                  │
│  • Service worker caches HTML/JS/CSS for offline open                  │
│  • All API requests go to /api/* on the same Vercel origin             │
└─────────────────────────────────┬──────────────────────────────────────┘
                                  │ HTTPS
                                  ▼
┌────────────────────────────────────────────────────────────────────────┐
│  Vercel — mobile/web/                                                  │
│  • Static PWA (index.html, app.js, chart.js, sw.js, styles.css)        │
│  • Serverless proxy: mobile/web/api/[...path].js                       │
│    – injects X-API-Key: $DASHBOARD_API_KEY                             │
│    – forwards to $API_ORIGIN/<path>                                    │
│    – 8-second timeout, never caches                                    │
│  ENV: API_ORIGIN, DASHBOARD_API_KEY  (server-only, never exposed)      │
└─────────────────────────────────┬──────────────────────────────────────┘
                                  │ HTTPS
                                  ▼
                       ┌──────────────────────┐
                       │  Cloudflare           │
                       │  proxy + WAF + SSL    │
                       └──────────┬───────────┘
                                  │
                                  ▼
┌────────────────────────────────────────────────────────────────────────┐
│  VPS — Caddy (HTTPS termination) → FastAPI (dashboard-api.service)     │
│                                                                        │
│  mobile/api/main.py     — routes + CORS + key auth                     │
│  mobile/api/auth.py     — X-API-Key middleware                         │
│  mobile/api/queries.py  — read-only SQL against pricer_live.db         │
│  mobile/api/settings.py — env loader                                   │
│                                                                        │
│  pricer_live.db opened in mode=ro — no schema changes, no writes ever  │
└────────────────────────────────────────────────────────────────────────┘
```

The agent (`agent.service`) is the sole writer to `pricer_live.db`. The dashboard API is strictly read-only. The PWA holds no credentials.

---

## API Reference

All routes except `/health` require the `X-API-Key` header. Every authenticated route is scoped to `live_mode = 1` — no query parameters, no strategy selection.

| Route | Auth | Purpose |
|---|---|---|
| `GET /health` | none | Liveness probe — returns `{status, db_mtime}`. |
| `GET /portfolio` | yes | Portfolio value, cash balance, realized & 24h P&L, total return, win rate, NAV curve, and `capital_events` ledger. |
| `GET /positions` | yes | Open live positions with order status, fill price, contracts, cost basis, edge, and time-to-expiry. |
| `GET /activity` | yes | Merged chronological feed of live buys and resolution events. |
| `GET /trade/{decision_id}` | yes | Price-time snapshot of one live decision (the data behind `agent trade [id]`) plus a Black-76 binary P&L-vs-futures-price curve. `404` if the id is not a live-book decision. |
| `GET /analytics` | yes | Win record, profit factor, edge calibration, edge-bucket performance, per-series P&L, and fill-rate/slippage metrics. |
| `GET /model` | yes | Pricing-model calibration over the live era: OLS fit of realized P&L on raw edge (global and per series) plus Brier-based scoring (BSS, MAE, bias) and whether the model beats the market. |
| `GET /agent` | yes | Agent heartbeat and health status (`online`/`stale`/`offline`), the **full untruncated** latest periodic self-review (`llm_summary`, rendered as prose in the Agent tab), active segment overrides, and LLM-engine status. |

Responses are UTC-normalized with a `Z` suffix. When live trading has not yet executed, `/portfolio` returns `account.live = false` and the PWA renders an "awaiting first live cycle" state.

---

## Configuration Model

Secrets live entirely on the server side; the browser holds none. The VPS env file carries `DASHBOARD_API_KEY` (the shared secret the FastAPI service requires as `X-API-Key`), `ALLOWED_ORIGIN` (the CORS allow-list, pinned to the Vercel domain), and `DB_PATH` (the read-only path to `pricer_live.db`, shared with the agent). The Vercel deployment carries the same `DASHBOARD_API_KEY` plus `API_ORIGIN` (the upstream FastAPI URL) — both server-only and never exposed to the client.

The result: inspecting the PWA in browser developer tools reveals no `X-API-Key`, no `API_ORIGIN`, and no Kalshi credentials. The proxy is the sole component that holds the shared secret in memory.

---

## The Experience

The PWA launches full-screen from the iPhone home screen. A service worker caches the static shell for offline open but never caches API responses — `Cache-Control: no-store` from the proxy enforces freshness. Navigation is a six-tab bottom bar — **Portfolio**, **Positions**, **Activity**, **Analytics**, **Model**, **Agent** — with the active tab persisted across launches. The agent-health pill in the app bar is colour-coded (green = Live, blue = Shadow, amber = Sleeping, red = Offline) and, when a heartbeat is present, shows a live "Next run" countdown; tapping it opens the **Agent** tab with the full self-review narrative, operational vitals, active overrides, and review-engine status.

---

## Security Model

- **The PWA holds no secrets.** No API key, no upstream URL, and no Kalshi credentials are present in the browser.
- **`DASHBOARD_API_KEY` is server-side only.** The proxy reads it via `process.env`; Vercel does not expose it to the client under any circumstances.
- **The FastAPI service is read-only by design.** It opens `pricer_live.db` with `mode=ro` in the SQLite URI — even a successful SQL injection cannot mutate state. All queries are fully static SQL strings with no user-supplied parameters, eliminating the injection surface entirely.
- **CORS is restricted** to `ALLOWED_ORIGIN`. The browser rejects any cross-origin attempt to call the API directly.
- **No write surface.** The dashboard API exposes only `GET` routes; the proxy rejects non-`GET` methods; CORS permits only `GET`.
- **Defense in depth:** bypassing the Cloudflare WAF and the proxy still requires the shared key to receive any response other than 401, and even with the key an attacker is confined to read access.

---

## Test Coverage

The API test suite mocks the SQLite connection, so no test touches a live database or network. The PWA has no dedicated suite — the serverless proxy is a ~30-line passthrough.

---

## Repository Structure

```
mobile/
├── CLAUDE.md                   # dashboard-specific conventions (scoping, tokens, render map)
├── api/                        # FastAPI service — runs on VPS
│   ├── main.py                 #   routes, CORS, app initialization
│   ├── auth.py                 #   X-API-Key middleware
│   ├── queries.py              #   read-only SQL queries and view-backed reads
│   ├── settings.py             #   env loader
│   ├── requirements.txt        #   fastapi, uvicorn
│   └── tests/                  #   pytest suite (mocked SQLite)
└── web/                        # PWA — deployed to Vercel
    ├── index.html              #   tabbed shell (Portfolio/Positions/Activity/Analytics/Model/Agent)
    ├── app.js                  #   tab routing, render logic, agent pill + tab, polling
    ├── chart.js                #   Chart.js wrappers — NAV curve, series P&L, calibration, model OLS, trade P&L
    ├── styles.css              #   theme tokens, tab layout, components
    ├── sw.js                   #   service worker — offline cache
    ├── manifest.webmanifest    #   PWA manifest (name, icons, theme color)
    ├── vercel.json             #   Vercel routing + security headers
    ├── icons/                  #   home-screen icons
    └── api/
        └── [...path].js        #   serverless proxy → VPS API
```

---

## Reference Documentation

- [README.md](../README.md) — project overview
- [docs/math.html](../docs/math.html) — mathematical reference
