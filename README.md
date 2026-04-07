# CryptoAlertBot

A production-grade Telegram bot for real-time cryptocurrency price alerts. Monitors every trading pair on Binance simultaneously via a single persistent WebSocket connection and delivers sub-second notifications through an event-driven Redis pipeline.

---

## Architecture

```
Binance WebSocket (2000+ pairs)
        │
        ▼
  priceService.ts          ← Single WS connection, firehose ingestion
        │
        ▼
  Redis Pub/Sub            ← Decoupled event broadcast
        │
        ▼
  alertService.ts          ← O(1) threshold validation via Redis ZSets
        │
        ▼
  notificationService.ts   ← Telegram Bot API delivery
        │
        ▼
     User
```

---

## Features

**Real-time engine**
- Ingests 2,000+ live Binance price streams through a single WebSocket connection — eliminates per-pair connection overhead
- Redis Pub/Sub decouples ingestion from alert evaluation — price updates never block notification delivery
- Redis Sorted Sets (ZSets) power O(1) threshold lookup — no linear scanning regardless of alert volume

**Alert system**
- Free tier: up to 30 simultaneous active alerts per user
- Premium tier: unlimited alerts via Telegram Payments subscription (monthly/yearly)
- Smart direction detection: enter any price, ZSet position determines long/short automatically — no manual up/down selection
- Inline alert management: list all alerts, delete any alert with a single tap

**User management**
- Auto-provisioning on `/start` — unique user record created with UUID on first interaction
- Session state management with Redis TTL for multi-step flows
- Prisma ORM with PostgreSQL for persistent user and alert storage
- Full Prisma migration history included

**Production-ready**
- PM2 cluster mode process management
- Winston logger with environment-aware formatting (JSON in prod, colorized in dev)
- Separate error and combined log transports
- Health check endpoint

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js, TypeScript |
| Bot framework | Grammy |
| Message broker | Redis Pub/Sub |
| Alert index | Redis Sorted Sets |
| Database | PostgreSQL + Prisma ORM |
| Process manager | PM2 |
| Logger | Winston |
| Payments | Telegram Payments API |

---

## Project Structure

```
src/
├── bot.ts                  # Bot initialization and middleware
├── index.ts                # Entry point, service bootstrap
├── config/
│   ├── cryptoList.ts       # Binance pair fetching
│   └── cryptoSym.ts        # Symbol normalization
├── controllers/
│   ├── inputController.ts          # User input validation
│   └── paymentServiceController.ts # Payment webhook handling
├── lib/
│   ├── prisma.ts           # Prisma client singleton
│   └── redisClient.ts      # Redis client singleton
├── models/
│   └── Alert.ts            # Alert domain model
├── router/
│   └── dashboard.ts        # Health + status endpoints
├── services/
│   ├── priceService.ts     # WebSocket firehose + Redis publish
│   ├── alertService.ts     # ZSet threshold evaluation
│   ├── notificationService.ts  # Telegram delivery
│   └── paymentService.ts   # Subscription management
└── utils/
    └── logger.ts           # Winston logger config
```

---

## Local Setup

**Prerequisites:** Node.js 18+, PostgreSQL, Redis

```bash
git clone https://github.com/Enmilo-dev/crypto-alert-bot.git
cd crypto-alert-bot
npm install
```

Create `.env`:

```env
DATABASE_URL="postgresql://user:password@localhost:5432/cryptoalertbot"
REDIS_URL="redis://localhost:6379"
BOT_TOKEN="your_telegram_bot_token"
ADMIN_ID="your_telegram_user_id"
NODE_ENV="development"
LOG_LEVEL="info"
```

```bash
npx prisma migrate deploy
npm run build
npm start
```

---

## Key Technical Decisions

**Why a single WebSocket instead of per-pair connections?**
Binance allows subscribing to multiple streams over one connection. Opening thousands of individual connections would hit rate limits and create massive overhead. The firehose approach streams everything through one pipe and filters in-process.

**Why Redis ZSets for alert thresholds?**
A sorted set keyed by price stores all alert thresholds in order. When a price update arrives, a single `ZRANGEBYSCORE` call returns all triggered alerts — O(log n) lookup regardless of how many alerts exist. No full table scans.

**Why Redis Pub/Sub between price ingestion and alert evaluation?**
Decoupling means the WebSocket handler never waits for database operations. If alert evaluation slows down under load, price ingestion is unaffected.

---

## License

MIT
