# Range Finder ⚡ — Pionex Grid Bot Dashboard

Streamlit + Plotly dashboard that pulls live crypto market data, runs 15+ technical indicators, scores grid-bot suitability, predicts regime transitions, and monitors active Pionex bots.

## Three views

**Range Finder** — scores each pair 0–10 for grid-bot suitability using lagging indicators (ADX, BB, CVD, POC, RSI, funding). Picks direction, derives range from ATR, recommends grid count.

**Signal Scanner** — predictive system using 6 leading indicators to detect regime transitions *before* they happen. Setup Score 0–10 with urgency ranking, ETA, and cross-reference to grid score. Anticipates good entries before the market shifts.

**Bot Monitor** — connects to Pionex API (read-only) to show active spot grid bots. Cross-references live P&L, range position, and market conditions to generate HOLD / CLOSE / TAKE PROFIT / WARNING alerts.

## Files

| File | Role |
|---|---|
| `app.py` | Streamlit UI — page router, Range Finder cards, spot trade setup |
| `config.py` | CFG, GRID_CONFIG, SIGNAL_CFG, BOT_MONITOR_CFG, DEFAULT_PAIRS, LEGENDS |
| `indicators.py` | 15+ indicator calculations (RSI, ATR, EMA, POC/AVWAP, CVD, ADX, MACD, BB, OBV, Donchian, squeeze, etc.) |
| `grid_calculator.py` | Range / direction / mode / viability / score / profit estimation |
| `signal_engine.py` | 6 leading indicator detectors + scorers + Setup Score aggregator |
| `signal_scanner.py` | Streamlit UI for Signal Scanner — urgency table, detail cards, charts |
| `pionex_client.py` | Pionex Bot API client — HMAC auth, list bots, get details (read-only) |
| `bot_advisor.py` | Bot health assessment — price position, trend, P&L, duration → alerts |
| `bot_monitor.py` | Streamlit UI for Bot Monitor — portfolio summary, bot cards, range gauge |
| `data_fetcher.py` | CCXT wrapper — OKX/Bybit/Binance fallback (klines, OI, funding) |
| `trade_logger.py` | SQLAlchemy models `MetricsCache` + `Trade` |
| `refresh_data.py` | Cron entry — fills MetricsCache with metrics + grid score + signal score |
| `phases/` | Phase 2/3 stubs |

## Local run

```bash
python -m venv .venv
. .venv/Scripts/activate        # Windows bash — use source .venv/bin/activate on mac/linux
pip install -r requirements.txt

cp .env.example .env             # then edit as needed (public endpoints work without keys)

python -m refresh_data           # one-shot fill the SQLite cache
streamlit run app.py             # http://localhost:8501
```

### Python version

Tested on 3.12. If you are on 3.14 and `pandas_ta` refuses to install, you can remove it from `requirements.txt` — this codebase does not import it (all math is ported directly from JS).

## Project structure

```
range-finder/
├─ app.py                  # Streamlit UI — page router
├─ config.py               # All thresholds and configuration
├─ indicators.py           # 15+ technical indicator calculations
├─ grid_calculator.py      # Grid scoring, range, direction, viability
├─ signal_engine.py        # Signal Scanner — 6 leading indicator detectors
├─ signal_scanner.py       # Signal Scanner UI
├─ pionex_client.py        # Pionex API client (read-only)
├─ bot_advisor.py          # Bot health assessment engine
├─ bot_monitor.py          # Bot Monitor UI
├─ data_fetcher.py         # CCXT exchange data fetcher
├─ trade_logger.py         # SQLAlchemy models
├─ refresh_data.py         # Data refresh pipeline
├─ requirements.txt
├─ runtime.txt
├─ .streamlit/
│   ├─ config.toml
│   └─ secrets.toml        ← gitignored; add locally only
└─ phases/
    ├─ phase2_trade_logger.py
    └─ phase3_telegram.py
```

`.gitignore` already excludes `.env`, `*.db`, `.claude/`, caches.

## Streamlit Community Cloud deploy

1. Push repo to GitHub (public or private).
2. Go to [share.streamlit.io](https://share.streamlit.io) → **New app** → pick repo, branch `main`, main file `app.py`.
3. Under **Advanced settings → Secrets**, paste your secrets (same format as `.streamlit/secrets.toml`):
   ```toml
   BINANCE_API_KEY = ""
   BINANCE_API_SECRET = ""
   BYBIT_API_KEY = ""
   BYBIT_API_SECRET = ""
   PIONEX_API_KEY = "your_key"
   PIONEX_API_SECRET = "your_secret"
   PYONEX_LOG_LEVEL = "INFO"
   ```
   Exchange keys are optional (public endpoints work without them). Pionex keys need `Bot reading` permission for the Bot Monitor page.
4. Click **Deploy**. Streamlit Cloud installs `requirements.txt` and starts the app.

**SQLite note:** Streamlit Cloud has no persistent disk. The SQLite cache (`pyonex.db`) lives in the system temp dir and resets on each redeployment. Data refreshes automatically on each page load via `refresh_one()` — no separate cron needed.

**Secrets in code:** Modules read secrets via `st.secrets` (Streamlit Cloud primary) with `os.getenv()` fallback (local `.env`). No code changes required between environments.

## Environment variables

| Name | Purpose | Required |
|---|---|---|
| `BINANCE_API_KEY` / `BINANCE_API_SECRET` | higher rate limits on Binance public endpoints | no |
| `BYBIT_API_KEY` / `BYBIT_API_SECRET` | same for Bybit | no |
| `PIONEX_API_KEY` / `PIONEX_API_SECRET` | Bot Monitor — read-only bot access (`Bot reading` permission) | for Bot Monitor |
| `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` | Phase 3 alerts | no |
| `PYONEX_DB_PATH` | SQLite file path | defaults to `pyonex.db` |
| `PYONEX_LOG_LEVEL` | `INFO` / `DEBUG` | defaults to `INFO` |

## Phases

- **Phase 1 (done)** — indicators, grid calc, Streamlit dashboard, SQLite cache, Streamlit Cloud deploy
- **Phase 1.5 (done)** — Signal Scanner — predictive leading indicators, Setup Score 0–10
- **Phase 1.6 (done)** — Bot Monitor — Pionex API read-only, active bot health alerts
- **Phase 2** — "Log New Trade" + monitored-trade table, close recommendations
- **Phase 3** — Telegram alerts on STRONG SETUP / bot alert transitions

## Changelog

- **v1.6** — Bot Monitor: Pionex API integration, bot health assessment (HOLD/CLOSE/TP/WARNING), portfolio summary, range gauge
- **v1.5** — Signal Scanner: 6 leading indicators (CVD divergence, squeeze progression, structure transition, funding/OI, momentum divergence, volume exhaustion), urgency ranking, ETA estimation. Spot Trade Setup direction tightened with signal override.
- **v1.0** — Range Finder: Python port from JS engine. Donchian (20/55) + squeeze detector. OKX/Bybit/Binance fallback. Streamlit Cloud deploy.
