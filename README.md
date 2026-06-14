# Gate.io Live Stats — backend service

Private **FastAPI + PostgreSQL + Node SSR** service for Gate.io spot trade ingest, live WebSocket fan-out, and SSR data for WordPress SEO pages.

**Private code:** [logicencoder/gate-live-stats-backend](https://github.com/logicencoder/gate-live-stats-backend)

**Product overview (portfolio):** [gate-live-stats-plugin-overview](https://github.com/logicencoder/gate-live-stats-plugin-overview) — live dashboard at [logicencoder.com/gate-app/](https://logicencoder.com/gate-app/)

## Service role

- **`gate_td_app.py`** — Gate.io WS ingest → PostgreSQL `gateio_trades`, REST + `/ws`, operator dashboard
- **`gate_td_app_ssr.js`** — React SSR per coin (`/ssr_m1/{symbol}`, `/coin/:symbol`)
- Public tunnel: `wss://ws2.logicencoder.com/ws`

See [REPOS.md](REPOS.md).

---

**Made by [Logic Encoder](https://logicencoder.com)** · [GitHub](https://github.com/logicencoder) · [Contact](https://logicencoder.com/contact/)
