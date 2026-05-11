# 03 — Execution Engine

Engine yang submit transaksi buy/sell ke Solana. Untuk personal-use (bukan competitive sniping), prioritas: **reliability > raw speed**.

---

## Design Goals

1. **High fill rate** — tx tidak gagal karena slippage/priority fee issue
2. **Reasonable latency** — target 1–3 detik dari signal ke fill (cukup untuk personal use)
3. **Idempotent** — retry tidak menghasilkan double buy
4. **Pluggable providers** — Jupiter primary, direct DEX fallback, optional Jito

---

## Provider Strategy

### Primary: Jupiter Aggregator v6

Pakai Jupiter sebagai default karena:
- Auto-route ke DEX terbaik (Raydium, Orca, Meteora, dll)
- Handle slippage calculation
- Built-in retry & versioning support
- Maintained by tim besar

```typescript
import { Jupiter } from '@jup-ag/api';

const quote = await jupiter.quoteGet({
  inputMint: SOL_MINT,
  outputMint: targetMint,
  amount: amountLamports,
  slippageBps: 1500, // 15%
  onlyDirectRoutes: false,
  asLegacyTransaction: false,
});

const { swapTransaction } = await jupiter.swapPost({
  swapRequest: {
    quoteResponse: quote,
    userPublicKey: wallet.publicKey.toString(),
    wrapAndUnwrapSol: true,
    prioritizationFeeLamports: computedPriorityFee,
    dynamicComputeUnitLimit: true,
  },
});
```

### Fallback: Direct DEX Calls

Jika Jupiter:
- Token belum ter-index Jupiter (sering untuk pool yang baru lahir <30 detik)
- Route gagal
- Slippage Jupiter terlalu pesimistik

Maka fallback ke direct call:

| DEX | Library |
|-----|---------|
| Raydium | `@raydium-io/raydium-sdk-v2` |
| Pump.fun | Direct anchor call atau `@pump-fun/pump-sdk` |
| Meteora | `@meteora-ag/dlmm` / `@meteora-ag/dynamic-amm-sdk` |
| Orca | `@orca-so/whirlpools-sdk` |

**Strategy**: Try Jupiter dulu (timeout 800ms), jika gagal → direct DEX adapter.

### Optional: Jito Bundles

Untuk personal use, **Jito biasanya overkill**. Tapi kalau Anda ingin coba:

- Untung: lebih kecil kemungkinan front-run, bisa bundle multiple tx
- Rugi: tip Jito tambahan (0.0001–0.001 SOL per bundle)
- Use case: kalau pool sangat hot dan banyak bot lain bersaing

Untuk personal-use default: **disable Jito**, enable hanya untuk pool high-value (score > 85).

---

## Priority Fee Strategy

Solana priority fee = micro-lamports per compute unit, dibayar untuk prioritization.

### Dynamic Priority Fee (Recommended)

```typescript
async function getDynamicPriorityFee(): Promise<number> {
  // Ambil recent prioritization fees
  const recent = await connection.getRecentPrioritizationFees();

  // Hitung percentile
  const fees = recent.map(r => r.prioritizationFee).sort((a, b) => a - b);
  const p75 = fees[Math.floor(fees.length * 0.75)];

  // Multiplier berdasarkan urgency
  const urgencyMultiplier = 1.5; // 1.0 normal, 2.0 hot
  const fee = Math.floor(p75 * urgencyMultiplier);

  // Clamp
  return Math.min(Math.max(fee, 10_000), 1_000_000);
}
```

### Static Priority Fee (Simple Fallback)

Untuk personal use yang tidak butuh perfect timing:

```yaml
execution:
  priorityFeeMode: static
  staticPriorityFeeMicroLamports: 100000  # 0.0001 SOL per 1M CU
```

### Cost Reality

Untuk personal use:
- Buy tx: ~200K compute units
- Priority fee 100,000 micro-lamports = 0.00002 SOL (~$0.005)
- Acceptable cost untuk reliability

---

## RPC Strategy

### Provider Recommendation (Personal Use)

| Provider | Tier | Recommendation |
|----------|------|----------------|
| **Helius** | Free → $99/mo | ⭐ Recommended. WebSocket reliable, enhanced APIs |
| **QuickNode** | $9/mo+ | Solid alternative |
| **Triton** | $$$ | Overkill untuk personal use |
| **Public RPC** | Free | ❌ Jangan, rate limited & slow |

### Multi-RPC Failover

Pakai minimum 2 RPC providers:

```typescript
class RpcManager {
  private primary: Connection;
  private fallback: Connection;

  async send(tx: VersionedTransaction): Promise<string> {
    try {
      return await this.primary.sendTransaction(tx, { 
        skipPreflight: false,
        maxRetries: 0, // kita handle retry sendiri
      });
    } catch (e) {
      log.warn('Primary RPC failed, trying fallback', e);
      return await this.fallback.sendTransaction(tx, { skipPreflight: false });
    }
  }
}
```

### Send to Multiple RPCs (Optional)

Untuk reliability lebih tinggi, send tx ke 2+ RPC sekaligus:

```typescript
async function broadcastTx(tx: VersionedTransaction) {
  const results = await Promise.allSettled([
    primary.sendTransaction(tx),
    fallback.sendTransaction(tx),
  ]);

  // Solana akan dedupe — kalau tx sama, hanya 1 yang executed
  const success = results.find(r => r.status === 'fulfilled');
  if (!success) throw new Error('All RPCs failed');

  return (success as PromiseFulfilledResult<string>).value;
}
```

---

## Transaction Lifecycle

```
1. Build tx
   ├─ Get latest blockhash (commitment: 'confirmed')
   ├─ Add ComputeBudget instructions (priority fee + CU limit)
   ├─ Add swap instructions (from Jupiter atau direct)
   └─ Sign

2. Simulate (optional but recommended)
   ├─ connection.simulateTransaction(tx)
   ├─ Cek error → abort kalau ada
   └─ Cek expected output → cek slippage acceptable

3. Submit
   ├─ Send via RpcManager
   ├─ skipPreflight: false (untuk safety)
   └─ Capture signature

4. Confirm
   ├─ Poll getSignatureStatus tiap 500ms
   ├─ Timeout 30 detik
   └─ Cek status: 'processed' → 'confirmed' → 'finalized'

5. Post-process
   ├─ Parse tx untuk actual amount filled
   ├─ Update DB (status: filled)
   └─ Emit event 'trade.filled'
```

---

## Retry Logic

```typescript
async function executeWithRetry(
  params: TradeParams,
  maxRetries = 3
): Promise<TradeResult> {
  let lastError: Error | undefined;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      // Re-fetch blockhash setiap attempt
      const blockhash = await connection.getLatestBlockhash('confirmed');
      const tx = await buildTx({ ...params, blockhash });
      
      // Re-simulate setiap attempt (price bisa berubah)
      const sim = await connection.simulateTransaction(tx);
      if (sim.value.err) throw new Error(`Simulation failed: ${sim.value.err}`);

      const signature = await rpcManager.send(tx);
      const confirmed = await confirmTx(signature, 30_000);
      
      return { signature, ...confirmed };
    } catch (e) {
      lastError = e as Error;
      log.warn(`Attempt ${attempt + 1} failed`, e);

      // Bedakan retryable vs not
      if (!isRetryable(e)) break;
      
      // Exponential backoff
      await sleep(500 * Math.pow(2, attempt));
    }
  }

  throw lastError;
}

function isRetryable(e: any): boolean {
  const msg = e.message || '';
  // Retryable: blockhash expired, RPC timeout, congestion
  if (msg.includes('blockhash not found')) return true;
  if (msg.includes('timeout')) return true;
  if (msg.includes('429')) return true;
  // Not retryable: slippage exceeded, insufficient funds, honeypot
  if (msg.includes('slippage')) return false;
  if (msg.includes('insufficient')) return false;
  return false;
}
```

---

## Slippage Strategy

### Dynamic Slippage by Pool Age

Pool sangat baru → harga volatile → butuh slippage lebih besar.

```typescript
function getSlippageBps(poolAgeSeconds: number): number {
  if (poolAgeSeconds < 30) return 2500;   // 25%
  if (poolAgeSeconds < 120) return 1500;  // 15%
  if (poolAgeSeconds < 300) return 1000;  // 10%
  return 500;                              // 5%
}
```

### Slippage Sanity Check

Jangan blind submit dengan slippage tinggi — selalu simulate dulu dan compare expected output:

```typescript
const quote = await getQuote({ ...params, slippageBps: 2500 });
const actualPriceImpact = quote.priceImpactPct;

if (actualPriceImpact > 30) {
  // Bahkan dengan slippage 25%, price impact masih 30%+
  // → liquidity sangat tipis, skip
  return { skipped: true, reason: 'extreme_price_impact' };
}
```

---

## Idempotency

Setiap trade order punya `clientOrderId` (UUID generate di sisi caller).

DB constraint:
```sql
CREATE UNIQUE INDEX idx_trades_client_order_id ON trades(client_order_id);
```

Flow:
1. Generate UUID di Risk Engine
2. INSERT row dengan status `pending` + clientOrderId
3. Submit tx
4. UPDATE row dengan signature & status `filled`

Jika retry:
- Cek DB dulu: ada row dengan clientOrderId ini & status `filled`? → skip, sudah done
- Kalau status `pending`: cek di RPC apakah signature pernah confirmed → recover atau resubmit

---

## Compute Unit Optimization

Set CU limit eksplisit (default 200K, biasanya cukup). Pakai `ComputeBudgetProgram`:

```typescript
import { ComputeBudgetProgram } from '@solana/web3.js';

const instructions = [
  ComputeBudgetProgram.setComputeUnitLimit({ units: 300_000 }),
  ComputeBudgetProgram.setComputeUnitPrice({ microLamports: priorityFee }),
  ...swapInstructions,
];
```

CU yang lebih kecil = fee lebih murah, tapi tx bisa gagal kalau swap kompleks. **300K aman untuk Jupiter routes**, **200K untuk direct DEX**.

---

## Observability

Per tx, log:

```json
{
  "event": "trade.executed",
  "clientOrderId": "uuid",
  "signature": "...",
  "mint": "...",
  "side": "buy",
  "amountIn": 0.05,
  "amountOut": 1234.56,
  "priorityFeeMicroLamports": 150000,
  "slippageBps": 1500,
  "actualSlippagePct": 8.2,
  "durationMs": 2134,
  "provider": "jupiter",
  "rpcUsed": "helius-primary",
  "attempts": 1
}
```

Metrics yang penting:
- Fill rate (success / total attempts)
- Latency P50/P95/P99
- Slippage realized (rata-rata vs konfigurasi)
- Cost per trade (priority fee + Jito tip)

---

## Pitfalls

- ❌ **Jangan pakai legacy transaction** — selalu VersionedTransaction
- ❌ **Jangan skip preflight kecuali Anda yakin** — error early > error late
- ❌ **Jangan pakai blockhash lama** — refresh setiap retry
- ❌ **Jangan hardcode priority fee tinggi** — burn money percuma
- ✅ **Always simulate sebelum submit untuk buy pertama kali**
- ✅ **Untuk sell, lebih aggressive** — boleh skip preflight, prioritas exit > sliperage