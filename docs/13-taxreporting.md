# 13 — Tax Reporting

Track trade untuk keperluan pajak. **Disclaimer**: Ini bukan tax advice. Konsultasi dengan akuntan/konsultan pajak di yurisdiksi Anda.

---

## Why This Matters

Di banyak negara (termasuk Indonesia per regulasi terbaru), profit dari crypto trading dikenakan pajak. Track yang rapi sejak awal jauh lebih mudah daripada reconcile bertahun-tahun kemudian.

**Indonesia spesifik**: Per PMK No. 68/PMK.03/2022, transaksi crypto kena PPN 0.11% dan PPh 0.1% dari nilai transaksi (final). Aturan bisa berubah — selalu cek versi terbaru.

---

## Data yang Perlu Di-Track

Untuk setiap trade:

1. **Timestamp** (UTC dan local)
2. **Token** (mint address, symbol)
3. **Side** (buy/sell)
4. **Amount SOL** (in)
5. **Amount token** (out)
6. **Price USD** at execution time
7. **Price SOL** at execution time
8. **Fees** (DEX fee, priority fee, Jupiter fee)
9. **Transaction signature** (audit trail on-chain)
10. **Cost basis** (untuk sell: berapa cost SOL untuk dapat token ini)

Untuk masuk/keluar wallet:
1. **Deposit** (dari mana, jumlah, harga SOL saat itu)
2. **Withdrawal** (ke mana, jumlah, harga SOL saat itu)

---

## Database Schema Addition

Tambah ke Prisma schema:

```prisma
model TaxLot {
  id              String   @id @default(uuid())
  tokenMint       String   @db.VarChar(64)
  
  // Lot info (1 lot = 1 pembelian)
  acquiredAt      DateTime
  amountTokens    String   // bigint
  costSol         Float    // total SOL spent including fees
  costUsd         Float    // USD value at acquisition
  
  // Remaining (untuk partial sells)
  remainingTokens String
  status          LotStatus  // OPEN, PARTIAL, CLOSED
  
  // Linked trade
  buyTradeId      String
  
  // FIFO ordering
  fifoIndex       Int
  
  createdAt       DateTime @default(now())
  
  disposals       TaxDisposal[]
  
  @@index([tokenMint, status])
  @@index([acquiredAt])
}

model TaxDisposal {
  id              String   @id @default(uuid())
  tokenMint       String   @db.VarChar(64)
  
  disposedAt      DateTime
  amountTokens    String
  proceedsSol     Float    // SOL received from sell
  proceedsUsd     Float    // USD value
  
  // Cost basis (dari lots yang dipakai)
  costBasisSol    Float
  costBasisUsd    Float
  
  // P&L
  gainLossSol     Float
  gainLossUsd     Float
  holdDays        Int      // untuk long-term vs short-term
  
  // Source
  sellTradeId     String
  lotsUsed        Json     // [{ lotId, amount, costPerUnit }]
  
  createdAt       DateTime @default(now())
  
  @@index([tokenMint])
  @@index([disposedAt])
}

model WalletEvent {
  id              String   @id @default(uuid())
  walletAddress   String   @db.VarChar(64)
  type            WalletEventType  // DEPOSIT, WITHDRAWAL
  
  amountSol       Float
  priceUsd        Float    // USD per SOL at time of event
  valueUsd        Float
  
  fromAddress     String?
  toAddress       String?
  txSignature     String   @unique
  
  notes           String?  @db.Text
  
  occurredAt      DateTime
  recordedAt      DateTime @default(now())
  
  @@index([walletAddress])
  @@index([type])
  @@index([occurredAt])
}

enum LotStatus {
  OPEN
  PARTIAL
  CLOSED
}

enum WalletEventType {
  DEPOSIT
  WITHDRAWAL
  STAKING_REWARD
  AIRDROP
}
```

---

## Accounting Method

### FIFO (First In, First Out) — Default Recommendation

Saat sell, lots yang paling lama dibeli yang di-dispose duluan.

**Pros**:
- Simple, predictable
- Default di banyak yurisdiksi (Indonesia, US)
- Tax-friendly di market rising (cost basis terendah → gain lebih besar, tapi predictable)

**Implementation**:

```typescript
async function recordSell(sellTrade: Trade) {
  const tokensToDispose = BigInt(sellTrade.amountInTokens);
  const proceedsSol = sellTrade.amountOutSol - sellTrade.feesSpentSol;
  const priceUsd = await getSolPriceUsd(sellTrade.confirmedAt);
  
  // Get open lots, oldest first (FIFO)
  const lots = await db.taxLot.findMany({
    where: { 
      tokenMint: sellTrade.tokenMint, 
      status: { in: ['OPEN', 'PARTIAL'] }
    },
    orderBy: { fifoIndex: 'asc' },
  });
  
  let remaining = tokensToDispose;
  let totalCostBasisSol = 0;
  const lotsUsed: any[] = [];
  
  for (const lot of lots) {
    if (remaining === 0n) break;
    
    const lotRemaining = BigInt(lot.remainingTokens);
    const fromThisLot = lotRemaining < remaining ? lotRemaining : remaining;
    
    // Proportional cost basis
    const costPerUnit = lot.costSol / Number(BigInt(lot.amountTokens));
    const costFromThisLot = costPerUnit * Number(fromThisLot);
    totalCostBasisSol += costFromThisLot;
    
    lotsUsed.push({
      lotId: lot.id,
      amount: fromThisLot.toString(),
      costPerUnit,
    });
    
    // Update lot
    const newRemaining = lotRemaining - fromThisLot;
    await db.taxLot.update({
      where: { id: lot.id },
      data: {
        remainingTokens: newRemaining.toString(),
        status: newRemaining === 0n ? 'CLOSED' : 'PARTIAL',
      },
    });
    
    remaining -= fromThisLot;
  }
  
  if (remaining > 0n) {
    throw new Error(`Not enough lots to cover sell. Missing: ${remaining}`);
  }
  
  // Calculate hold days (weighted average)
  const oldestLotDate = lots[0]?.acquiredAt;
  const holdDays = Math.floor(
    (sellTrade.confirmedAt!.getTime() - oldestLotDate.getTime()) / 86400_000
  );
  
  // Create disposal record
  await db.taxDisposal.create({
    data: {
      tokenMint: sellTrade.tokenMint,
      disposedAt: sellTrade.confirmedAt!,
      amountTokens: tokensToDispose.toString(),
      proceedsSol,
      proceedsUsd: proceedsSol * priceUsd,
      costBasisSol: totalCostBasisSol,
      costBasisUsd: totalCostBasisSol * priceUsd, // simplified, ideally use historical
      gainLossSol: proceedsSol - totalCostBasisSol,
      gainLossUsd: (proceedsSol - totalCostBasisSol) * priceUsd,
      holdDays,
      sellTradeId: sellTrade.id,
      lotsUsed,
    },
  });
}
```

### LIFO (Last In, First Out) — Alternative

Some yurisdictions allow. Tidak default di Indonesia.

### Specific Identification

Pilih lot mana yang di-dispose secara manual. Complex, biasanya untuk minimize tax.

**Untuk sniper bot personal, FIFO sudah cukup.**

---

## Price Data

Untuk akurasi USD valuation, butuh SOL/USD price di setiap timestamp.

### Source

```typescript
class HistoricalPriceFetcher {
  // Coingecko free API: 5-15 min granularity
  async getSolUsdAt(timestamp: Date): Promise<number> {
    const date = timestamp.toISOString().split('T')[0];
    const cached = await redis.get(`price:sol:${date}`);
    if (cached) return JSON.parse(cached);
    
    const url = `https://api.coingecko.com/api/v3/coins/solana/history?date=${date}`;
    const res = await fetch(url);
    const data = await res.json();
    const price = data.market_data.current_price.usd;
    
    await redis.set(`price:sol:${date}`, JSON.stringify(price), 'EX', 86400 * 30);
    return price;
  }
}
```

Untuk granularity lebih tinggi (per-minute), bayar Coingecko Pro atau pakai Binance public API.

---

## Reports

### Annual Tax Report

Generate CSV untuk tax filing:

```bash
pnpm cli tax-report --year 2026 --format csv > tax-2026.csv
```

Output structure:

```csv
Date,Type,Token,Symbol,Amount Tokens,Proceeds (SOL),Proceeds (USD),Cost Basis (SOL),Cost Basis (USD),Gain/Loss (SOL),Gain/Loss (USD),Hold Days,Tx Signature
2026-01-15,SELL,DezX...B263,BONK,1234567,0.0541,7.65,0.05,7.10,0.0041,0.55,0,5x8...
2026-01-15,BUY,DezX...B263,BONK,1234567,-0.0500,-7.10,,,,,,3y2...
2026-01-16,SELL,WIF1...,WIF,890,0.0451,6.40,0.05,7.10,-0.0049,-0.70,0,8u9...
...
```

### Monthly Summary

```
Tax Year 2026 - January Summary

Transactions:
  Buys:        237
  Sells:       189
  Net trades:  189 (closed positions)

Volume:
  Total buy USD:    $13,247
  Total sell USD:   $14,123
  Net gain USD:     $876

Realized Gains:
  Short-term (< 1y):   $876
  Long-term (>= 1y):   $0
  
Fees:
  DEX fees:         $52.30
  Priority fees:    $12.45
  Total:            $64.75

Wallet movements:
  Deposits:    $500 (Jan 5)
  Withdrawals: $300 (Jan 28)
```

### Per-Token Report

```bash
pnpm cli tax-report --token DezX...B263 --year 2026
```

---

## Indonesia Specific (Per Current Regulation)

**Disclaimer**: Konsultasi konsultan pajak, regulasi bisa berubah.

### Berdasarkan PMK 68/2022 & follow-up:

1. **PPN 0.11%** dari nilai transaksi (final) — sudah dipotong di exchange. Untuk DEX trading, status compliance masih grey area. Konsultasi.

2. **PPh 0.1%** dari nilai transaksi (final) — sama, biasanya dipotong di exchange.

3. **Untuk DEX (off-exchange)**:
   - Capital gain dilaporkan di SPT Tahunan
   - Treatment: penghasilan lain-lain (Pasal 4(2))
   - Bracket pajak ikut PPh OP normal

### Yang Perlu Dilaporkan

- Total volume transaksi (untuk audit trail)
- Total realized gain/loss
- Saldo asset crypto di akhir tahun

### Recommendation

1. **Pisahkan wallet**: 1 wallet untuk trading bot, 1 untuk hodl. Lebih mudah accounting.
2. **Onramp via exchange Indonesia compliant** (Indodax, Tokocrypto, Pintu) → ada bukti PPN/PPh dipotong.
3. **Withdraw to fiat via exchange** → audit trail jelas.
4. **Simpan semua record minimum 10 tahun** (sesuai aturan perpajakan umum).

---

## Integration with Other Tools

### Koinly / CoinTracker / CoinLedger

Bisa export data sniper ke format mereka:

```bash
# Koinly format
pnpm cli tax-report --format koinly --year 2026 > koinly-import.csv
```

Mereka handle perhitungan tax otomatis untuk berbagai yurisdiksi.

### Excel/Google Sheets

```bash
pnpm cli tax-report --format xlsx --year 2026 > tax-2026.xlsx
```

Sheets:
- Sheet 1: Summary
- Sheet 2: All transactions
- Sheet 3: Realized gains by token
- Sheet 4: Wallet movements

---

## CLI Commands

```bash
# Generate tax lots from existing trades (one-time setup)
pnpm cli tax rebuild-lots

# Annual report
pnpm cli tax report --year 2026

# Per-token report
pnpm cli tax report --token <mint> --year 2026

# Verify on-chain vs DB
pnpm cli tax reconcile --wallet $WALLET_ADDRESS

# Record manual deposit/withdrawal
pnpm cli tax record-deposit --amount 1.5 --tx <sig>
pnpm cli tax record-withdrawal --amount 0.5 --tx <sig>
```

---

## Pitfalls

- ❌ **No tracking dari awal** — reconstruct 12 bulan later = nightmare
- ❌ **Mix wallet pribadi dengan bot wallet** — sulit attribution
- ❌ **Tidak record off-chain events** (deposit, withdrawal) — accounting incomplete
- ❌ **Pakai live price at report-time** untuk historical — pakai price at trade-time
- ❌ **Skip fees** — material untuk perhitungan accurate
- ✅ **Backup tax data offline** — kalau DB lost, ini paling penting
- ✅ **Reconcile bulanan** — gampang catch error early
- ✅ **Konsultasi pajak** sebelum tax year berakhir