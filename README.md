<div align="center">

<img src="docs/assets/banner.jpg" alt="PaperPip: real market, paper money" width="100%" />

# PaperPip

**Real-time crypto paper trading on the live market.**
Trade BTC, ETH and SOL with up to 100x leverage, real spreads and a real liquidation engine.
The only thing that isn't real is the money.

[**Open the app →**](https://paperpip.sanskarshukla.com) &nbsp;·&nbsp; [Watch the launch film](#-launch-film) &nbsp;·&nbsp; [How it works](#-how-it-works) &nbsp;·&nbsp; [Run it locally](#-run-it-locally)

![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178C6?logo=typescript&logoColor=white)
![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)
![Bun](https://img.shields.io/badge/Bun-1.3-000000?logo=bun&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-pub%2Fsub-DC382D?logo=redis&logoColor=white)
![Kafka](https://img.shields.io/badge/Kafka-streams-231F20?logo=apachekafka&logoColor=white)
![TimescaleDB](https://img.shields.io/badge/TimescaleDB-PG16-FDB515?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-compose-2496ED?logo=docker&logoColor=white)

</div>

---

## 🎬 Launch film

<a href="https://paperpip.sanskarshukla.com">
  <img src="docs/assets/launch-poster.jpg" alt="Watch the PaperPip launch film" width="100%" />
</a>

<!--
  To embed the playable video here: edit this README on github.com, drag
  launch-video/PaperPip_Launch_github.mp4 into the editor on the line below,
  and GitHub will insert a https://github.com/user-attachments/assets/... link
  that renders as an inline player.
-->

---

## Why PaperPip

Most people learn what "liquidation" means the expensive way. PaperPip lets you learn it for free.

You get **$5,000 in paper money** and a trading terminal wired to the **live Binance market**. Prices, spreads and liquidations behave the way they would on a real exchange, so the habits you build here are the habits you'd take to a real account.

| | |
|---|---|
| ⚡ **Live market data** | BTC, ETH and SOL streamed from Binance and pushed to your screen over WebSockets |
| 📈 **Pro charting** | Candlesticks at 1m, 1d and 1w, with live Buy/Sell price lines, an OHLC legend and follow mode |
| 🎚️ **Leverage up to 100x** | 1x, 5x, 10x, 20x or 100x, with a live risk meter and warnings before you commit |
| 🛡️ **Risk controls** | Take-profit, stop-loss and trailing stop-loss, with quick presets and estimated P&L |
| 💥 **Real liquidations** | Positions are closed automatically the moment the price crosses your liquidation level |
| ✂️ **Position management** | Close fully, close partially (1–99%), or add margin to move your liquidation price |
| 🔔 **Instant feedback** | Order opened, closed and liquidated events are pushed to you in real time |
| 🔐 **Accounts** | Email + password with 4-digit email verification, or Google / GitHub sign-in |

---

## 📐 Trading rules

PaperPip uses the same mechanics as a real CFD broker. Nothing is simplified.

| Rule | Value |
|---|---|
| Starting balance | $5,000 (paper) |
| Markets | BTC/USDT, ETH/USDT, SOL/USDT |
| Leverage | 1x · 5x · 10x · 20x · 100x |
| Spread | 0.05% around mid price (configurable via `SPREAD_PERCENT`) |
| Fees | 0.5% of margin on open, 0.5% of margin on close |
| Buy price / sell price | You buy at the **ask** and sell at the **bid** |
| Liquidation (long) | `entry × (leverage − 1) / leverage` |
| Liquidation (short) | `entry × (leverage + 1) / leverage` |
| Order types | Market orders, with optional TP, SL and trailing SL |

> Example: a 20x long on BTC at $112,970 liquidates at about $107,322, a 5% move against you.

---

## 🧠 How it works

PaperPip runs as four independent services behind a single web entry point, the same shape as a small exchange.

```mermaid
flowchart LR
    B[Binance<br/>aggTrade streams] -->|ticks| P[Price Poller]
    P -->|bid / ask| R[(Redis<br/>pub/sub)]
    P -->|trade events| K[[Kafka]]
    K -->|batched writes| T[(PostgreSQL<br/>+ TimescaleDB)]
    R --> W[WebSocket server]
    R --> A[Backend API<br/>matching + risk engine]
    A -->|order events| R
    A <-->|users, orders, snapshots| T
    W -->|prices + your orders| F[React client]
    F -->|REST / JWT| A
```

**The path of a price tick**

1. **Price Poller** subscribes to Binance aggregate-trade streams for BTC, ETH and SOL, applies the spread, and fans every tick out to Redis (real time) and Kafka (history).
2. **WebSocket server** holds authenticated client connections and relays price updates plus each user's own order events.
3. **Backend API** keeps every open position **in memory**, checks TP / SL / trailing SL / liquidation on every tick, and publishes order events back through Redis.
4. **Kafka consumer** batches trades (500 trades or 5 seconds, deduplicated by Binance trade ID) into a TimescaleDB hypertable that powers the candle charts.

### Engineering decisions

- **Money is never a float.** Prices are stored as integers scaled by 10,000 and balances in cents. P&L is computed with `BigInt`, so rounding can't create or destroy money.
- **Risk checks don't wait for the database.** Positions live in memory so a liquidation fires on the exact tick. A snapshot is written to Postgres every 10 seconds and restored on startup, so a crash doesn't wipe anyone's positions.
- **The chart can't be overwhelmed.** Ticks arrive faster than the screen can draw, so the client coalesces them and repaints at most once per animation frame.
- **Self-healing pipeline.** A watchdog in the history pipeline exits if no database write succeeds within its time limit, and Docker restarts the service cleanly instead of letting it hang.
- **Abuse protection.** Rate limits on auth (5 per 15 min), signup (10 per 15 min), trading (30 per minute) and closes (60 per minute).

---

## 🧰 Tech stack

| Layer | Technology |
|---|---|
| Client | React 19, Vite 7, Tailwind CSS 4, lightweight-charts 5 |
| API | Express on Bun, JWT auth, Zod validation, Google & GitHub OAuth |
| Real time | `ws` WebSocket server, Redis pub/sub |
| Streaming | Kafka (Confluent 7.5) |
| Storage | PostgreSQL 16 + TimescaleDB 2.17, Prisma ORM |
| Email | Resend (verification codes) |
| Monorepo | Turborepo, Bun workspaces, shared packages |
| Infra | Docker Compose, nginx, automatic TLS, self-managed VPS |

---

## 📁 Project structure

```
.
├── apps/
│   ├── Backend/        # REST API, matching + risk engine, snapshots, OAuth, email
│   ├── Websocket/      # authenticated real-time gateway
│   ├── Price_Poller/   # Binance ingest → Redis + Kafka → TimescaleDB
│   └── Frontend/       # React trading terminal
├── packages/
│   ├── database/       # Prisma schema, migrations, generated client
│   ├── shared/         # price/money scaling, assets, leverage constants
│   └── typescript-config, eslint-config
├── scripts/            # historical data seeding + health checks
├── deploy/             # nginx site config, Timescale retention policy
├── docker-compose.yml       # local infrastructure
└── docker-compose.prod.yml  # full production stack
```

---

## 🚀 Run it locally

**Prerequisites:** Bun 1.3+, Docker, Node 18+

```bash
# 1. Install and start infrastructure (Postgres/Timescale, Redis, Kafka)
bun install
docker compose up -d

# 2. Configure the backend: create apps/Backend/.env (see below)

# 3. Database
bun run prisma generate
bun run prisma migrate dev
bun run seed-data          # pulls 7 days of Binance history (5–10 min)

# 4. Start everything
bun run dev
```

<details>
<summary><b>Backend environment variables</b></summary>

```env
DATABASE_URL=postgresql://user:password@localhost:5432/trades_db
REDIS_URL=redis://localhost:6379
KAFKA_BROKERS=localhost:9092
JWT_SECRET=at-least-32-random-characters
PORT=5000
INITIAL_BALANCE_USD=5000
CORS_ORIGINS=http://localhost:5173

# optional
RESEND_API_KEY=            # email verification (auto-verifies if empty)
EMAIL_FROM=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
```
</details>

| Service | URL |
|---|---|
| Trading terminal | http://localhost:5173 |
| REST API | http://localhost:5000/api/v2 |
| WebSocket | ws://localhost:8080 |
| Prisma Studio | `bun run prisma studio` → http://localhost:5555 |

---

## 🔌 API reference

All trading endpoints need `Authorization: Bearer <token>`.

<details>
<summary><b>Auth & account</b></summary>

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v2/user/signup` | Create an account (sends a 4-digit code) |
| `POST` | `/api/v2/user/verify-email` | Verify the email code |
| `POST` | `/api/v2/user/resend-code` | Resend the verification code |
| `POST` | `/api/v2/user/signin` | Sign in, returns a JWT |
| `GET` | `/api/v2/user/balance` | Current paper balance |
| `GET` | `/api/v2/auth/google` · `/api/v2/auth/github` | OAuth sign-in |
</details>

<details>
<summary><b>Trading</b></summary>

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/v2/trade/open` | Open a position (margin, leverage, optional TP / SL / trailing SL) |
| `POST` | `/api/v2/trade/close` | Close a position, returns realized P&L |
| `POST` | `/api/v2/trade/partial-close` | Close 1–99% of a position |
| `POST` | `/api/v2/trade/add-margin` | Add collateral to push liquidation further away |
| `GET` | `/api/v2/trade/open` | Open positions |
| `GET` | `/api/v2/trade/history` | Closed positions with P&L |
</details>

<details>
<summary><b>Market data</b></summary>

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v2/asset` | Assets with current bid / ask |
| `GET` | `/api/v2/asset/candles?asset=BTC&ts=1m` | OHLC candles (`1m`, `1d`, `1w`) from TimescaleDB |
</details>

<details>
<summary><b>WebSocket events</b></summary>

| Event | Direction | Payload |
|---|---|---|
| `AUTH` | client → server | `{ token }` |
| `SUBSCRIBE` / `UNSUBSCRIBE` | client → server | `{ symbol }` |
| `PRICE_UPDATE` | server → client | `{ symbol, bidPrice, askPrice }` |
| `ORDER_OPENED` / `ORDER_CLOSED` / `ORDER_LIQUIDATED` | server → client | order details and P&L |
</details>

---

## ☁️ Deployment

Production runs the whole stack with a single command on a self-managed VPS:

```bash
cp env.prod.example .env.prod   # fill in secrets
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --build
```

Only one port is public: the web container, which serves the client and proxies `/api/v2` and `/ws` internally. Postgres, Redis, Kafka and the services are bound to localhost. The full step-by-step runbook is in [`DEPLOY.md`](DEPLOY.md).

---

## ⚠️ Disclaimer

PaperPip is a **simulator**. No real money is deposited, traded or withdrawn, and nothing here is financial advice. Leveraged trading on real exchanges can lose you more than you expect. That is exactly why this exists.

---

<div align="center">

Built by **[Sanskar Shukla](https://www.sanskarshukla.com)** &nbsp;·&nbsp; [X @sanskar0627](https://x.com/sanskar0627) &nbsp;·&nbsp; [LinkedIn](https://linkedin.com/in/sanskar2003) &nbsp;·&nbsp; [GitHub](https://github.com/sanskar0627)

If PaperPip saved you from a real liquidation, a ⭐ helps.

</div>
