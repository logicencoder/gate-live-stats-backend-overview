# Gate.io Live Stats — backend service

Private **FastAPI + PostgreSQL + Node SSR** pipeline for [logicencoder.com/gate-app/](https://logicencoder.com/gate-app/). Ingests Gate.io spot trades, persists history, aggregates statistics, and serves live WebSocket plus SSR data for WordPress pretty URLs.

**Private code:** [logicencoder/gate-live-stats-backend](https://github.com/logicencoder/gate-live-stats-backend)

**Public product (portfolio):** [gate-live-stats-plugin-overview](https://github.com/logicencoder/gate-live-stats-plugin-overview)

The plugin renders the live dashboard and routes **`/gate/{SYMBOL}/`** SSR pages. This backend is ingest, storage, and API.

## Data flow

```
Gate.io spot WebSocket
        │
        ▼
gate_td_app.py — parse → asyncpg batch insert
        │
        ├── PostgreSQL `gateio_trades`
        ├── WebSocket /ws → plugin + visitors
        ├── REST stats → SSR + operator dashboard
        └── cache-warmup → wp-json/gate/v1/cache-warmup
```

Public live stream: **`wss://ws2.logicencoder.com/ws`** (Python default port **8083**)

## Python ingest (`gate_td_app.py`)

- Gate.io spot WebSocket with connection pooling and batched async inserts
- Pair discovery via **`gateio_usdt_pairs.py`**
- Stats cache and dead-symbol tracking — manual **`POST /api/reload-symbols`** (not auto-reload on every deploy)
- Same architectural pattern as MEXC backend: FastAPI REST + msgpack/WebSocket fan-out

## REST and WebSocket surface

| Path | Role |
|------|------|
| `/ws` | Live trade stream |
| `/ws-dashboard`, `/ws-dashboard-logs` | Operator monitoring |
| `/api/stats`, `/api/bootstrap/all`, `/api/trades` | Pair stats and history |
| `/api/coins`, `/api/dead-symbols` | Symbol management |
| `/api/orderbook`, `/api/orderbook/live` | Orderbook snapshots |
| `/api/monitoring`, `/api/metrics`, `/api/storage-stats` | Health |
| `/ssr_m1/{symbol}` | SSR data for WordPress + Node |

Outbound REST cache warmup to WordPress **`gate/v1`** namespace keeps plugin-side caches aligned after symbol list changes.

## Node SSR (`gate_td_app_ssr.js`)

Express + React (default port **3001**). Routes include **`/coin/:symbol`**, debug endpoints, and redirects to the WordPress hub. Canonical SEO URLs point to **`logicencoder.com/gate/{SYMBOL}/`** — plugin `template_redirect` serves SSR HTML with Schema.org JSON-LD.

Unlike MEXC, Gate SEO uses **live SSR** through WordPress routing rather than disk snapshot POSTs.

## Operator dashboard

**`/dashboard`** static UI — subscription counts, Gate connection health, queue metrics (same pattern as MEXC ops dashboard).

## Distinction from MEXC Live Stats

Shared design language (FastAPI + PostgreSQL + dual WS + SSR) but **separate exchange feed**, database table, tunnel hostname, and WordPress integration model (SSR vs snapshot files).

See [REPOS.md](REPOS.md).

---

**Made by [Logic Encoder](https://logicencoder.com)** · [GitHub](https://github.com/logicencoder) · [Contact](https://logicencoder.com/contact/)
