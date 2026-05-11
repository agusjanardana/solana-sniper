# 14 — Strategy Tuning

Cara iterate strategi: log eksperimen, evaluate, dan systematic improvement.

---

## Philosophy

Sniper bot personal-use **bukan** "set and forget". Market kondisi berubah, scammer adapt, profitable pattern decay.

Tuning adalah **continuous process**, bukan event one-time.

**Rule #1**: Setiap perubahan strategi harus terukur. Kalau Anda tidak bisa quantify hasilnya, Anda tidak tahu apakah lebih baik atau lebih buruk.

---

## Tuning Cycle

```
[Hypothesis] → [Backtest] → [Paper Trade] → [Small Live] → [Evaluate]
      ▲                                                          │
      └──────────────────────────────────────────────────────────┘
                    iterate based on data
```

**Setiap perubahan signifikan**: minimum 2 minggu data sebelum conclude.

---

## What to Track per Experiment

```typescript
interface Experiment {
  id: string;
  name: string;
  hypothesis: string;
  
  // What changed
  configDiff: Record<string, [oldValue, newValue]>;
  
  // When
  startedAt: Date;
  endedAt?: Date;
  
  // Where
  mode: 'backtest' | 'paper' | 'live';
  dataset?: string; // untuk backtest
  
  // Results
  metrics: {
    trades: number;
    winRate: number;
    pnlSol: number;
    avgWinSol: number;
    avgLossSol: number;
    maxDrawdown: number;
    sharpeRatio: number;
    profitFactor: number;
  };
  
  // Conclusion
  outcome: 'promoted' | 'rejected' | 'inconclusive';
  notes: string;
}
```

Simpan ke `docs/experiments/EXP-001-tighter-sl.md`:

```markdown
# EXP-001: Tighter Stop Loss (20% → 15%)

**Hypothesis**: Stop loss lebih tight mengurangi avg loss tanpa banyak mengurangi win rate.

**Changes**:
- `tpsl.stopLossPct`: 20 → 15

**Method**: Backtest Dec 1-31, kemudian paper trade Jan 1-14

## Backtest Results

| Metric | Baseline (SL 20%) | New (SL 15%) | Change |
|--------|-------------------|--------------|--------|
| Trades | 201 | 201 | - |
| Win rate | 43.3% | 38.2% | -5.1pp |
| Avg win | +0.082 SOL | +0.082 SOL | - |
| Avg loss | -0.041 SOL | -0.031 SOL | -24% |
| Total P&L | +2.34 SOL | +2.18 SOL | -7% |
| Max DD | -0.42 SOL | -0.31 SOL | -26% |
| Sharpe | 1.42 | 1.56 | +10% |

## Paper Trade Results
[fill after paper trade]

## Conclusion
[fill after evaluation]

## Decision
[promoted / rejected]
```

---

## Key Metrics

### Primary (yang harus improve)

1. **Profit Factor** = total wins / total losses (absolute)
   - Target: > 1.5
   - Below 1.0 = losing strategy

2. **Sharpe Ratio** = (return - risk_free) / std_dev
   - Target: > 1.0
   - Higher = better risk-adjusted return

3. **Max Drawdown** = peak-to-trough decline
   - Target: < 30% of bankroll
   - Higher = unsustainable

### Secondary (informative)

4. **Win Rate** — bisa rendah tapi profitable kalau avg_win >> avg_loss
5. **Avg Hold Time** — terlalu lama? mungkin SL longgar atau market sideways
6. **Slippage Realized** — terlalu tinggi? execution issue
7. **Filter Pass Rate** — terlalu strict atau loose?

### Anti-Metrics (jangan optimize ini)

❌ **Total P&L absolut** — bisa karena lucky big trade
❌ **Number of trades** — quantity ≠ quality
❌ **Win streak** — random variance

---

## Parameter Categories

### Filter Parameters (tuning frequent)

| Param | Range | Default | Notes |
|-------|-------|---------|-------|
| `minLiquidityUsd` | 1k-50k | 5k | Lower = more pools, more rugs |
| `maxLiquidityUsd` | 50k-1M | 100k | Higher = late entry, less alpha |
| `minHolders` | 5-50 | 20 | Lower = earlier entry, riskier |
| `maxTop10Pct` | 30-70 | 50 | Lower = better distribution |
| `requireMintRenounced` | bool | true | Critical |
| `requireLpBurned` | bool | true | Critical |
| `lpBurnThreshold` | 0.5-1.0 | 0.95 | Lower = more permissive |
| `scoreThreshold` | 40-90 | 60 | Lower = more trades, more noise |

### Risk Parameters

| Param | Range | Default | Notes |
|-------|-------|---------|-------|
| `maxPositionSizeSol` | 0.01-1.0 | 0.05 | Personal: keep small |
| `maxConcurrent` | 1-10 | 3 | Higher = more parallel, harder monitor |
| `dailyLossLimitSol` | 0.1-2.0 | 0.5 | Should be 5-10x position size |
| `cooldownSec` | 0-300 | 60 | Higher = lebih conservative after loss |

### TP/SL Parameters

| Param | Range | Default | Notes |
|-------|-------|---------|-------|
| `takeProfitPct` | 20-200 | 50 | Lower = more wins, smaller |
| `stopLossPct` | 10-40 | 20 | Lower = smaller losses but more |
| `trailingStopPct` | 5-30 | 15 | Activates after breakeven |
| `maxHoldMinutes` | 5-120 | 30 | Force exit if stale |

### Execution Parameters

| Param | Range | Default | Notes |
|-------|-------|---------|-------|
| `slippageBps` | 500-5000 | 1500 | Higher fill rate, worse price |
| `priorityFeeMicroLamports` | 10k-1M | 100k | Higher = faster confirmation |
| `maxRetries` | 1-5 | 3 | Higher = more attempts after fail |

---

## Common Experiments

### Experiment 1: Filter Tightening

**Hypothesis**: Strict filter → less trades, higher win rate.

**Variations**:
- `minHolders: 20 → 30`
- `maxTop10Pct: 50 → 40`
- `scoreThreshold: 60 → 70`

**Evaluate**: Win rate up? Total P&L up? (kalau tidak, terlalu strict)

### Experiment 2: TP Optimization

**Hypothesis**: Lower TP = quicker exits, higher win rate.

**Variations**:
- TP 30%, 50%, 75%, 100%
- Plot: win rate, total P&L, avg hold time

**Watch out**: Lower TP miss big wins (meme bisa 10x), tapi steadier curve.

### Experiment 3: Multi-Tier TP (Partial)

**Hypothesis**: Scale-out di TP1 + TP2 + TP3 lebih bagus dari single TP.

```yaml
partialTakeProfit:
  - { atPct: 30, sellPct: 33 }
  - { atPct: 75, sellPct: 33 }
  - { atPct: 150, sellPct: 34 }
```

**Watch**: lebih kompleks, slippage tax setiap exit.

### Experiment 4: DEX Selection

**Hypothesis**: Pump.fun rug rate lebih tinggi dari Raydium.

```bash
pnpm cli analyze --group-by dex --period 30d
```

Output:
```
DEX               Trades  WR%   AvgP&L   TotalP&L
Raydium AMM v4    142    48%   +0.012    +1.70
Pump.fun           89    32%   -0.003    -0.27
Meteora DLMM       34    44%   +0.008    +0.27
Orca Whirlpool     12    50%   +0.015    +0.18
```

**Action**: Mungkin disable Pump.fun, atau strict filter khusus Pump.

### Experiment 5: Time-of-Day

**Hypothesis**: Performance lebih baik di jam tertentu.

```bash
pnpm cli analyze --group-by hour-of-day
```

Mungkin US market open hours (13:00-21:00 UTC) lebih banyak volume → lebih ramai → lebih banyak peluang & rug. Personal preference.

### Experiment 6: Latency Sensitivity

**Hypothesis**: Trade lebih cepat = better entry price.

Measure: latency P50, P95 vs avg_pnl_pct per latency bucket.

Kalau ada strong negative correlation (slow trades = bad pnl), invest di RPC/execution improvement.

---

## A/B Testing (Advanced)

Run 2 strategi paralel:

```yaml
# config.yaml
abTest:
  enabled: true
  strategies:
    - name: "control"
      weight: 50
      config: { filter: strict, tp: 50 }
    - name: "experiment_a"
      weight: 50
      config: { filter: strict, tp: 75 }
```

Setiap pool yang lolos filter base, random assign ke strategy. Track per-strategy metrics.

**Pros**: Fair comparison di market kondisi yang sama
**Cons**: Butuh banyak trades untuk statistical significance (≥100 per arm)

---

## Statistical Significance

Jangan conclude dari 10 trades:

| Sample Size | Confidence |
|-------------|------------|
| < 30 | Anecdotal, jangan promote ke live |
| 30-100 | Suggestive, OK promote ke paper |
| 100-300 | Meaningful, OK promote ke small live |
| 300+ | Statistically significant |

**Rule of thumb**: Setelah hypothetical 2 minggu paper trade dengan ≥50 trades → boleh proceed.

### Sharpe Ratio Significance

Bahkan Sharpe 2.0 di backtest bisa noise. Minimum threshold:
- Backtest 1 bulan → Sharpe > 1.5
- Paper trade 2 minggu → Sharpe > 1.0
- Live 1 bulan → Sharpe > 0.5

Sharpe biasanya **degrade** dari backtest ke paper ke live. Ini normal.

---

## Avoiding Overfitting

❌ **Sign of overfitting**:
- Backtest sangat bagus (Sharpe 3+) tapi paper trade tidak
- Tuning 1 parameter mengubah hasil drastis
- Strategy bagus di 1 periode tapi tidak periode lain
- Anda punya 15+ parameter yang di-tune

✅ **Anti-overfitting**:
- **Hold-out set**: Backtest di 80% data, validate di 20% terakhir (yang belum pernah di-touch)
- **Walk-forward**: Train di month 1, test di month 2; train di month 1-2, test di month 3; etc
- **Parameter parsimony**: 5-7 params max
- **Cross-validate**: Apakah strategy bagus juga di periode lain?
- **Sanity check**: Apakah hasil masuk akal secara intuitif?

---

## Tracking Dashboard

Halaman `/experiments` di dashboard:

```
┌─────────────────────────────────────────────────────────────┐
│ Experiments                                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│ Active:                                                      │
│ • EXP-007: Multi-tier TP        Started: 7d ago             │
│   Live • 42 trades • +0.18 SOL • Sharpe 1.21                │
│   [View Details]                                             │
│                                                              │
│ • EXP-008: Stricter Pump filter Started: 3d ago             │
│   Paper • 23 trades • +0.04 SOL                             │
│   [View Details]                                             │
│                                                              │
│ Completed (last 30d):                                        │
│ • EXP-006: Trailing 20%→15%     Promoted ✅                 │
│ • EXP-005: Disable Pump          Rejected ❌                │
│ • EXP-004: TP 50%→75%            Inconclusive ⏸             │
│                                                              │
│ [+ New Experiment]                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Config Versioning

Setiap promoted experiment = config version bump:

```yaml
# config.yaml
version: "v0.7.2"
versionHistory:
  - { version: "v0.7.2", date: "2026-01-15", change: "Trailing 15%" }
  - { version: "v0.7.1", date: "2026-01-08", change: "Disable Meteora DBC" }
  - { version: "v0.7.0", date: "2026-01-01", change: "Multi-tier TP" }
```

Trades di-tag dengan config version:

```typescript
await db.trade.create({
  data: {
    ...trade,
    configVersion: 'v0.7.2',
  }
});
```

Memungkinkan analisis per-version.

---

## Quarterly Review

Setiap 3 bulan, review komprehensif:

- [ ] Total P&L trend
- [ ] Strategy version yang paling profitable
- [ ] DEX yang paling profitable / merugi
- [ ] Pattern: jam, hari, market condition
- [ ] Filter reject reasons: ada yang baru muncul?
- [ ] Lessons dari incidents
- [ ] Update strategy berdasarkan learnings

---

## Pitfalls

- ❌ **Tune setelah loss** — emotional, not data-driven
- ❌ **Change banyak parameter sekaligus** — tidak tahu mana yang impact
- ❌ **Promote ke live tanpa paper trade** — confidence palsu dari backtest
- ❌ **Stop tracking after live** — strategy yang bagus 6 bulan lalu bisa busuk sekarang
- ❌ **Optimize for max P&L** — terbukti over-fit
- ✅ **Document every experiment** — bahkan yang gagal
- ✅ **Small incremental changes** — 1 perubahan, 1 evaluasi
- ✅ **Trust the process, not gut feelings**