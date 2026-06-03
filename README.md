# Gate.io Live Trading Statistics — Backend (Public Overview)

Public description of the **private** FastAPI service that ingests Gate.io spot trades, stores them in PostgreSQL, and streams realtime stats to WordPress and the ops dashboard.

**WebSocket / API:** [ws2.logicencoder.com](https://ws2.logicencoder.com)  
**Private implementation:** [gate-live-stats-backend](https://github.com/logicencoder/gate-live-stats-backend)

---

## What it does

- Subscribes to Gate.io spot trade WebSockets per USDT pair
- Persists trades to PostgreSQL (`gateio_trades`)
- Broadcasts live stats and trades to dashboard clients and public WS consumers
- Exposes REST: `/api/stats`, `/api/coins`, `/api/dead-symbols`, `/api/reload-symbols`, SSR via Node on port 3001
- Ops UI at `/dashboard` (queue, subscriptions, Gate.io connection health)

---

## Deployment (LogicEncoder)

| Service | Host | Port |
|---------|------|------|
| gate-live-stats-python | SOL | 8083 |
| gate-live-stats-ssr | SOL | 3001 |
| Cloudflare tunnel | ws2.logicencoder.com | → 8083 |

Managed by Universal Service Manager on SOL.

Plugin overview: [gate-live-stats-plugin-overview](https://github.com/logicencoder/gate-live-stats-plugin-overview).
