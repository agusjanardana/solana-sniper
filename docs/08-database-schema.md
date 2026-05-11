# 08 — Database Schema

Detail Prisma schema, relasi, index strategy, dan migration practices.

---

## Overview

Pakai PostgreSQL 15+ karena:
- JSONB native (untuk store raw event payload)
- Window function untuk analytics (rolling P&L, drawdown)
- Materialized view untuk dashboard performance
- Mature & predictable di production

ORM: Prisma. Reasoning: type-safe, good migration tooling, fits TypeScript stack.

---

## Schema Overview

```
┌──────────┐       ┌──────────┐       ┌──────────┐
│  Token   │ 1──┐  │   Pool   │ 1──┐  │  Trade   │
└──────────┘    └──│          │    └──│          │
                   └──────────┘       └──────────┘
                                            │ N
                                            ▼
                                      ┌──────────┐
                                      │ Position │
                                      └──────────┘

┌─────────────┐  ┌───────────┐  ┌────────────────┐
│ FilterLog   │  │ DailyStats│  │ SystemEvent    │
└─────────────┘  └───────────┘  └────────────────┘

┌────────────────┐  ┌──────────────────┐
│ HistoricalPool │  │  BacktestRun     │
│ (for backtest) │  │                  │
└────────────────┘  └──────────────────┘
```

---

## Full Schema

```prisma
// prisma/schema.prisma

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [pg_trgm]
}

// ============================================================
// CORE TRADING ENTITIES
// ============================================================

model Token {
  mint            String   @id @db.VarChar(64)
  symbol          String?  @db.VarChar(32)
  name            String?  @db.VarChar(128)
  decimals        Int
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  // On-chain attributes
  mintAuthority   String?  @db.VarChar(64)
  freezeAuthority String?  @db.VarChar(64)
  totalSupply     String?  // bigint as string untuk avoid JS number limit
  
  // Metadata
  uri             String?  @db.Text
  imageUrl        String?  @db.Text
  description     String?  @db.Text
  socials         Json?    // { twitter, telegram, website }
  
  // Filter cache
  scamScore       Float?
  lastFilteredAt  DateTime?
  filterPassed    Boolean?
  
  // Relations
  pools           Pool[]
  positions       Position[]
  filterLogs      FilterLog[]
  
  @@index([symbol])
  @@index([createdAt])
  @@index([scamScore])
}

model Pool {
  address         String   @id @db.VarChar(64)
  dex             DexType
  baseMint        String   @db.VarChar(64)
  quoteMint       String   @db.VarChar(64)
  
  // Initial state (saat detect)
  initialLiquidityUsd  Float?
  initialBaseReserve   String?
  initialQuoteReserve  String?
  creatorAddress       String?  @db.VarChar(64)
  
  // LP info
  lpMint          String?  @db.VarChar(64)
  lpBurnedPct     Float?
  
  // Time tracking
  createdAt       DateTime
  createdAtSlot   BigInt?
  detectedAt      DateTime @default(now())
  
  // Status
  status          PoolStatus @default(ACTIVE)
  
  // Raw event data untuk debugging
  rawEvent        Json?
  
  // Relations
  token           Token    @relation(fields: [baseMint], references: [mint])
  trades          Trade[]
  
  @@index([baseMint])
  @@index([dex])
  @@index([createdAt])
  @@index([status])
}

model Trade {
  id              String      @id @default(uuid())
  clientOrderId   String      @unique @db.VarChar(64)
  poolAddress     String      @db.VarChar(64)
  tokenMint       String      @db.VarChar(64)
  
  side            TradeSide
  mode            TradeMode
  
  // Amounts
  amountInSol     Float
  amountInLamports BigInt
  amountOutTokens String?     // bigint as string
  priceUsd        Float?
  priceSol        Float?
  
  // Execution detail
  provider        String?     // 'jupiter' | 'raydium-direct' | etc
  slippageBps     Int
  actualSlippagePct Float?
  priorityFeeMicroLamports Int?
  computeUnitsUsed Int?
  
  // Transaction
  txSignature     String?     @db.VarChar(128)
  txStatus        TxStatus    @default(PENDING)
  blockTime       DateTime?
  slot            BigInt?
  
  // Outcome
  status          TradeStatus
  reason          String?     @db.Text
  errorMessage    String?     @db.Text
  
  // Linking
  positionId      String?
  parentTradeId   String?     // untuk partial sell, link ke buy
  
  // Timing
  createdAt       DateTime    @default(now())
  submittedAt     DateTime?
  confirmedAt     DateTime?
  durationMs      Int?
  attempts        Int         @default(1)
  
  // Relations
  pool            Pool        @relation(fields: [poolAddress], references: [address])
  position        Position?   @relation(fields: [positionId], references: [id])
  
  @@unique([txSignature])
  @@index([tokenMint])
  @@index([status])
  @@index([mode])
  @@index([createdAt])
  @@index([positionId])
}

model Position {
  id              String      @id @default(uuid())
  tokenMint       String      @db.VarChar(64)
  poolAddress     String      @db.VarChar(64)
  mode            TradeMode
  
  // Entry
  entryPrice      Float
  entryPriceSol   Float
  amountTokens    String      // bigint as string
  amountInSol     Float
  
  // Risk params (snapshot saat open)
  tpPrice         Float
  slPrice         Float
  trailingStopPct Float?
  maxHoldMinutes  Int
  
  // Live tracking
  currentPrice    Float?
  trailingHigh    Float?
  unrealizedPnlSol Float?
  unrealizedPnlPct Float?
  
  // Partial TPs
  partialsTriggered Float[]   @default([])  // [30, 100] = sudah trigger di +30% & +100%
  remainingTokens String?     // bigint as string, sisa setelah partial sells
  
  // Lifecycle
  status          PositionStatus
  openedAt        DateTime    @default(now())
  closedAt        DateTime?
  
  // Outcome (saat closed)
  realizedPnlSol  Float?
  realizedPnlPct  Float?
  exitReason      String?     // 'tp' | 'sl' | 'trailing' | 'timeout' | 'manual' | 'emergency'
  holdDurationSec Int?
  
  // Relations
  trades          Trade[]
  token           Token       @relation(fields: [tokenMint], references: [mint])
  
  @@index([status])
  @@index([mode])
  @@index([openedAt])
  @@index([tokenMint])
}

// ============================================================
// FILTER & DECISION LOGGING
// ============================================================

model FilterLog {
  id              String      @id @default(uuid())
  poolAddress     String      @db.VarChar(64)
  tokenMint       String      @db.VarChar(64)
  
  passed          Boolean
  finalScore      Float?
  rejectReason    String?     @db.VarChar(128)
  
  // Detail per layer (untuk analysis)
  layer1Results   Json?       // hard rules
  layer2Results   Json?       // on-chain checks
  layer3Results   Json?       // holder analysis
  layer4Results   Json?       // honeypot sim
  
  durationMs      Int
  filterProfile   String      // 'strict' | 'medium' | 'aggressive'
  
  createdAt       DateTime    @default(now())
  
  token           Token       @relation(fields: [tokenMint], references: [mint])
  
  @@index([poolAddress])
  @@index([tokenMint])
  @@index([passed])
  @@index([rejectReason])
  @@index([createdAt])
}

// ============================================================
// AGGREGATES & STATS
// ============================================================

model DailyStats {
  date                DateTime    @id @db.Date
  mode                TradeMode
  
  totalTrades         Int         @default(0)
  buyTrades           Int         @default(0)
  sellTrades          Int         @default(0)
  
  wins                Int         @default(0)
  losses              Int         @default(0)
  breakeven           Int         @default(0)
  
  pnlSol              Float       @default(0)
  pnlUsd              Float       @default(0)
  feesSpentSol        Float       @default(0)
  
  winRate             Float?
  profitFactor        Float?
  largestWinSol       Float?
  largestLossSol      Float?
  
  // Filter stats
  poolsScanned        Int         @default(0)
  poolsRejected       Int         @default(0)
  poolsApproved       Int         @default(0)
  
  // Operations
  killSwitchTriggered Boolean     @default(false)
  killSwitchReason    String?
  
  @@unique([date, mode])
  @@index([date])
}

// ============================================================
// SYSTEM EVENTS & OPS
// ============================================================

model SystemEvent {
  id              String      @id @default(uuid())
  type            EventType
  severity        EventSeverity
  source          String      @db.VarChar(64)  // 'scanner' | 'execution' | 'risk' | etc
  message         String      @db.Text
  context         Json?
  createdAt       DateTime    @default(now())
  
  @@index([type])
  @@index([severity])
  @@index([source])
  @@index([createdAt])
}

model KillSwitchEvent {
  id              String      @id @default(uuid())
  triggered       Boolean
  reason          String      @db.Text
  triggeredBy     String?     @db.VarChar(64)  // 'auto' | 'manual:username'
  resolvedAt      DateTime?
  resolvedBy      String?     @db.VarChar(64)
  resolveNote     String?     @db.Text
  context         Json?
  createdAt       DateTime    @default(now())
  
  @@index([triggered])
  @@index([createdAt])
}

model WalletBalance {
  id              String      @id @default(uuid())
  walletAddress   String      @db.VarChar(64)
  balanceSol      Float
  balanceLamports BigInt
  
  // Drift detection
  expectedBalanceSol Float?
  driftSol           Float?
  
  recordedAt      DateTime    @default(now())
  
  @@index([walletAddress, recordedAt])
}

// ============================================================
// BACKTEST DATA
// ============================================================

model HistoricalPool {
  id              String      @id @default(uuid())
  poolAddress     String      @unique @db.VarChar(64)
  dex             DexType
  baseMint        String      @db.VarChar(64)
  
  createdAt       DateTime
  createdAtSlot   BigInt?
  
  // Snapshot at creation
  initialLiquidityUsd  Float
  initialPrice         Float
  creatorAddress       String?  @db.VarChar(64)
  mintAuthorityAtT0    String?
  freezeAuthorityAtT0  String?
  lpBurnedAtT0         Boolean?
  
  // Time series (OHLCV)
  priceHistory    PriceSnapshot[]
  holderSnapshots HolderSnapshot[]
  
  // Outcome label
  outcome         PoolOutcome?
  maxPricePct     Float?      // peak gain dari entry
  finalPricePct   Float?
  ruggedAt        DateTime?
  
  collectedAt     DateTime    @default(now())
  
  @@index([dex])
  @@index([createdAt])
  @@index([outcome])
}

model PriceSnapshot {
  id              String      @id @default(uuid())
  poolId          String
  timestamp       DateTime
  
  open            Float
  high            Float
  low             Float
  close           Float
  volume          Float
  liquidityUsd    Float
  
  historicalPool  HistoricalPool @relation(fields: [poolId], references: [id], onDelete: Cascade)
  
  @@index([poolId, timestamp])
}

model HolderSnapshot {
  id              String      @id @default(uuid())
  poolId          String
  timestamp       DateTime
  
  totalHolders    Int
  topHolders      Json        // [{ address, pct, amount }]
  top10Pct        Float
  top1Pct         Float
  
  historicalPool  HistoricalPool @relation(fields: [poolId], references: [id], onDelete: Cascade)
  
  @@index([poolId, timestamp])
}

model BacktestRun {
  id              String      @id @default(uuid())
  name            String?     @db.VarChar(128)
  startDate       DateTime
  endDate         DateTime
  
  config          Json        // snapshot config yang dipakai
  
  // Results
  totalTrades     Int
  wins            Int
  losses          Int
  pnlSol          Float
  maxDrawdownSol  Float
  sharpeRatio     Float?
  profitFactor    Float?
  
  // Metadata
  durationMs      Int
  status          BacktestStatus
  
  createdAt       DateTime    @default(now())
  completedAt     DateTime?
  
  @@index([createdAt])
}

// ============================================================
// ENUMS
// ============================================================

enum DexType {
  RAYDIUM_AMM_V4
  RAYDIUM_CLMM
  RAYDIUM_CPMM
  PUMP
  PUMP_SWAP
  METEORA_DLMM
  METEORA_DBC
  ORCA_WHIRLPOOL
}

enum PoolStatus {
  ACTIVE
  RUGGED
  LOW_LIQUIDITY
  CLOSED
}

enum TradeSide {
  BUY
  SELL
}

enum TradeMode {
  LIVE
  PAPER
  BACKTEST
}

enum TradeStatus {
  PENDING
  SUBMITTED
  FILLED
  FAILED
  SKIPPED
  EXPIRED
}

enum TxStatus {
  PENDING
  PROCESSED
  CONFIRMED
  FINALIZED
  FAILED
  DROPPED
}

enum PositionStatus {
  OPEN
  PARTIALLY_CLOSED
  CLOSED
}

enum EventType {
  STARTUP
  SHUTDOWN
  RPC_ERROR
  RPC_RECOVERED
  WS_DISCONNECT
  WS_RECONNECT
  KILL_SWITCH
  BALANCE_DRIFT
  CONFIG_RELOAD
  HEARTBEAT_MISSED
}

enum EventSeverity {
  DEBUG
  INFO
  WARN
  ERROR
  CRITICAL
}

enum PoolOutcome {
  RUG          // liquidity drained / dump > 90%
  MOON         // peak > 5x
  GOOD         // peak 2-5x
  FLAT         // peak < 2x, no rug
  UNKNOWN      // belum cukup data
}

enum BacktestStatus {
  RUNNING
  COMPLETED
  FAILED
  CANCELLED
}
```

---

## Index Strategy

### Why These Indexes?

| Index | Use Case |
|-------|----------|
| `Trade(status)` | Worker query: "ambil semua trade pending" |
| `Trade(mode)` | Dashboard filter live vs paper |
| `Trade(createdAt)` | Time-range queries untuk daily stats |
| `Position(status)` | TP/SL monitor query: "semua open position" |
| `FilterLog(rejectReason)` | Analytics: alasan reject paling sering |
| `Pool(createdAt)` | Scanner: pool baru dalam 5 menit |
| `PriceSnapshot(poolId, timestamp)` | Backtest: replay price history |

### Composite Indexes (Tambahan)

Untuk query yang sering:

```sql
-- Daily P&L report
CREATE INDEX idx_trade_mode_created ON "Trade" (mode, "createdAt" DESC);

-- Open position lookup
CREATE INDEX idx_position_status_opened ON "Position" (status, "openedAt" DESC) 
  WHERE status = 'OPEN';

-- Filter analysis
CREATE INDEX idx_filter_passed_created ON "FilterLog" (passed, "createdAt" DESC);

-- Trade by token (P&L per token)
CREATE INDEX idx_trade_token_mode ON "Trade" ("tokenMint", mode);
```

Partial index `WHERE status = 'OPEN'` di Position kecil tapi sangat sering di-query → big win.

---

## Materialized View (Optional, Performance)

Untuk dashboard yang butuh agregasi cepat:

```sql
CREATE MATERIALIZED VIEW mv_token_performance AS
SELECT 
  t.mint,
  t.symbol,
  COUNT(DISTINCT tr.id) AS total_trades,
  SUM(CASE WHEN p."realizedPnlSol" > 0 THEN 1 ELSE 0 END) AS wins,
  SUM(p."realizedPnlSol") AS total_pnl_sol,
  AVG(p."realizedPnlPct") AS avg_pnl_pct,
  MAX(p."realizedPnlPct") AS best_pnl_pct,
  MIN(p."realizedPnlPct") AS worst_pnl_pct
FROM "Token" t
JOIN "Trade" tr ON tr."tokenMint" = t.mint
JOIN "Position" p ON p."tokenMint" = t.mint
WHERE p.status = 'CLOSED' AND p.mode = 'LIVE'
GROUP BY t.mint, t.symbol;

-- Refresh nightly
CREATE INDEX idx_mv_token_perf_pnl ON mv_token_performance (total_pnl_sol DESC);
```

Refresh schedule: cron job `REFRESH MATERIALIZED VIEW CONCURRENTLY mv_token_performance` setiap jam.

---

## Migration Strategy

### Development

```bash
# Edit schema.prisma
pnpm prisma migrate dev --name add_partial_tp_field
```

### Production

```bash
# Generate migration (jangan run di prod)
pnpm prisma migrate dev --create-only --name add_partial_tp_field

# Review migration SQL
cat prisma/migrations/<timestamp>_add_partial_tp_field/migration.sql

# Apply di prod
pnpm prisma migrate deploy
```

### Backwards-Compatible Migrations

Rules:
- ✅ Add nullable column → safe
- ✅ Add index `CONCURRENTLY` → safe
- ❌ Drop column → 2-step: stop using → next release drop
- ❌ Rename column → 2-step: add new → migrate data → drop old
- ❌ Change column type → biasanya butuh downtime

---

## Data Retention

PostgreSQL tidak auto-cleanup. Setup retention manual:

```sql
-- Cleanup old filter logs (keep 30 days)
DELETE FROM "FilterLog" 
WHERE "createdAt" < NOW() - INTERVAL '30 days'
  AND passed = false;

-- Cleanup paper trade after 90 days
DELETE FROM "Trade" 
WHERE mode = 'PAPER' 
  AND "createdAt" < NOW() - INTERVAL '90 days';

-- Vacuum after big deletes
VACUUM ANALYZE "FilterLog";
```

Schedule via cron (lihat [11-deployment.md](./11-deployment.md)).

---

## Backup Strategy

### Daily Logical Backup

```bash
# scripts/backup.sh
#!/bin/bash
DATE=$(date +%Y%m%d)
pg_dump $DATABASE_URL \
  --format=custom \
  --compress=9 \
  --file=/backups/sniper-$DATE.dump

# Encrypt
gpg --encrypt --recipient backup@yourdomain /backups/sniper-$DATE.dump

# Upload (rclone ke encrypted cloud)
rclone copy /backups/sniper-$DATE.dump.gpg backup-remote:sniper-backups/

# Retain 30 days locally, 90 days remote
find /backups -name "sniper-*.dump.gpg" -mtime +30 -delete
```

### Point-in-Time Recovery (Advanced)

Kalau bot udah jalan serius, enable PITR:
- WAL archiving ke S3
- Bisa restore ke point manapun dalam 7-30 hari

Untuk personal use awal: daily dump cukup.

---

## Connection Pool

Prisma default pool: `connection_limit = num_cpus * 2 + 1`. Untuk worker + dashboard di server kecil:

```env
DATABASE_URL="postgresql://user:pass@host:5432/sniper?connection_limit=10&pool_timeout=20"
```

Monitor `pg_stat_activity` untuk leak detection.

---

## Pitfalls

- ❌ **Pakai Float untuk amount kripto** — precision loss. Pakai string (bigint as string) atau Decimal
- ❌ **Index semua kolom** — over-indexing slow write. Profile dulu
- ❌ **Skip migration review di prod** — destructive migration tanpa testing = data loss
- ❌ **No backup** — disk fail = lose semua trade history
- ✅ **Always reversible migrations** — punya `down` script atau plan
- ✅ **Test migration di staging** dengan data realistic size