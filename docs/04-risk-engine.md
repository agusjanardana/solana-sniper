# 04 — Risk Engine + TP/SL Logic

Risk engine = gatekeeper terakhir sebelum execution. Filter sudah bilang "token ini OK", risk engine bilang "tapi apakah kita boleh trade sekarang, dan berapa size-nya?".

**Prinsip utama**: untuk personal use yang fokus profit konsisten, **survival > maximize profit**.

---

## Risk Layers

```
Trade proposal (from filter)
    │
    ▼
[Layer A: Capital Allocation]
    │ approved size & amount
    ▼
[Layer B: Concurrency Limit]
    │ slot available?
    ▼
[Layer C: Daily Limits]
    │ daily loss not hit?
    ▼
[Layer D: Kill Switch]
    │ system not in emergency stop?
    ▼
[Layer E: Cooldown]
    │ not in cooldown after loss?
    ▼
APPROVED → Execution
```

Setiap layer bisa reject. Reject ≠ filter fail — ini "trade OK tapi kondisi sekarang tidak ideal".

---

## Layer A: Capital Allocation

### Position Sizing

Tiga model, pilih sesuai gaya:

#### 1. Fixed Size (Simple, Recommended untuk pemula)

```yaml
risk:
  sizingModel: fixed
  fixedSizeSol: 0.05
```

Setiap trade: 0.05 SOL. Sederhana, predictable.

#### 2. Percentage of Bankroll

```yaml
risk:
  sizingModel: percentage
  pctOfBankroll: 2  # 2% dari total bankroll
```

Sizing menyesuaikan bankroll yang ada. Kalau menang besar → size naik, kalau rugi → size turun.

```typescript
const bankroll = await getBankrollSol(wallet);
const sizeSol = bankroll * (config.pctOfBankroll / 100);
```

#### 3. Kelly-Lite

Adaptive berdasarkan win rate historis. **Hati-hati**, butuh ≥50 trade history dulu.

```typescript
function kellySize(bankroll: number, winRate: number, avgWinPct: number, avgLossPct: number): number {
  // Kelly fraction = (W * winRate - lossRate) / W, where W = avgWin/avgLoss
  const W = avgWinPct / avgLossPct;
  const kellyFraction = (W * winRate - (1 - winRate)) / W;
  
  // Half-Kelly untuk safety
  const safeFraction = Math.max(0, Math.min(kellyFraction * 0.5, 0.05));
  return bankroll * safeFraction;
}
```

### Hard Caps

Apapun model-nya, ada hard limit:

```yaml
risk:
  maxPositionSizeSol: 0.5      # cap absolut per trade
  minPositionSizeSol: 0.01     # tidak worth trading di bawah ini (fee dominasi)
```

---

## Layer B: Concurrency Limit

Berapa posisi terbuka secara bersamaan?

```yaml
risk:
  maxConcurrentPositions: 3
```

**Kenapa penting?**
- Lebih banyak posisi = harder to monitor manually
- Capital fragmented = sulit react ke peluang besar
- Bot bug = lebih banyak posisi yang error

Default: 3 posisi. Naikkan hanya kalau system sudah proven.

```typescript
const openCount = await db.position.count({ where: { status: 'open' }});
if (openCount >= config.maxConcurrentPositions) {
  return { rejected: true, reason: 'concurrency_limit' };
}
```

---

## Layer C: Daily Limits

### Daily Loss Limit

```yaml
risk:
  dailyLossLimitSol: 0.5
```

Kalau kumulatif loss hari ini ≥ limit → stop trading sampai besok 00:00 UTC.

```typescript
const today = startOfDayUtc();
const todayPnl = await db.trade.aggregate({
  where: { createdAt: { gte: today }, status: 'filled' },
  _sum: { pnlSol: true },
});

if ((todayPnl._sum.pnlSol ?? 0) <= -config.dailyLossLimitSol) {
  return { rejected: true, reason: 'daily_loss_hit' };
}
```

### Daily Trade Count Limit

Mencegah over-trading saat tilted:

```yaml
risk:
  maxTradesPerDay: 20
```

### Daily Win Cap (Optional, Counterintuitive but Effective)

Setelah win besar, sering kali trader mulai over-confident & rugi. Cap di sisi atas juga membantu:

```yaml
risk:
  dailyWinCapSol: 1.0  # stop trading kalau profit hari ini sudah > 1 SOL
```

Opsional, tapi pertimbangkan kalau Anda secara psikologis rentan tilt.

---

## Layer D: Kill Switch

Emergency stop. Bisa trigger dari:

### 1. Consecutive Losses

```yaml
risk:
  killSwitchAfterConsecutiveLosses: 5
```

5 loss berurutan → strategy mungkin broken atau market kondisi salah → stop.

### 2. RPC Health

Kalau primary + fallback RPC sama-sama unhealthy → stop. Trading dengan RPC bermasalah = recipe for disaster.

### 3. Wallet Balance Anomaly

```typescript
if (currentBalance < expectedBalance * 0.8) {
  // Balance tiba-tiba turun 20%+ di luar tracked trades
  // → kemungkinan ada drain / bug
  triggerKillSwitch('unexpected_balance_drop');
}
```

### 4. Manual via Dashboard

Tombol big red button di dashboard. Wajib ada.

### Reset Kill Switch

Kill switch hanya reset **manual** lewat CLI/dashboard. Jangan auto-reset.

```bash
pnpm cli kill-switch reset --reason "investigated rpc issue"
```

Reset event log ke DB untuk audit.

---

## Layer E: Cooldown After Loss

Setelah loss, jangan langsung trade lagi:

```yaml
risk:
  cooldownAfterLossSeconds: 60
```

Tujuannya: kalau ada systemic issue (misal RPC lag, market crash), kita tidak terus rugi.

---

## TP/SL Engine

Setelah posisi terbuka, TP/SL engine yang handle exit.

### Exit Conditions

Posisi di-exit jika **salah satu** kondisi terpenuhi:

1. **Take Profit hit**
2. **Stop Loss hit**
3. **Trailing Stop hit** (jika aktif)
4. **Max hold time** terlewati
5. **Manual exit** dari dashboard
6. **Emergency exit** (kill switch)

### Configuration

```yaml
tpsl:
  takeProfitPct: 50          # exit kalau +50%
  stopLossPct: 20            # exit kalau -20%
  trailingStopPct: 15        # trailing 15% dari peak setelah breakeven
  maxHoldMinutes: 30         # exit kalau > 30 menit
  partialTakeProfit:         # optional: scale out
    - { atPct: 30, sellPct: 50 }   # +30% jual 50%
    - { atPct: 100, sellPct: 50 }  # +100% jual sisa 50%
```

### Price Monitoring Loop

```typescript
async function monitorPositions() {
  while (true) {
    const positions = await db.position.findMany({ where: { status: 'open' }});
    
    for (const pos of positions) {
      const currentPrice = await getPrice(pos.tokenMint);
      const pnlPct = ((currentPrice - pos.entryPrice) / pos.entryPrice) * 100;
      
      // Update trailing high
      if (currentPrice > (pos.trailingHigh ?? pos.entryPrice)) {
        await db.position.update({
          where: { id: pos.id },
          data: { trailingHigh: currentPrice },
        });
      }
      
      const exitDecision = evaluateExit(pos, currentPrice, pnlPct);
      if (exitDecision.shouldExit) {
        await emitExitSignal(pos, exitDecision);
      }
    }
    
    await sleep(2000); // poll setiap 2 detik
  }
}
```

**Polling interval**: 2 detik adalah sweet spot untuk personal use. Lebih cepat = lebih banyak RPC call, lebih lambat = exit yang lambat.

### Exit Decision Logic

```typescript
function evaluateExit(pos: Position, currentPrice: number, pnlPct: number): ExitDecision {
  // 1. Stop Loss (paling prioritas)
  if (pnlPct <= -config.stopLossPct) {
    return { shouldExit: true, reason: 'stop_loss', urgency: 'high' };
  }
  
  // 2. Take Profit
  if (pnlPct >= config.takeProfitPct) {
    return { shouldExit: true, reason: 'take_profit', urgency: 'normal' };
  }
  
  // 3. Trailing Stop (aktif hanya setelah breakeven)
  if (pos.trailingHigh && pos.trailingHigh > pos.entryPrice) {
    const dropFromPeak = ((pos.trailingHigh - currentPrice) / pos.trailingHigh) * 100;
    if (dropFromPeak >= config.trailingStopPct) {
      return { shouldExit: true, reason: 'trailing_stop', urgency: 'high' };
    }
  }
  
  // 4. Max hold time
  const ageMinutes = (Date.now() - pos.openedAt.getTime()) / 60_000;
  if (ageMinutes >= config.maxHoldMinutes) {
    return { shouldExit: true, reason: 'max_hold', urgency: 'normal' };
  }
  
  return { shouldExit: false };
}
```

### Urgency-Aware Exit

Sell tx pakai parameter berbeda berdasarkan urgency:

| Urgency | Slippage | Priority Fee | Retry |
|---------|----------|--------------|-------|
| `normal` | 10% | dynamic p75 | 3x |
| `high` | 25% | dynamic p95 + 2x multiplier | 5x |
| `emergency` | 50% | max (1M micro-lamports) | 10x |

Untuk stop-loss & trailing stop (urgency `high`), kita rela bayar fee lebih untuk pastikan keluar.

### Partial Take Profit (Scale-Out)

Untuk balance "ambil profit cepat" vs "let winners run":

```typescript
// Contoh config:
// +30% → sell 50%
// +100% → sell sisa 50%

async function checkPartialTP(pos: Position, pnlPct: number) {
  for (const tier of config.partialTakeProfit) {
    if (pnlPct >= tier.atPct && !pos.partialsTriggered.includes(tier.atPct)) {
      const sellAmount = pos.amountTokens * (tier.sellPct / 100);
      await emitExitSignal(pos, { 
        reason: `partial_tp_${tier.atPct}`,
        amount: sellAmount,
        urgency: 'normal',
      });
      // Mark this tier as triggered
      await markPartialTriggered(pos.id, tier.atPct);
    }
  }
}
```

**Trade-off**: Lebih kompleks tapi smooth out P&L. Recommended kalau Anda sudah comfortable dengan basic TP/SL.

---

## Bankroll Management

Long-term survival:

### Rule 1: Withdraw Profit

Setiap kali bankroll naik 50%, withdraw 50% profit ke cold wallet.

Contoh:
- Mulai: 1 SOL
- Bankroll: 1.5 SOL → withdraw 0.25 SOL (50% dari 0.5 profit) → trade dengan 1.25 SOL
- Bankroll: 1.875 SOL (1.25 × 1.5) → withdraw 0.3125 → trade dengan 1.5625 SOL

Ini mencegah "menang banyak terus balik ke nol" yang umum di degen trading.

### Rule 2: Never Top Up

Kalau bankroll burner wallet habis, **jangan langsung top-up**. Stop, analisis loss, baru top-up dengan modal kecil.

### Rule 3: Track Everything

Setiap deposit/withdraw ke burner wallet → log di DB dengan reason. Jangan campur dengan trade P&L.

---

## Reporting

Daily report (di-push ke Telegram setiap 00:00 lokal):

```
📊 Daily Report - 2026-01-15
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Trades: 12 (8W / 4L)
Win rate: 66.7%
P&L: +0.32 SOL
Best trade: +0.18 SOL (BONK)
Worst trade: -0.05 SOL (RUGCOIN)
Avg hold: 4m 23s
Filter rejects: 247
Kill switch: ❌ not triggered

Top reject reasons:
- mint_authority_active (89)
- lp_not_burned (54)
- top10_too_concentrated (38)
```

---

## Pitfalls

- ❌ **Jangan adjust TP/SL saat posisi terbuka** (kecuali emergency)
- ❌ **Jangan skip stop loss "hanya kali ini"** — slippery slope
- ❌ **Jangan over-leverage** — meme coin sudah leverage natural via volatility
- ✅ **TP lebih kecil > SL lebih kecil** — banyak small wins lebih sehat daripada hope-and-pray big wins
- ✅ **Review SL/TP setting setiap minggu** berdasarkan data backtest