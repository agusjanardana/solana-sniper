# 06 — Backtesting + Paper Trading

Testing layers sebelum live. **Skip ini = lose money guaranteed.**

---

## Why Both?

| Mode | Tujuan | Trade-off |
|------|--------|-----------|
| **Backtest** | Validate strategi vs data historis, banyak iterasi cepat | Tidak refleksi latency real, slippage estimate |
| **Paper Trading** | Validate eksekusi & latency di market live | Lambat (real-time), bergantung market kondisi sekarang |

Workflow recommended:
1. **Backtest** untuk filter & strategy tuning (jalan dalam menit, banyak skenario)
2. **Paper trade** untuk eksekusi validation (1-2 minggu di market live)
3. **Live small** dengan modal kecil

---

## Backtest Engine

### Architecture

```
[Historical Data Store]
        │
        ▼
[Time-Ordered Event Replay]
        │
        ▼
[Same Filter + Risk + TP/SL pipeline]
        │
        ▼
[Simulated Execution (with fee/slippage model)]
        │
        ▼
[Trade Log + Metrics]
```

**Prinsip kunci**: Pipeline sama dengan live mode. Hanya `Executor` yang di-swap dengan `BacktestExecutor`.

### Data Collection

Untuk backtest, butuh dataset historis:

#### What to Collect

Untuk setiap pool baru (selama periode backtest):

```typescript
interface HistoricalPool {
  poolAddress: string;
  dex: string;
  baseMint: string;
  createdAt: Date;
  createdAtSlot: number;
  
  // Snapshot at creation
  initialLiquidityUsd: number;
  mintAuthority: string | null;
  freezeAuthority: string | null;
  lpBurnedAtT0: boolean;
  
  // Time series price (OHLCV)
  prices: Array<{
    timestamp: number;
    open: number;
    high: number;
    low: number;
    close: number;
    volume: number;
  }>;
  
  // Holder snapshots (t=0, t=1min, t=5min)
  holderSnapshots: Array<{
    timestamp: number;
    topHolders: Array<{ address: string; pct: number }>;
    totalHolders: number;
  }>;
  
  // Outcome (untuk supervised learning eventually)
  outcome: 'rug' | 'moon' | 'flat' | 'unknown';
  maxPricePct: number;  // peak gain
  finalPricePct: number; // current value
}
```

#### Where to Get Historical Data

| Source | Pro | Con |
|--------|-----|-----|
| **Bitquery archive** | Comprehensive | Paid for deep history |
| **Helius archived data** | Easy access | Limited backlog |
| **DexScreener API** | Free OHLCV | Rate limited, missing edge cases |
| **Birdeye historical** | OHLCV, holder data | Paid for full access |
| **Self-collected** | Customizable | Butuh waktu kumpulkan |

**Recommended (personal use)**: Self-collect dengan menjalankan scanner di "log-only mode" selama 2-4 minggu. Setiap event di-dump ke PostgreSQL. Setelah cukup data, baru backtest.

#### Self-Collection Mode

Tambah mode `collect` di worker:

```bash
pnpm run collect --duration=14d
```

```typescript
async function collectMode() {
  await subscribeAllDexes(async (poolEvent) => {
    // Capture snapshot at t=0
    await snapshotPool(poolEvent, 0);
    
    // Schedule follow-up snapshots
    setTimeout(() => snapshotPool(poolEvent, 60), 60_000);
    setTimeout(() => snapshotPool(poolEvent, 300), 300_000);
    setTimeout(() => snapshotPool(poolEvent, 1800), 1800_000);
    setTimeout(() => labelOutcome(poolEvent), 86400_000); // 24 hours
  });
}
```

### Replay Engine

```typescript
class BacktestEngine {
  async run(params: { from: Date; to: Date; config: Config }) {
    const pools = await db.historicalPool.findMany({
      where: { createdAt: { gte: params.from, lte: params.to }},
      orderBy: { createdAt: 'asc' },
    });
    
    const stats = new BacktestStats();
    
    for (const pool of pools) {
      // Simulate time-ordered event
      const filterResult = await runFilter(pool, params.config);
      stats.recordFilterResult(filterResult);
      
      if (!filterResult.pass) continue;
      
      const riskDecision = await runRisk(pool, stats.getOpenPositions(), params.config);
      if (!riskDecision.approved) continue;
      
      // Simulate fill at t=0 + delay (latency model)
      const fillDelay = simulateFillDelay(); // ~2-3 detik default
      const fillPrice = priceAt(pool, pool.createdAt.getTime() + fillDelay);
      
      const position = stats.openPosition({
        mint: pool.baseMint,
        entryPrice: fillPrice,
        size: riskDecision.sizeSol,
      });
      
      // Simulate price movement & TP/SL
      const exit = simulateExit(pool, position, params.config);
      stats.closePosition(position, exit);
    }
    
    return stats.generateReport();
  }
}
```

### Fee & Slippage Model

Backtest harus realistis soal cost:

```typescript
function simulateExecution(pool: HistoricalPool, side: 'buy' | 'sell', amountSol: number) {
  // 1. Pool fee (0.25% Raydium standard, 1% pump, dst)
  const dexFeeBps = getDexFee(pool.dex);
  
  // 2. Priority fee (assume 0.0001 SOL per tx)
  const priorityFeeSol = 0.0001;
  
  // 3. Slippage berdasarkan liquidity
  const slippagePct = estimateSlippage(amountSol, pool.initialLiquidityUsd);
  
  // 4. Optional: latency-based price drift
  const drift = pool.dex === 'pump' ? 5 : 2; // % slippage tambahan dari latency
  
  const effectiveAmount = amountSol * (1 - dexFeeBps/10000 - slippagePct/100 - drift/100);
  
  return {
    netAmount: effectiveAmount - priorityFeeSol,
    cost: amountSol - effectiveAmount + priorityFeeSol,
  };
}
```

**Calibration**: setelah jalan paper trade, compare actual vs simulated slippage. Adjust model.

### Metrics

Output backtest:

```
══════════════════════════════════════════
   BACKTEST REPORT
   Period: 2025-12-01 → 2025-12-31
   Config: strict-profile-v2.yaml
══════════════════════════════════════════

Pools scanned:        12,847
Filter rejected:      12,234  (95.2%)
Risk rejected:           412
Trades executed:         201

  Wins:                  87  (43.3%)
  Losses:               101  (50.2%)
  Breakeven:             13  ( 6.5%)

P&L:
  Total:              +2.34 SOL
  Avg win:            +0.082 SOL
  Avg loss:           -0.041 SOL
  Best trade:         +0.31 SOL (BONK-like)
  Worst trade:        -0.08 SOL (rug)
  
Risk metrics:
  Max drawdown:        -0.42 SOL (17.9%)
  Sharpe ratio:        1.42
  Profit factor:       1.73
  Win/loss ratio:      2.00
  
Hold time:
  Avg:                 6m 12s
  Median:              3m 45s
  
Filter breakdown (rejects):
  mint_authority_active:    4,891 (40.0%)
  lp_not_burned:            3,012 (24.6%)
  top10_concentration:      1,890 (15.5%)
  honeypot_sell_fails:        823 ( 6.7%)
  low_liquidity:              789 ( 6.5%)
  ...
```

### What to Optimize For

❌ **Jangan optimize total P&L saja** — bisa over-fit
✅ **Optimize**:
- **Profit factor** > 1.5
- **Win/loss ratio** > 1.5 (avg win lebih besar dari avg loss)
- **Max drawdown** < 30% bankroll
- **Sharpe** > 1.0

### Over-Fitting Trap

Backtest yang terlalu tuned ke historical data akan gagal di live.

**Anti-overfit**:
1. **Split data**: 80% training (tune di sini), 20% holdout (validate)
2. **Multi-period**: jalan backtest di 3-4 periode berbeda (bull, bear, sideways)
3. **Sensitivity check**: ubah TP% ±5%, hasilnya berubah drastis? → over-fit
4. **Simplicity bias**: jangan tuning 20 parameter, pilih 5-6 saja

---

## Paper Trading Engine

### Architecture

Sama persis dengan live mode, kecuali:
- `Executor` di-swap dengan `PaperExecutor`
- Wallet balance disimulasikan di DB
- Tidak ada real tx di-submit

### Paper Executor

```typescript
class PaperExecutor implements Executor {
  async executeBuy(params: BuyParams): Promise<TradeResult> {
    // 1. Get current real pool state
    const poolInfo = await dexAdapter.getPoolInfo(params.poolAddress);
    const currentPrice = poolInfo.price;
    
    // 2. Simulate fill delay (realistic 2-3s)
    await sleep(2000 + Math.random() * 1000);
    
    // 3. Get price after delay (real, current)
    const fillPrice = (await dexAdapter.getPoolInfo(params.poolAddress)).price;
    
    // 4. Apply realistic slippage
    const slippage = estimateSlippage(params.amountSol, poolInfo.liquidityUsd);
    const effectiveFillPrice = fillPrice * (1 + slippage);
    
    // 5. Calculate tokens received
    const tokensReceived = params.amountSol / effectiveFillPrice;
    
    // 6. Apply fees
    const dexFee = params.amountSol * 0.0025;
    const priorityFee = 0.0001;
    
    // 7. Update virtual wallet
    await this.virtualWallet.deductSol(params.amountSol + priorityFee);
    await this.virtualWallet.addToken(params.mint, tokensReceived);
    
    // 8. Log as if real
    await db.trade.create({
      data: {
        mode: 'paper',
        side: 'buy',
        amountIn: params.amountSol,
        amountOut: tokensReceived,
        priceUsd: effectiveFillPrice,
        status: 'filled',
        // ...
      }
    });
    
    return { signature: `paper-${Date.now()}`, amountOut: tokensReceived };
  }
  
  async executeSell(params: SellParams): Promise<TradeResult> { /* similar */ }
}
```

### Realism Add-Ons

Untuk paper jadi lebih realistis:

1. **Random fail rate**: 5-10% buy gagal (mirip live tx failure)
2. **Variable latency**: 1-5 detik fill, random
3. **Slippage model**: Lebih besar untuk pool kecil
4. **MEV simulation**: 2% buy "kena front-run" → fill di harga lebih tinggi

### Paper Trading Checklist

Sebelum graduate ke live, paper trade harus:

- [ ] Minimum 1 minggu duration
- [ ] Minimum 30 trades executed
- [ ] Win rate ≥ 35%
- [ ] Profit factor > 1.3
- [ ] Max drawdown < 25%
- [ ] Zero unexplained crashes / bugs
- [ ] Reviewed semua loss trades manually
- [ ] RPC usage tracked & sustainable
- [ ] Notifications work end-to-end

Kalau ada yang gagal → lanjut paper, jangan force ke live.

---

## Comparison: Backtest vs Paper vs Live

Diharapkan profit konsisten degrade di setiap tahap:

| Metric | Backtest | Paper | Live (expected) |
|--------|----------|-------|-----------------|
| Win rate | 50% | 42% | 35% |
| Avg profit per trade | +8% | +6% | +4% |
| Max drawdown | 15% | 22% | 30% |

Kalau gap **terlalu besar** antara backtest dan paper → backtest over-fit / model salah.

Kalau gap **terlalu kecil** → mungkin paper engine kurang realistis.

---

## Reporting

Tiap mode auto-generate report:

```bash
pnpm cli report --mode=backtest --run-id=abc123
pnpm cli report --mode=paper --period=last-7d
```

Output: PDF/HTML dengan grafik P&L, trade list, filter analysis.

---

## Iteration Loop

```
[Backtest] ──► tune config ──► [Backtest again]
   │
   ▼ (config promising)
[Paper trade 1 week]
   │
   ├─ result bad ──► back to [Backtest]
   │
   ▼ (paper OK)
[Live small capital, 0.1-0.5 SOL]
   │
   ├─ result bad ──► back to [Paper] (debug execution)
   │
   ▼ (live OK)
[Scale slowly]
```

Setiap iterasi: log semua, jangan delete data lama. Comparison antar-iterasi adalah aset paling berharga.

---

## Pitfalls

- ❌ **Backtest "too good"** — biasanya overfit atau data leakage
- ❌ **Skip paper trade** — confidence palsu dari backtest = mahal
- ❌ **Paper trade pakai parameter beda dari yang akan live** — tidak valid
- ❌ **Backtest dengan data 1 minggu** — terlalu pendek, butuh minimum 1 bulan multi-market-condition
- ✅ **Save semua run** — bandingan over time adalah signal terkuat