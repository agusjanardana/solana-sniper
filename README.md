# Solana Meme Sniper Bot

Personal-use automated sniper bot untuk meme coin di Solana multi-DEX (Raydium, Pump.fun, Meteora, Orca).

Fokus: **profit konsisten dengan risk management ketat**, bukan ultra-low-latency competitive sniping.

---

## Features

- 🔍 Multi-DEX token scanner (Raydium, Pump.fun, PumpSwap, Meteora, Orca)
- 🛡️ Scam/rug filter engine (mint authority, LP burn, holder distribution, dll)
- 📊 Risk engine dengan position sizing & kill switch
- 🎯 Auto TP/SL execution
- 🧪 Backtesting & paper trading mode
- 📈 Realtime dashboard (Next.js)
- 🔔 Notifications (Telegram/Discord)

---

## Quickstart

### Prerequisites

- Node.js 20+
- PostgreSQL 15+
- Redis 7+
- Solana wallet (**burner only!**)
- RPC endpoint (Helius/Quicknode/Triton recommended)

### Installation

```bash
# Clone
git clone https://github.com/agusjanardana/solana-sniper.git
cd solana-sniper

# Install dependencies
pnpm install

# Setup environment
cp .env.example .env
# Edit .env (lihat docs/07-security.md sebelum isi private key)

# Database migration
pnpm prisma migrate dev

# Start in paper trading mode (default)
pnpm dev
```

### Running Modes

```bash
# Backtest mode - replay data historis
pnpm run backtest --from=2025-01-01 --to=2025-01-31

# Paper trading - realtime tapi pakai fake money
pnpm run paper

# Live trading - HATI-HATI, pakai real money
pnpm run live
```

---

## Documentation

Semua dokumentasi teknis ada di folder [`docs/`](./docs):

| Doc | Isi |
|-----|-----|
| [01 - Architecture](./docs/01-architecture.md) | Arsitektur sistem, folder structure, data flow |
| [02 - Scam Filter](./docs/02-scam-filter.md) | Rules & scoring untuk filter rug/scam |
| [03 - Execution Engine](./docs/03-execution-engine.md) | Jito, priority fee, RPC strategy |
| [04 - Risk Engine](./docs/04-risk-engine.md) | Position sizing, TP/SL, kill switch |
| [05 - Data Infrastructure](./docs/05-data-infrastructure.md) | WebSocket, RPC providers, event ingestion |
| [06 - Backtesting](./docs/06-backtesting.md) | Backtest engine & paper trading spec |
| [07 - Security](./docs/07-security.md) | Wallet management, secrets, opsec |

---

## Tech Stack

**Frontend:** Next.js 14, Tailwind, shadcn/ui
**Backend:** Node.js, TypeScript
**Solana:** @solana/web3.js, Jupiter API, Raydium SDK, Pump.fun SDK
**Queue:** Redis + BullMQ
**Database:** PostgreSQL + Prisma ORM
**Monitoring:** Pino logger, Grafana (optional)

---

## ⚠️ Disclaimer

Software ini disediakan **as-is untuk tujuan edukasi & personal use**. Trading meme coin sangat berisiko — sebagian besar trader retail rugi. Anda bertanggung jawab penuh atas keputusan finansial Anda sendiri.

- ❌ Bukan financial advice
- ❌ Bukan jaminan profit
- ✅ Selalu pakai burner wallet
- ✅ Selalu paper trade dulu sebelum live
- ✅ Mulai dengan modal kecil

Lihat [docs/07-security.md](./docs/07-security.md) untuk security best practices.

---

## License

MIT (atau sesuai pilihan Anda)