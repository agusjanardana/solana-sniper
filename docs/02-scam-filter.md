# 02 — Scam Filter Engine

Memfilter token rug/scam **sebelum** Risk Engine. Ini layer paling kritis — sniper bot tanpa filter kuat = ATM untuk scammer.

---

## Design Philosophy

1. **Multi-layer**: Hard rules (instant reject) + soft scoring (threshold-based)
2. **Cheap-first**: Cek yang murah & cepat dulu (RPC call sederhana), baru cek mahal (holder analysis)
3. **Cacheable**: Sekali fail, simpan di DB, jangan re-check
4. **Configurable**: Strict/medium/aggressive profile via config

---

## Filter Pipeline

```
Pool detected
    │
    ▼
[Layer 1: Hard Rules]  ─── reject? ──► SKIP (log reason)
    │ pass
    ▼
[Layer 2: On-Chain Checks]  ─── reject? ──► SKIP
    │ pass
    ▼
[Layer 3: Holder Analysis]  ─── reject? ──► SKIP
    │ pass
    ▼
[Layer 4: Honeypot Simulation]  ─── reject? ──► SKIP
    │ pass
    ▼
[Layer 5: Scoring]
    │
    ▼
score >= threshold? ──► APPROVED for Risk Engine
                    └─► SKIP
```

---

## Layer 1: Hard Rules (Instant Reject)

Cek tercepat. Pakai data dari pool creation event.

| Rule | Reject jika | Reasoning |
|------|-------------|-----------|
| Min liquidity | `liquidityUsd < $5,000` | Liquidity terlalu rendah = high slippage, manipulatable |
| Max liquidity | `liquidityUsd > $100,000` | Sudah terlalu late, alpha hilang |
| Quote token | Bukan SOL/USDC/WSOL | Hindari pasangan aneh |
| Pool age | `> 5 menit` saat di-scan | Late entry, kemungkinan sudah pump-dump |
| Blacklist | Creator wallet di blacklist | Repeat scammers |

**Implementasi**: Single function, sync, no RPC call. Output: `{ pass: boolean, reason?: string }`.

---

## Layer 2: On-Chain Token Checks

Cek metadata token via RPC. Ini lapisan paling penting untuk rug detection.

### 2.1 Mint Authority

```typescript
const mintInfo = await getMint(connection, mintPubkey);
if (mintInfo.mintAuthority !== null) {
  return { pass: false, reason: 'mint_authority_active' };
}
```

**Kenapa?** Mint authority = creator bisa mint token baru kapan saja → infinite dilution rug.

**Strict mode**: reject. **Aggressive mode**: warn only.

### 2.2 Freeze Authority

```typescript
if (mintInfo.freezeAuthority !== null) {
  return { pass: false, reason: 'freeze_authority_active' };
}
```

**Kenapa?** Freeze authority = creator bisa freeze wallet Anda → tidak bisa jual.

**Strict mode**: reject. **Selalu rekomendasi reject ini.**

### 2.3 LP Token Burn Status

LP token harus burned/locked supaya creator tidak bisa rug-pull liquidity.

```typescript
// Untuk Raydium AMM v4
const lpMint = poolInfo.lpMint;
const lpSupplyInfo = await getMint(connection, lpMint);
const burnedAccount = await connection.getTokenAccountBalance(BURN_ADDRESS);
const burnedPct = burnedAccount.amount / lpSupplyInfo.supply;

if (burnedPct < 0.95) {
  return { pass: false, reason: 'lp_not_burned' };
}
```

**Threshold**: minimum 95% LP burned atau locked di service terpercaya (PinkLock, Team Finance).

### 2.4 Token Metadata Sanity

- Punya metadata? (Metaplex)
- Ada nama & simbol?
- URI metadata accessible?
- **Trick check**: simbol/nama mengandung Unicode confusables (cyrillic 'а' vs latin 'a')?

```typescript
const NORMALIZE = (s: string) => s.normalize('NFKD').replace(/[\u0300-\u036f]/g, '');
if (hasConfusables(token.symbol)) {
  return { pass: false, reason: 'unicode_confusable' };
}
```

---

## Layer 3: Holder Analysis

Distribusi holder = signal terkuat untuk detect "creator dump"-ready token.

### 3.1 Top Holder Concentration

```typescript
const holders = await getTopHolders(mint, 10);
const top1Pct = holders[0].percentage;
const top10Pct = holders.slice(0, 10).reduce((s, h) => s + h.percentage, 0);

if (top1Pct > 15) return { pass: false, reason: 'top1_too_concentrated' };
if (top10Pct > 50) return { pass: false, reason: 'top10_too_concentrated' };
```

**Catatan**: Kecualikan LP pool address, burn addresses (`1nc1nerator11111111111111111111111111111111`), dan known CEX wallets.

### 3.2 Minimum Holders

```typescript
if (holders.length < 20) {
  return { pass: false, reason: 'too_few_holders' };
}
```

**Threshold**: ≥20 holders dalam 5 menit pertama = sehat. <20 = potensi token wash trading sendiri.

### 3.3 Bundled Buy Detection

Cek apakah top holders beli di transaksi yang sama / blok berurutan (bundling = setup pump & dump).

```typescript
const firstBuys = await getFirstBuys(mint, 20);
const sameSlot = firstBuys.filter(b => b.slot === firstBuys[0].slot).length;

if (sameSlot >= 10) {
  return { pass: false, reason: 'bundled_buys' };
}
```

---

## Layer 4: Honeypot Simulation

**Sangat penting**. Token honeypot = bisa buy tapi tidak bisa sell (creator pakai transfer fee 100% atau whitelist).

### Approach: Simulate Sell Tx

```typescript
// 1. Build buy tx (jangan submit)
const buyTx = await buildBuyTx({ mint, amount: 0.001 });

// 2. Simulate (RPC simulateTransaction)
const buyResult = await connection.simulateTransaction(buyTx);
if (buyResult.value.err) {
  return { pass: false, reason: 'buy_simulation_failed' };
}

// 3. Build sell tx based on expected output
const sellTx = await buildSellTx({ mint, amount: expectedOut });
const sellResult = await connection.simulateTransaction(sellTx);
if (sellResult.value.err) {
  return { pass: false, reason: 'honeypot_sell_fails' };
}

// 4. Check expected sell output vs buy input
const slippage = (buyAmount - sellAmount) / buyAmount;
if (slippage > 0.30) {
  return { pass: false, reason: 'excessive_transfer_tax' };
}
```

**Cost**: ~2 RPC simulate calls per pool. Ini lebih mahal, jadi pakai setelah layer 1-3 pass.

### Alternative: 3rd Party Honeypot APIs

- [rugcheck.xyz API](https://rugcheck.xyz/) — gratis dengan rate limit
- [solsniffer.com](https://solsniffer.com/) — scoring lengkap
- [GoPlus Labs Solana](https://gopluslabs.io/) — token security API

Tradeoff: external dependency vs self-built. Recommended: bikin sendiri layer 1-3, pakai rugcheck sebagai cross-validation.

---

## Layer 5: Scoring

Untuk token yang lolos hard rules tapi belum 100% clear, pakai weighted scoring.

```typescript
interface ScoreInputs {
  liquidityUsd: number;
  holderCount: number;
  top10Pct: number;
  ageMinutes: number;
  hasSocials: boolean;        // X, telegram di metadata
  creatorHistoryScore: number; // 0-1, dari riwayat creator wallet
}

function calculateScore(i: ScoreInputs): number {
  let score = 0;

  // Liquidity sweet spot
  if (i.liquidityUsd >= 10_000 && i.liquidityUsd <= 50_000) score += 25;
  else if (i.liquidityUsd >= 5_000) score += 15;

  // Holder count
  if (i.holderCount >= 50) score += 20;
  else if (i.holderCount >= 30) score += 15;
  else if (i.holderCount >= 20) score += 10;

  // Distribution
  if (i.top10Pct < 25) score += 20;
  else if (i.top10Pct < 40) score += 10;

  // Age (sweet spot: 30s - 3min)
  if (i.ageMinutes >= 0.5 && i.ageMinutes <= 3) score += 15;

  // Socials
  if (i.hasSocials) score += 10;

  // Creator history
  score += i.creatorHistoryScore * 10;

  return Math.min(score, 100);
}
```

**Threshold default**: `score >= 60` → approved.

---

## Filter Profiles

Tiga profile untuk berbagai market condition:

### Strict (default)
- Semua hard rules aktif
- Mint & freeze authority WAJIB renounced
- LP burn ≥ 95%
- Top 10 holders < 50%
- Min holders 20
- Honeypot simulation aktif
- Score threshold: 70

### Medium
- Mint authority WAJIB renounced
- Freeze authority opsional (warn)
- LP burn ≥ 80%
- Top 10 holders < 60%
- Score threshold: 60

### Aggressive
- Hanya hard rules + honeypot
- Skip holder analysis (faster)
- Score threshold: 50
- **Risiko tinggi, hanya untuk eksperimen kecil**

---

## Output Format

Filter selalu return:

```typescript
interface FilterResult {
  pass: boolean;
  reason?: string;          // jika fail
  score?: number;           // 0-100
  details: {
    layer1: RuleResult[];
    layer2: RuleResult[];
    layer3: RuleResult[];
    layer4: RuleResult[];
    finalScore: number;
  };
  durationMs: number;
}
```

Setiap hasil disimpan di DB untuk analisis: "filter mana yang paling sering reject?", "apakah filter terlalu strict?".

---

## Continuous Improvement

Filter bukan static. Workflow:

1. Log semua decision (pass/fail + reason)
2. Setelah 100+ pool processed, analisis:
   - False positive: token yang di-skip tapi ternyata pump → kenapa di-skip?
   - False negative: token yang lolos tapi rug → rule apa yang kurang?
3. Adjust threshold di `config.yaml`
4. Re-run backtest untuk validate

---

## Pitfalls

- ❌ **Jangan trust 100% pada 3rd party API** — bisa down, bisa salah, bisa rate-limited
- ❌ **Jangan skip honeypot check** — meskipun mahal, ini layer paling kritis
- ❌ **Jangan terlalu strict di awal** — paper trade dulu untuk calibrate threshold
- ✅ **Cache hasil filter** — token yang sudah di-reject tidak perlu di-check ulang