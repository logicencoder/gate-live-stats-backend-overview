# Gate Live Statistics — Market Data Service

Following several Gate.io markets is easier when the latest trades and the statistics around them sit in one place. Gate Live Statistics is a service that subscribes to Gate.io spot markets, stores every trade and order-book snapshot, and turns them into live and 24-hour statistics for each pair, which the companion WordPress plugin presents on logicencoder.com. It is built for people who watch trading activity on a pair and want to see who is trading, how large the trades are and whether the activity looks organic.

## Tech stack

| Layer | Technologies |
|---|---|
| Service | Python with FastAPI and asyncio, aiohttp WebSocket clients |
| Market data | Gate.io spot WebSocket API v4, trades and order books |
| Storage | PostgreSQL through asyncpg |
| Pages | A Node.js and Express server that renders pair pages with React on the server |
| Delivery | A WebSocket feed to the dashboard and to the WordPress plugin |

## Collecting the markets

The service reads the list of tracked pairs from a coin file and subscribes to each pair on Gate.io, up to 80 pairs per WebSocket connection, using as many connections as the list needs. Trades are stored as they arrive and order-book snapshots are taken every second by default; both are kept for 168 hours (seven days) unless the retention is changed, and older rows are deleted. A reload call applies a changed pair list and restarts the affected connections without stopping the service.

## Statistics for each pair

For every pair the service calculates first and last price, high and low, trade count, buy and sell volume in the coin and in USDT, the average trade size, the volume-weighted average price over 24 hours and an hourly volume series. Trades are split into small, medium and large by their USDT size, with the share of each, and the largest trade is reported with its size, side and time. The figures update continuously and are also available for history queries of recent trades and order-book snapshots.

## Trading pattern signals

Beside the plain statistics the service counts patterns that often point to automated trading: repeated identical trade sizes, round-lot sizes, bursts of trades and repeated time intervals. These combine into a bot score per pair with the share of each pattern. The score is an indicator derived from the observed trades and does not identify who traded.

## Pair pages and publishing

The server-rendered pair pages show the statistics to visitors and search engines, and a snapshot endpoint returns the data for one pair. When enabled, the service pushes structured data about each pair to WordPress in batches so that the plugin pages carry current schema. The live dashboard receives a push from the service every second while a client is connected.

## Coin list management

Operators can fetch coin information, add or remove pairs, list the backups of the coin file and revert to one, and prune pairs that have not traded. A status call shows, for each pair, whether it is subscribed, blocked or unknown, and a dead-symbol report lists pairs without trades since startup so that the list stays useful.

## Operating the service

A monitoring view shows connection health, activity, storage figures and the memory held per symbol. A debug call returns the live state of one symbol, a metrics call reports the service's counters, and a list of recent visitors shows the dashboard activity. Access lines are left out of the log by default so the service manager's log stays readable.

---

Private code: [gate-live-stats-backend](https://github.com/logicencoder/gate-live-stats-backend)

[Gate WordPress dashboard](https://github.com/logicencoder/gate-live-stats-plugin-overview)

See [REPOS.md](REPOS.md) for related repositories.

**[Logic Encoder](https://logicencoder.com)** · [GitHub](https://github.com/logicencoder) · [Contact](https://logicencoder.com/contact/)
