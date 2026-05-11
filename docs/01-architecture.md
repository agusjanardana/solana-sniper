# 01 — Architecture Overview

## High-Level Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCES                                │
│  Helius/Quicknode RPC  │  Bitquery  │  DEX WebSocket subscriptions │
└──────────┬─────────────────────────────────────────┬───────────────┘
           │                                         │
           ▼                                         ▼
┌──────────────────────┐                  ┌──────────────────────┐
│   Token Scanner      │                  │   Price Feed         │
│  (per-DEX adapter)   │                  │  (for TP/SL & pos.)  │
└──────────┬───────────┘                  └──────────┬───────────┘
           │ new pool event                          │
           ▼                                         │
┌──────────────────────┐                             │
│  Scam Filter Engine  │                             │
│  (rules + scoring)   │                             │
└──────────┬───────────┘                             │
           │ pass                                    │
           ▼                                         │
┌──────────────────────┐                             │
│   Risk Engine        │◄────────────────────────────┘
│  (position sizing)   │
└──────────┬───────────┘
           │ approved trade
           ▼
┌──────────────────────┐         ┌──────────────────────┐
│  Execution Engine    │────────►│  Position Manager    │
│  (Jupiter/direct)    │  filled │  (open positions DB) │
└──────────────────────┘         └──────────┬───────────┘
                                            │ monitor
                                            ▼
                                 ┌──────────────────────┐
                                 │  TP/SL Engine        │
                                 │  (auto exit)         │
                                 └──────────┬───────────┘
                                            │ exit signal
                                            ▼
                                 ┌──────────────────────┐
                                 │  Execution Engine    │
                                 │  (sell)              │
                                 └──────────┬───────────┘
                                            │
                                            ▼
                                 ┌──────────────────────┐
                                 │  Notification +      │
                                 │  Trade Log (DB)      │
                                 └──────────────────────┘
```

---

## Core Design Decisions

### 1. Event-Driven with Queue

Setiap stage berkomunikasi via **BullMQ queue**, bukan direct call:

- `pool.detected` → consumed by Scam Filter
- `pool.passed-filter` → consumed by Risk Engine
- `trade.approved` → consumed by Execution Engine
- `trade.filled` → consumed by Position Manager
- `position.exit-signal` → consumed by Execution Engine (sell side)

**Kenapa?** Mode switching (paper vs live) tinggal swap consumer. Retry, observability, dan back-pressure handling jadi mudah.

### 2. Pluggable DEX Adapters

Tiap DEX punya adapter dengan interface yang sama:

```typescript
interface DexAdapter {
  name: 'raydium' | 'pump' | 'meteora' | 'orca';
  subscribeNewPools(): AsyncIterable<PoolEvent>;
  getPoolInfo(poolAddress: string): Promise<PoolInfo>;
  buildSwapTx(params: SwapParams): Promise<VersionedTransaction>;
  getPrice(poolAddress: string): Promise<number>;
}
```

Tambah DEX baru = bikin adapter baru, tidak perlu ubah core.

### 3. Mode-Aware Execution

Execution Engine punya 3 implementasi:

- `LiveExecutor` — real tx via Jupiter/direct DEX
- `PaperExecutor` — simulate fill dengan realistic slippage
- `BacktestExecutor` — replay historical fill

Yang lain (scanner, filter, risk) **mode-agnostic** — output sama untuk semua mode.

### 4. Idempotent Operations

Setiap trade punya `clientOrderId` (UUID). Retry tidak menghasilkan double buy.

DB punya constraint:
```sql
UNIQUE (client_order_id, status)
```

---

## Data Model (Prisma Sketch)

```prisma
model Token {
  mint            String   @id
  symbol          String?
  name            String?
  decimals        Int
  createdAt       DateTime @default(now())
  // filter results
  mintAuthority   String?
  freezeAuthority String?
  lpBurned        Boolean?
  scamScore       Float?
  pools           Pool[]
}

model Pool {
  address         String   @id
  dex             String   // 'raydium' | 'pump' | 'meteora' | 'orca'
  baseMint        String
  quoteMint       String   // SOL/USDC
  liquidityUsd    Float?
  createdAt       DateTime
  token           Token    @relation(fields: [baseMint], references: [mint])
  trades          Trade[]
}

model Trade {
  id              String   @id @default(uuid())
  clientOrderId   String   @unique
  poolAddress     String
  side            String   // 'buy' | 'sell'
  mode            String   // 'live' | 'paper' | 'backtest'
  amountIn        Float
  amountOut       Float?
  priceUsd        Float?
  txSignature     String?
  status          String   // 'pending' | 'filled' | 'failed' | 'skipped'
  reason          String?
  createdAt       DateTime @default(now())
  pool            Pool     @relation(fields: [poolAddress], references: [address])
}

model Position {
  id              String   @id @default(uuid())
  tokenMint       String
  entryPrice      Float
  amountTokens    Float
  amountInSol     Float
  tpPrice         Float
  slPrice         Float
  trailingHigh    Float?
  status          String   // 'open' | 'closed'
  pnlSol          Float?
  openedAt        DateTime @default(now())
  closedAt        DateTime?
}

model DailyStats {
  date            DateTime @id
  trades          Int
  wins            Int
  losses          Int
  pnlSol          Float
  killSwitchHit   Boolean
}
```

---

## Folder Structure (Detail)

```
solana-sniper/
├── apps/
│   ├── dashboard/                  # Next.js dashboard
│   │   ├── app/
│   │   │   ├── (dashboard)/
│   │   │   │   ├── positions/
│   │   │   │   ├── trades/
│   │   │   │   └── settings/
│   │   │   └── api/
│   │   └── components/
│   │
│   └── worker/                     # Background worker (main entry)
│       ├── src/
│       │   ├── index.ts            # Boot all workers
│       │   ├── workers/
│       │   │   ├── scanner.ts
│       │   │   ├── filter.ts
│       │   │   ├── execution.ts
│       │   │   └── tpsl.ts
│       │   └── modes/
│       │       ├── live.ts
│       │       ├── paper.ts
│       │       └── backtest.ts
│       └── package.json
│
├── packages/
│   ├── scanner/
│   │   ├── src/
│   │   │   ├── adapters/
│   │   │   │   ├── raydium.ts
│   │   │   │   ├── pump.ts
│   │   │   │   ├── meteora.ts
│   │   │   │   └── orca.ts
│   │   │   ├── types.ts
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── filter/
│   │   ├── src/
│   │   │   ├── rules/
│   │   │   │   ├── mint-authority.ts
│   │   │   │   ├── freeze-authority.ts
│   │   │   │   ├── lp-burn.ts
│   │   │   │   ├── holder-distribution.ts
│   │   │   │   ├── liquidity.ts
│   │   │   │   ├── honeypot.ts
│   │   │   │   └── metadata.ts
│   │   │   ├── scorer.ts
│   │   │   └── index.ts
│   │
│   ├── risk/
│   ├── execution/
│   │   ├── src/
│   │   │   ├── providers/
│   │   │   │   ├── jupiter.ts
│   │   │   │   ├── raydium-direct.ts
│   │   │   │   └── jito.ts
│   │   │   ├── priority-fee.ts
│   │   │   ├── retry.ts
│   │   │   └── index.ts
│   │
│   ├── tpsl/
│   ├── position/
│   ├── backtest/
│   ├── paper/
│   ├── notify/
│   ├── db/
│   ├── shared/
│   └── solana/
│
├── services/
│   ├── api/                        # Fastify HTTP API
│   └── webhook/                    # Helius webhook receiver
│
├── prisma/
│   └── schema.prisma
│
├── docs/
└── ...
```

---

## Configuration Strategy

Config dibaca dari `.env` + `config.yaml`:

- `.env` — secrets (RPC key, private key, telegram token)
- `config.yaml` — strategy params (TP%, SL%, max position size, filters)

`config.yaml` boleh hot-reload (watch file), `.env` tidak.

Contoh `config.yaml`:

```yaml
mode: paper  # paper | live | backtest

risk:
  maxPositionSizeSol: 0.1
  maxConcurrentPositions: 3
  dailyLossLimitSol: 0.5
  killSwitchAfterConsecutiveLosses: 5

filter:
  minLiquidityUsd: 5000
  maxLiquidityUsd: 100000
  requireMintRenounced: true
  requireLpBurned: true
  minHolders: 20
  maxTopHolderPct: 15

execution:
  slippageBps: 1500  # 15%
  priorityFeeMode: dynamic  # static | dynamic
  staticPriorityFeeMicroLamports: 100000
  maxRetries: 3
  useJito: false

tpsl:
  takeProfitPct: 50
  stopLossPct: 20
  trailingStopPct: 15
  maxHoldMinutes: 30

dex:
  enabled:
    - raydium
    - pump
    - meteora
    - orca
```

---

## Observability

- **Logs**: Pino structured JSON, level per module
- **Metrics**: Counter (trades, errors), Histogram (latency, slippage)
- **Tracing**: Optional OpenTelemetry, trace per pool event end-to-end
- **Dashboard**: Live P&L, open positions, recent trades, error rate

---

## Testing Strategy

- **Unit tests**: Filter rules, risk calculations, TP/SL math
- **Integration tests**: Mock RPC, full pipeline scanner → execution (paper mode)
- **Backtest**: Historical replay sebagai regression test
- **Paper trading**: Mandatory before live, minimum 1 minggu

---

## Related Docs

- [02 — Scam Filter](./02-scam-filter.md)
- [03 — Execution Engine](./03-execution-engine.md)
- [04 — Risk Engine](./04-risk-engine.md)
- [05 — Data Infrastructure](./05-data-infrastructure.md)
- [06 — Backtesting](./06-backtesting.md)
- [07 — Security](./07-security.md)