# Gate.io Live Stats — backend

**FastAPI + PostgreSQL + Node SSR backend** for [logicencoder.com/gate-app/](https://logicencoder.com/gate-app/) and every **[/gate/{SYMBOL}/](https://logicencoder.com/gate/)** SEO page. The private service ingests Gate.io spot trades over JSON WebSockets, stores rolling history, computes 24h pair analytics, broadcasts live updates to browsers, and pushes Schema.org bundles to WordPress. Traders see the dashboard through the plugin; this repo documents the server-side half of that product pair.

## Tech stack

| Layer | Technologies |
|-------|--------------|
| Ingest | Python 3, FastAPI, uvicorn, Gate.io spot WebSocket (JSON), async workers |
| Storage | PostgreSQL, batched async inserts, rolling-window retention |
| Realtime | WebSocket with MessagePack frames (`gateio_trade`, `gateio_stats`), per-symbol subscription rooms |
| Analytics | SQL aggregates — 24h OHLC, volume splits, trade-size tiers, hourly buckets, bot-activity heuristics |
| WordPress sync | Batched Schema.org JSON push to plugin cache-warmup REST |
| SSR service | Node.js, Express, React server render for crawler HTML + JSON-LD |
| Ops | Operator dashboard WebSockets, monitoring REST, symbol reload API, static ops UI |
| Hosting | Self-hosted Linux server for compute; WordPress on shared hosting for public pages |

## Realtime trade ingest

The service maintains long-lived **Gate.io spot WebSocket** connections — one or more shards depending on fleet size, each subscribed to a chunk of USDT pairs. Incoming deal messages use Gate.io’s JSON shape (`currency_pair`, `price`, `amount`, `side`, `create_time_ms`); each tick is normalized to database format (`BTCUSDT`), deduplicated with a generated **trade id**, and **queued for batch insert** so the hot path never blocks on per-tick disk writes.

Broadcast to browsers happens **immediately** from the ingest path: trades enter a per-symbol FIFO queue with throttling so bandwidth spreads smoothly without dropping prints. USDT notional is computed at ingest time. **Exchange precision and full names** are cached from the Gate.io API and exposed through bootstrap so the WordPress UI formats prices and amounts correctly.

Pair list reload is **operator-triggered only** — a manual reload restarts WebSocket shards, loads precision for newly listed symbols, and avoids flapping subscriptions on every deploy. Stale rows fall out of the configured rolling window so queries and SSR builds stay bounded. **Roughly 700 USDT spot pairs** run on the live Gate fleet today.

## PostgreSQL analytics and REST

Trades land in PostgreSQL. Background workers run **optimized CTE queries** per symbol to recompute **24h open, high, low, volumes** in base asset and USDT, **buy/sell imbalance**, trade-count ratios, **VWAP**, volatility hints, **trade-size histograms** (small / medium / large tiers), largest-print metadata, average interval between prints, and **hourly buckets** for charts. Results populate an in-memory stats cache consumed by WebSocket broadcasts and REST.

Public and operator clients use REST for:

- **Bootstrap** — full symbol list, cached stats snapshot, and per-symbol precision metadata in one response for dashboard cold start.
- **Per-symbol memory stats** — fast path when a visitor switches pairs; also feeds a **priority queue** so actively viewed symbols recalculate sooner in large fleets.
- **Recent trades** — rolling-window history for tape backfill and hero price seeding when WebSocket has not printed yet.
- **Symbol status** — which pairs are live, idle, or never traded — mirrored in wp-admin Coin Manager.
- **Dead symbols** — diff against the exchange so operators prune delisted markets from sitemap and fleet lists.
- **Coin list CRUD** — add/remove tracked symbols from the server-side list; removals trigger connection reload.
- **Monitoring** — connection counts, throughput, queue depth, and health JSON for the plugin Monitor Dashboard.

Browsers open MessagePack WebSockets for live streams; the WordPress plugin proxies some admin calls while public clients connect directly for realtime data.

## WebSocket broadcast

Browser clients authenticate with an API key when required, then receive **MessagePack-compressed** frames. Trade frames use type **`gateio_trade`**; periodic aggregate refreshes use **`gateio_stats`**. Clients send **subscribe / unsubscribe** actions so only the active pair’s stats and trades update headline panels — cross-symbol contamination is ignored server-side.

An internal broadcast queue fans normalized frames to every session in a symbol room. **Operator dashboards** use separate dashboard sockets for throughput, latency, compression ratio, and structured boot logs without mixing admin traffic into the public tape. Server-wide stats broadcasts give wp-admin Monitor cards messages-per-second, download rate, reconnect counts, and PostgreSQL totals.

## WordPress schema sync

Unlike the MEXC pipeline’s disk snapshot POST pattern, Gate.io SEO leans on **live SSR through WordPress routing** plus **batched Schema.org pushes**. A background task walks the stats cache in rotating batches and POSTs structured **`Dataset` JSON-LD**, optional full names, and symbol keys to the WordPress plugin **cache-warmup** REST route with an API key header. WordPress stores transients the plugin and social previews can read without recomputing analytics in PHP.

Operators enable or tune batch size and interval via environment configuration. After a symbol reload or large listing batch, sync keeps plugin-side caches aligned with the authoritative server fleet.

## Node SSR for SEO pages

A companion **Express + React SSR** process renders crawler HTML for **`/gate/{SYMBOL}/`** pages: KPI strip, bot-activity section, 24h hourly table, JSON-LD block, and internal links. SSR fetches the same Python API bundles the live dashboard uses, so search snippets reflect computed fields rather than static marketing copy. WordPress **`template_redirect`** serves SSR HTML to crawlers; regular visitors stay on the interactive **`/gate-app/`** shell unless a test flag forces SSR.

Debug routes on the SSR service help operators validate a single symbol’s HTML structure before shipping a new listing — the same checks wp-admin Schema validation exercises from the plugin.

## Symbol fleet and precision management

**Pair discovery** pulls USDT markets from Gate.io APIs and optional operator-maintained coin files. Before subscribe, symbols pass **existence validation** so dead markets never wedge a connection shard. **Precision reload** fetches price and amount decimals plus **full names** for any symbol missing from cache.

Manual **`reload-symbols`** compares the new fleet to the active set, logs adds/removes, cancels old WebSocket tasks, and starts fresh shards — the workflow Coin Manager’s **Reload symbols on server** button triggers. Stats calculation uses **rotation across the full universe** when symbol count exceeds per-cycle limits, while **priority symbols** (recently viewed on the dashboard) and **actively subscribed pairs** refresh first.

## Operator surfaces

- **Static ops dashboard** — browser UI on the backend host for at-a-glance Gate connection health (same family as the MEXC ops dashboard).
- **Dashboard WebSocket** — live throughput, latency min/avg, compression ratio, subscription room counts.
- **Dashboard logs WebSocket** — filtered connection lifecycle (connect, auth, bulk subscribe) without per-trade spam.
- **Monitoring REST** — JSON snapshot for plugin AJAX when admin sockets are unavailable.

Operators diagnose “stale public site” from wp-admin Monitor without SSH, though all compute runs on the self-hosted server — not on shared hosting.

## Stats engine behaviour at fleet scale

With **~700 pairs**, the stats calculator cannot recompute every symbol every second. Each cycle caps how many symbols run heavy SQL, prioritizing **dashboard-polled symbols**, **WebSocket-subscribed pairs**, and a **rotating slice** of the rest so the full universe catches up over time. Results merge into `gateio_stats_cache` and emit **`gateio_stats`** frames to subscribed clients on a throttle per symbol — matching the live app’s 24h panels, bot scores, and hourly chart without polling REST on every tick.

## Not the trading bot

**[mexc_trading_app](https://github.com/logicencoder/mexc_trading_app-overview)** is a separate local trading console with order placement. This backend ingests **public Gate.io spot trades only** for the Logic Encoder stats site — no order execution, no withdrawal APIs.

## Shared hosting headroom

The public site lives on **WordPress shared hosting**. That environment is ideal for pages, shortcodes, sitemaps, and cached HTML — not for subscribing to hundreds of USDT markets and writing millions of trades. This backend exists so **all heavy work stays off PHP**: persistent exchange sockets, deduplication, PostgreSQL, stats recomputation, MessagePack fan-out, Schema.org batch sync, and SSR data bundles run on **self-hosted Linux servers** with async workers, then push compact results to WordPress.

**Roughly 700 Gate.io USDT pairs** stream through this pipeline; the parallel MEXC backend carries **1,400+**. Realtime data still reaches browsers over WebSocket; REST and WordPress transients cover networks that block WS. WordPress remains a **display, routing, and SEO layer** — not the database of record for ticks.

After iteratively offloading ingest, batching writes, and narrowing what PHP regenerates on each request, shared-hosting resource graphs show large margins on CPU, memory, PHP workers, throughput, IOPS, and process limits.

![Shared hosting — CPU and memory usage vs plan limits](assets/hostinger-cpu-memory.jpg)

![Shared hosting — disk throughput and PHP worker count](assets/hostinger-throughput-workers.jpg)

![Shared hosting — storage IOPS and max processes](assets/hostinger-iops-processes.jpg)

Private code: [gate-live-stats-backend](https://github.com/logicencoder/gate-live-stats-backend)

Plugin overview: [gate-live-stats-plugin-overview](https://github.com/logicencoder/gate-live-stats-plugin-overview)

Sibling backend (MEXC exchange feed): [mexc-live-stats-backend-overview](https://github.com/logicencoder/mexc-live-stats-backend-overview)

See [REPOS.md](REPOS.md).

---

**Made by [Logic Encoder](https://logicencoder.com)** · [GitHub](https://github.com/logicencoder) · [Contact](https://logicencoder.com/contact/)
