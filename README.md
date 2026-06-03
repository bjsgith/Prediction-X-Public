# PredictionX

**TL;DR** — A live trading system that prices binary commodity contracts on Kalshi with a Black-76 futures model, ranks them by edge, and lets an autonomous agent place real Kelly-sized limit orders. Scope is **energy + metals** (WTI, Brent, natural gas, gold, silver, copper). Shadow by default; `--live` arms real RSA-signed execution. The portfolio is tracked through a phone-first PWA.

> A live quantitative trading system for commodity prediction markets — fair-value pricing, an autonomous strategy agent, and real RSA-signed order execution on Kalshi.

**PredictionX prices binary commodity contracts on [Kalshi](https://kalshi.com) against a Black-76 futures model, ranks the market by edge, and executes against it.** The agent places real RSA-signed limit orders, sized by fractional Kelly on cash-on-hand. All fills are reconciled, every contract is scored against actual settlement, and the portfolio is surfaced through a phone-first dashboard.

Coverage is deliberately narrow: **energy and metals only** — WTI crude, Brent crude, natural gas, gold, silver, copper. These are the asset classes in which a futures-curve model holds a structural informational advantage over the prediction-market mid.

> **About this repository.** This is a public **showcase** of a personal quantitative-trading project. The implementation lives in a separate private repository; this repo presents the design, methodology, and results. It is meant to be read, not run.

## Documentation

| | |
|---|---|
| ▤ **Live Dashboard** | [predictionx-zeta.vercel.app](https://predictionx-zeta.vercel.app) |
| ∑ **Pricing & Math** | Black-76 derivation, Kelly sizing, fee adjustments → [`docs/math.html`](docs/math.html) |
| ◷ **Shadow Testing Results** | 6 weeks of paper-trading analysis → [`docs/shadow_showcase.html`](docs/shadow_showcase.html) |
| ▢ **Mobile PWA** | Phone-first dashboard architecture & API → [`mobile/README.md`](mobile/README.md) |

## Engineering highlights

- **Real, signed execution.** Live RSA-PSS/SHA-256 request signing against the Kalshi v2 API places limit orders on a real account — not a backtest harness.
- **First-principles pricing.** Binary contracts are valued as cash-or-nothing digital options via a closed-form Black-76 `N(d₂)` model fit to the futures strip, with regime-aware realized volatility.
- **Settlement-source correctness.** Gold and silver are routed through Pyth Network spot (`XAU/USD`, `XAG/USD`) — the actual Kalshi settlement source — rather than COMEX futures, a distinction that flips real resolutions.
- **Self-calibrating learning loop.** The agent debiases its own edge with an OLS confidence regression on its resolved track record, and a periodic Claude "fund-manager review" writes runtime overrides — no restart, no redeploy.
- **Risk-aware sizing.** Fractional Kelly on cash-on-hand (not total bankroll), EV-ordered, behind layered safety guards (stale-data rejection, implausible-edge gate, per-series and portfolio exposure caps).
- **Honest measurement.** Every decision is scored against actual settlement with Brier Skill Score *relative to the market mid* as the benchmark — a strictly proper test of whether the model beats the market it trades against.
- **Full test isolation.** Every external API (Kalshi, yfinance, Pyth, Anthropic) is mocked; no test issues a live network call.

---

## Problem Statement

Kalshi offers contracts of the form *"Will WTI crude settle above $80 at month-end?"* — each paying \$1 if the condition holds, \$0 otherwise. Economically, each constitutes a **cash-or-nothing digital option** on the underlying futures price. The hypothesis: Kalshi-quoted implied probabilities systematically deviate from fair value, because most order flow is not conditioned on the futures curve.

1. **Fetch** the live Kalshi orderbook — YES and NO sides independently (each maintains its own book; the two sides are not complements).
2. **Build** the underlying price series — the futures strip (Yahoo Finance) for all six series; for gold and silver, the front-of-strip price is replaced with the live Pyth Network Hermes spot (`XAU/USD`, `XAG/USD`), as Kalshi settles those contracts against Pyth rather than COMEX.
3. **Fit** a log-normal distribution at expiry using the log-interpolated forward and a regime-aware realized volatility estimate — 252-day daily for normal-dated contracts, 5-day hourly for sub-week.
4. **Integrate** the closed-form `N(d₂)` digital-option formula to obtain a fair probability.
5. **Compare** fair value against the Kalshi mid — that spread constitutes the **edge**.
6. **Trade** it: the agent debiases raw edge through an OLS confidence regression fitted on its own resolved track record (restricted to post-Pyth-alignment runs), sizes with fractional Kelly on cash-on-hand, and submits a signed limit order in live mode.
7. **Score** all decisions against actual settlements using Brier score and Brier Skill Score relative to the market mid as the benchmark forecast.

A periodic **fund-manager review** (Claude) reads the accumulated track record, identifies structurally under- or out-performing segments, and writes runtime overrides that recalibrate the agent's confidence — a learning loop that tightens calibration as resolutions accumulate.

---

## Order Execution and Settlement

| | |
|---|---|
| **Authentication** | Kalshi v2 RSA request signing — `RSA-PSS / SHA-256`, base64, timestamp-bound. The private key resides on the VPS; only the public key is registered with Kalshi. |
| **Orders** | Limit only (1–99¢). A stale or erroneous signal rests unfilled rather than executing against the book. |
| **Sizing** | Half-Kelly, `f* = adjusted_edge / (1 − price)`, capped at 10% of bankroll, sized off **cash-on-hand**, funded in decreasing order of `f*` within each cycle. |
| **Reconciliation** | Pending orders are polled each cycle — `executed` orders record actual fill price and contract count (including fees); `cancelled`/`expired` entries are flagged and their reserved capital released. P&L settles on the real fill cost, not the decision-time mid. |
| **Bankroll** | The live Kelly bankroll is the real Kalshi balance, synced every cycle via `sync_live_balance()`. A capital-events ledger separates trading P&L from deposits and withdrawals. |
| **Kill switch** | Disarming live mode reverts the agent to shadow for all new cycles; open Kalshi orders are unaffected. |
| **Guardrails** | Stale futures data is rejected before any order is placed (hourly bars older than 6h, daily bars older than 96h). A price frozen at the same value across ≥5 observations within a 12h rolling window (spanning at least 6h) is also rejected. Raw edge ≥ 25pp is forced to `watch` (implausible-edge gate). Per-cycle exposure caps: 20% of account value per series, 40% portfolio-wide. Directional asymmetry cap: 3 same-direction buys per series per cycle. Orders are skipped when no valid ask is present. Settlement deposit deferral prevents double-counting of Kalshi payouts. Startup is refused without valid credentials, and the Kalshi balance is reconciled against the database at initialization. |

Live and shadow decisions are recorded in the same `strategy_decisions` table, distinguished by a `live_mode` flag, so the OLS confidence model is trained on the complete decision history.

---

## Methodological Considerations

- **Settlement-source correctness.** Kalshi's gold and silver Daily contracts settle on the **Pyth Network spot price** (`XAU/USD`, `XAG/USD`) — not COMEX GC/SI futures. On 2026-05-21, Pyth XAU = $4,543.07 fell *inside* the resolution bracket while COMEX GC stood *outside* it. The engine routes gold and silver through `data/pyth.py`; each series carries an explicit `settlement_kind` in the asset registry.
- **YES/NO independence.** `yes_mid + no_mid ≠ 1` in general. Each side is priced against its own independently-quoted mid. When the API omits NO prices, pricing the NO side explicitly (`--side no`) raises an error; pricing the YES side continues normally with `no_market_price=None`. `no_prices_from_api` is recorded in analytics.
- **Regime-aware volatility.** Long-dated contracts employ 252-day daily realized vol (`√252`); sub-week contracts employ 5-day hourly vol (`√(365×23)`), as daily vol is insufficiently informative at that horizon.
- **Edge debiasing with training cutoff.** The raw model edge is systematically biased. The agent regresses realized P&L on raw edge at the most specific segment available (`series + direction + bucket` → `series + bucket` → global → prior `(0, 0.5)`). A configurable `training_cutoff_date` restricts the regression to post-Pyth-alignment runs, preventing pre-correction data from distorting the fitted coefficients. The traded signal is `adjusted_edge = α + β · raw_edge`.
- **Kelly on cash-on-hand, EV-ordered.** Sizing Kelly off total bankroll while capital is locked in unresolved positions implicitly over-levers the book. PredictionX sizes off `bankroll − Σ open Kelly stakes` and funds the highest-`f*` candidates first each cycle.
- **Proper scoring, honestly.** Brier Skill Score is computed *against the Kalshi mid as the reference forecast*. BSS > 0 indicates the engine is a strictly superior probability estimator relative to the market it trades against. Calibration is reported in 10pp bins.

---

## Summary of Contributions

| | |
|---|---|
| **Live trading** | RSA-signed limit orders on a real Kalshi account, Kelly-sized, reconciled every cycle |
| **Coverage** | 6 active series — energy (WTI, Brent, natural gas) + metals (gold, silver, copper) |
| **Pricing** | Log-normal Black-76 futures model, closed-form `N(d₂)` digital-option formula |
| **Settlement sources** | Futures strip (Yahoo Finance) for WTI/Brent/natural gas/copper; Pyth Hermes spot for gold/silver |
| **Volatility** | 252-day daily realized vol; 5-day hourly vol for sub-week contracts |
| **YES/NO independence** | Each side priced against its own independently-quoted mid; NO side requires API quote (no arb-consistent fallback) |
| **Decay curve** | Full `P(target)` path from today to expiry with a deterministic 0/1 endpoint |
| **Scenarios** | Bull / base / bear under ±1σ forward shocks |
| **Edge debiasing** | OLS regression of realized PnL on raw edge, segment-hierarchical, training-cutoff configurable, runtime-overridable |
| **Sizing** | Fractional Kelly on cash-on-hand, EV-ordered |
| **Safety guards** | Frozen-data rejection, 25pp implausible-edge gate, per-cycle exposure caps (20% series / 40% portfolio), settlement deposit deferral |
| **Learning loop** | Periodic Claude fund-manager review → `segment_overrides` honored at runtime, no restart |
| **Scoring** | Brier, Brier Skill Score vs. market, calibration buckets, edge accuracy, per-DTE breakdown |
| **Persistence** | SQLite with 30+ analytics views — runs, decay curves, resolutions, decisions, live orders, capital events, bankroll, LLM spend, reviews |
| **Interfaces** | Rich terminal tables, JSON export, local Streamlit dashboard, phone-first PWA on Vercel |

---

## Instrument Universe

Scope is constrained to asset classes in which a futures-curve or Pyth-spot model carries an informational advantage. Crypto, equity-index, and agricultural series are deliberately excluded.

| Asset class | Markets | Futures root | Active Kalshi series | Settlement source |
|---|---|---|---|---|
| Energy | WTI crude | `CL` | `KXWTI` | Futures (Yahoo Finance) |
| Energy | Brent crude (daily) | `BZ` | `KXBRENTD` | Futures (Yahoo Finance) |
| Energy | Natural gas (daily) | `NG` | `KXNATGASD` | Futures (Yahoo Finance) |
| Metals | Gold (daily) | `GC` | `KXGOLDD` | Pyth Hermes (`XAU/USD`) |
| Metals | Silver (daily) | `SI` | `KXSILVERD` | Pyth Hermes (`XAG/USD`) |
| Metals | Copper (daily) | `HG` | `KXCOPPERD` | Futures (Yahoo Finance) |

---

## Capabilities

A single `pricer` CLI (Python 3.11+) drives the system — the surface area at a glance:

| Capability | What it does |
|---|---|
| **Browse** | Lists open Kalshi contracts across all six series — tickers, liquidity, expiries. |
| **Price** | Values a single contract: fair value vs. market, forward price, annualized vol, ±1σ scenarios, and a full time-decay curve. |
| **Scan** | Prices the whole market in parallel and ranks it by edge, with volume / mid / DTE filters. |
| **Agent** | Runs the autonomous `reconcile → scan → price → classify → size → execute` loop in shadow or live mode. |
| **Auto-resolve** | Pulls settlement outcomes from the Kalshi API for expired contracts and records realized P&L. |
| **Performance** | Scores the model against actual settlements — Brier, Brier Skill Score vs. the market mid, calibration bins, edge accuracy, per-DTE breakdown. |
| **Review** | Triggers the periodic Claude fund-manager review that recalibrates the agent via runtime segment overrides. |
| **Dashboards** | A local Streamlit analytics view and a deployed phone-first PWA over the live portfolio. |

Pricing and scanning run against Kalshi's unauthenticated public market data; the learning loop and live execution add Claude and RSA-signed Kalshi credentials respectively. Shadow and live runs write to separate SQLite databases, distinguished by a `live_mode` flag so the calibration model trains on the complete decision history.

---

## Illustrative Output

```
─────────────────────────────────────────────────────────────────────────────
 KXWTI-26MAY-T65   YES   ABOVE 65.00   exp 2026-05-30   DTE 45
 Will the WTI front-month settle above $65.00 on 2026-05-30?
─────────────────────────────────────────────────────────────────────────────

   Metric                Value
   ────────────────────  ─────────
   Fair Value            62.4%
   Market Price          58.0%
   Edge                  +4.4%      ← engine sees YES as underpriced
   Forward Price         $67.83
   Annual Vol            34.2%

   Scenarios (±1 SD forward shock)
   ┌───────────────┬────────┬───────────────┐
   │ Bull (+1 SD)  │  Base  │  Bear (-1 SD) │
   │    78.1%      │ 62.4%  │     44.7%     │
   └───────────────┴────────┴───────────────┘

   Time-decay curve (DTE 1 → 45):
   :▁▂▃▄▅▆▇█▇▆
   DTE 1: 72.3%   ...   DTE 45: 62.4%

 Priced at 2026-04-15 14:22:11 UTC   ·   saved to pricer.db (run #4218)
```

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                  KALSHI API  (public reads + signed writes)              │
│  ├── GET  /markets?series=…              list open contracts             │
│  ├── GET  /markets/{ticker}              YES+NO bid/ask, expiry, title   │
│  ├── GET  /markets/{ticker} (expired)    settlement YES/NO + price       │
│  ├── POST /portfolio/orders   [signed]   place limit order (live)        │
│  └── GET  /portfolio/balance  [signed]   account balance (live)          │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    DATA INGESTION  (data/)                               │
│  kalshi.py        — parse target, direction, series, expiry; RSA signing │
│  futures.py       — yfinance strip across liquid delivery months         │
│                     (all 6 series; Pyth anchors front price for Au/Ag)   │
│  pyth.py          — Pyth Hermes spot anchor: XAU/USD, XAG/USD           │
│                     (replaces front-of-strip price for gold/silver)      │
│  volatility.py    — 252d daily / 5d hourly realized vol                  │
│  calendar.py      — DTE (int days) and dte_hours (float hours to expiry) │
│  commodity_config — series → settlement_kind, futures root, events       │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    PRICING CORE  (pricing/)                              │
│  distribution.py  — log-normal fit, P(target) via N(d₂)                 │
│  time_value.py    — full DTE → expiry decay curve                        │
│  aggregator.py    — combine p_base + p_tv + ±1σ scenarios                │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│              OUTPUT  (output/)  +  STORAGE  (storage/db.py)              │
│  Rich terminal table · JSON export · SQLite + 30+ analytics views        │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    AUTONOMOUS AGENT  (agent/)                            │
│  Every cycle:                                                            │
│    reconcile live orders → scan → filter → price                         │
│      → OLS confidence regression (training_cutoff_date applied)          │
│      → classify (watch/auto-buy/clear-buy)                               │
│      → safety guards (frozen data, edge sanity, exposure caps)           │
│      → Kelly sizing (cash-on-hand, EV-ordered)                           │
│      → [live] sign + POST limit order for Kelly buys                     │
│      → journal to strategy_decisions + heartbeat                         │
│  Periodically (every 10h or N resolutions): fund-manager review → segment_overrides │
└──────────────────────────────────────────────────────────────────────────┘
```

The mathematical treatment — `d₂` derivation, vol regimes, time-value decay, scenario shocks, Brier scoring, OLS confidence model, Kelly sizing, and the override mechanism — is rendered with MathJax at **[`docs/math.html`](docs/math.html)**.

---

## Autonomous Trading Agent

**Signal pipeline.** For every contract that clears the pre-pricing filters (sufficient liquidity, mid within the priceable band, within the active TTX/DTE window, spread sufficiently tight), the agent prices both sides, computes `raw_edge = fair_value − market_price`, then retrieves `(α, β)` from an OLS regression of historical realized P&L on raw edge at the most specific segment available — `series + direction + bucket` → `series + bucket` → `global + bucket` → `global` → prior `(0, 0.5)`. The regression is restricted to runs at or after `training_cutoff_date` (if set). The traded signal is `adjusted_edge = α + β · raw_edge`. Classification: `< θ` → **watch** (zero size, journaled) · `[θ, 2θ)` → **auto-buy** · `≥ 2θ` → **clear-buy**.

**Sizing.** Half-Kelly on the binary formula, capped at 10% of bankroll, sized off cash-on-hand, funded in decreasing order of `f*` within the cycle.

**Safety guards.** Before any live order: *(1)* stale or frozen futures data is rejected before pricing proceeds; *(2)* raw edge ≥ 25pp → `watch` with reason `raw_edge_implausible`; *(3)* insufficient live cash for the current cycle; *(4)* directional asymmetry — at most 3 buys in the same `(series, direction)` per cycle; *(5)* series exposure cap — execution is blocked if it would push that series beyond 20% of account value; *(6)* portfolio exposure cap — execution is blocked if total deployed capital would exceed 40% of account value. The absence of a valid ask results in a skip rather than a 1¢ resting bid.

**Live order hook.** When armed (`--live`) and a Kelly decision is classified as a buy, the agent derives a limit price from the side's ask, computes a contract count from the Kelly dollar size, signs and POSTs the order, and records the order id with `order_status='pending'`. Placement failure downgrades the decision to skip with an explicit failure reason — no silent partial state. Kalshi's terminal fill status is `executed`; contract count is taken from `fill_count_fp` and total fill cost from `taker_fill_cost_dollars`, `maker_fill_cost_dollars`, `taker_fees_dollars`, and `maker_fees_dollars`. P&L settles on the real execution price.

**Fund-manager review.** Every 10 hours (or when sufficient new resolutions accumulate), the agent loads its analytics, identifies under- and out-performing segments, calls `claude-sonnet-4-6`, and writes `segment_overrides` (`confidence_multiplier` / `exclude` / `min_edge_override`) to a table the signal pipeline reads at runtime. A silent review never clears an existing active override set. No restart is required; this is the only active LLM path in the system.

---

## Portfolio Monitoring Interface

A phone-first PWA on Vercel surfaces the live portfolio. All requests pass through a Vercel serverless proxy that injects the `DASHBOARD_API_KEY` before forwarding to a read-only FastAPI service on the VPS, which queries `pricer_live.db` in `mode=ro`. No credentials are transmitted to the client.

The dashboard exposes only the **live Kelly book** (`live_mode = 1`), organized as six tabs:

- **Portfolio** — current NAV, all-time P&L, return-on-staked, NAV chart.
- **Positions** — open contracts with order-status badges (pending / filled), fill price, contract count, current Kalshi mid, unrealized P&L.
- **Activity** — merged feed of fills, settlements, and closed decisions with realized P&L per trade. Individual decisions are tappable, opening a price-time snapshot and a Black-76 binary P&L-versus-futures-price curve.
- **Analytics** — win record, edge calibration (realized hit rate by edge bucket), fill quality relative to the decision-time mid, per-series P&L breakdown.
- **Model** — pricing-model calibration over the live era: the OLS fit of realized P&L on raw edge (global and per series), Brier-based scoring (BSS, MAE, bias), and whether the model beats the market.
- **Agent** — heartbeat and health status, the full untruncated periodic self-review rendered as prose, active segment overrides, and LLM-engine status.

An always-visible **agent-health pill** reports the agent's state — Live, Shadow, Sleeping (no heartbeat within 15 minutes), or Offline (none within 60 minutes) — and navigates to the Agent tab when tapped. Setup and Cloudflare configuration are documented in **[`mobile/README.md`](mobile/README.md)**.

---

## Operational Architecture

Three independent runtime components. Shadow and live operate against **separate SQLite databases** — `pricer.db` (shadow) and `pricer_live.db` (live) — so paper and real decisions never share state while still feeding one calibration model.

```
┌──────────────────────────────── VPS ────────────────────────────────┐
│                                                                     │
│   agent.service                    predictionx-dashboard.service    │
│   writes pricer_live.db            reads pricer_live.db (mode=ro)   │
│   ANTHROPIC_API_KEY                DASHBOARD_API_KEY                │
│   KALSHI_API_KEY + private .pem    ↑ port 8000                      │
│   AGENT_LIVE_MODE=true             DB_PATH=pricer_live.db           │
│   DB_PATH=pricer_live.db                                            │
└────────────────────────────────────┼────────────────────────────────┘
                                     │
                       Caddy (HTTPS) + Cloudflare
                                     │
                          ┌──────────▼──────────┐
                          │   Vercel proxy      │
                          │   API_ORIGIN +      │
                          │   DASHBOARD_API_KEY │
                          └──────────┬──────────┘
                                     │
                          ┌──────────▼──────────┐
                          │     iPhone PWA      │
                          └─────────────────────┘
```

The agent runs 24/7 as a systemd service — the sole writer and the only holder of Kalshi credentials. The FastAPI service is read-only. The PWA neither accesses the database directly nor receives any upstream credentials — a strict separation of write authority, read access, and presentation. The mobile dashboard's architecture is described in **[`mobile/README.md`](mobile/README.md)**.

---

## Repository Structure

> **Note.** This is the **public documentation repository**. The source code — the `pricer` package, the agent, the mobile API/PWA, tests, and deploy units — is maintained in a separate **private** repository and is not published here. The tree below describes that private repository; files such as `agent/config.py`, `deploy/setup.sh`, and `mobile/CLAUDE.md` referenced throughout these docs live there. Published here are `README.md`, `mobile/README.md`, and `docs/`.

```
PredictionX/
├── pricer.py                 # price() entry point
├── cli.py                    # typer CLI: list, price, scan, history, resolve,
│                             #   auto-resolve, performance, agent, serve
├── models.py                 # KalshiContract, Position, PricingResult
├── exceptions.py             # typed exceptions (OrderPlacementError, PythAPIError, …)
├── dashboard.py              # local Streamlit dashboard (dev only)
│
├── data/                     # ingestion — Kalshi (+ RSA signing), futures, Pyth, vol
│   ├── kalshi.py             #   public + RSA-signed client, settlement symbol parsing
│   ├── futures.py            #   yfinance strip, staleness + frozen-price guards
│   ├── pyth.py               #   Pyth Hermes spot client (XAU/USD, XAG/USD)
│   ├── volatility.py         #   daily / hourly realized vol
│   ├── calendar.py           #   DTE, dte_hours, events
│   ├── commodity_config.py   #   asset registry: settlement_kind, futures root, events
│   └── ticker_utils.py       #   exchange suffixes
├── pricing/                  # log-normal distribution, time-value, aggregator
├── output/                   # rich terminal table, JSON export
├── storage/db.py             # SQLite persistence + 30+ analytics views
│
├── agent/                    # autonomous strategy agent
│   ├── agent.py              #   loop, safety guards, live order hook, reconciliation
│   ├── config.py             #   AgentConfig (env-var overridable, live_mode, training_cutoff_date)
│   ├── filters.py            #   pre-pricing filters
│   ├── signals.py            #   OLS regression, segment_overrides, training cutoff
│   ├── sizing.py             #   Kelly sizing
│   ├── llm.py                #   LLM stub (per-cycle veto removed)
│   └── review.py             #   periodic fund-manager learning loop
│
├── mobile/
│   ├── api/                  # FastAPI VPS service (predictionx-dashboard.service)
│   │   ├── main.py           #   /portfolio /positions /activity /trade /analytics /model /agent
│   │   ├── queries.py        #   read-only SQL, live_mode=1 scoped
│   │   ├── auth.py           #   X-API-Key middleware
│   │   └── settings.py       #   env loader
│   └── web/                  # Vercel-hosted PWA + serverless proxy
│
├── docs/
│   ├── math.html             # full mathematical reference (MathJax)
│   └── shadow_showcase.html  # generated shadow-validation one-pager (inline SVG, offline)
├── scripts/build_showcase.py # builds shadow_showcase.html from the paper-trade DB
├── deploy/                   # systemd units, Caddyfile, setup.sh
└── tests/                    # pytest suite, all external APIs mocked
```

---

## Test Coverage

Every external API — Kalshi (public and signed), yfinance, Pyth Hermes, Anthropic — is mocked; no test issues a live network call. RSA signing, the safety guards, live order reconciliation, and Pyth spot pricing each have dedicated test suites, alongside the read-only mobile API service.

---

## Reference Documentation

| Document | For |
|---|---|
| [README.md](README.md) | Project overview (this file) |
| [docs/math.html](docs/math.html) | Complete mathematical reference (MathJax) |
| [docs/shadow_showcase.html](docs/shadow_showcase.html) | Shadow-validation one-pager — the paper-trade evidence that armed live |
| [mobile/README.md](mobile/README.md) | Mobile PWA dashboard architecture |

---

## Risk Disclosure

PredictionX places **real orders with real capital** when run with `--live`. The live bankroll is intentionally small and the system is a personal research vehicle, not investment advice or a managed product. Edge scores are model outputs from a log-normal approximation of the underlying price dynamics; real markets exhibit jumps, fat tails, and mean-reversion the model does not fully capture. Kelly is the only strategy; all live order placement, sizing, and reconciliation operates on the Kelly book exclusively. Prediction-market trading carries substantial risk of loss and is not legal in all jurisdictions — you are responsible for compliance with local law and for any capital you put at risk.
