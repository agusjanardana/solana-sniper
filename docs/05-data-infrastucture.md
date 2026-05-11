# 05 — Data Infrastructure

Bagaimana bot mendapat data: pool baru, transaksi, harga, holder.

---

## Data Sources Overview

| Source | Tujuan | Cost |
|--------|--------|------|
| **Helius RPC + WebSocket** | New pool events, balance, sims | Free tier OK, $99/mo recommended |
| **Helius Webhooks** | Pool creation push | Included Helius |
| **Bitquery GraphQL** | Historical data, holder info | Free tier OK |
| **Jupiter API** | Quotes, swaps | Free, rate limited |
| **DEX Public APIs** | Pool info, price | Free |
| **Birdeye API** | Aggregate price, OHLCV | Free tier OK |
| **Rugcheck API** | Token risk score | Free tier OK |

Untuk personal use, **Helius + Birdeye + Bitquery** sudah cukup.

---

## WebSocket Subscriptions

### Pattern: Log Subscriptions

Pakai `logsSubscribe` di RPC untuk listen instruction tertentu.

```typescript
const subscriptionId = connection.onLogs(
  RAYDIUM_AMM_V4_PROGRAM_ID,
  (logs, ctx) => {
    if (logs.err) return;
    
    // Filter log untuk "initialize" instruction (pool baru)
    const isPoolInit = logs.logs.some(l => 
      l.includes('initialize2') || l.includes('InitializeInstruction2')
    );
    
    if (isPoolInit) {
      // Parse tx untuk dapat pool address
      enqueuePoolDetected({
        signature: logs.signature,
        slot: ctx.slot,
        dex: 'raydium',
      });
    }
  },
  'processed' // commitment
);
```

**Trade-off `processed` vs `confirmed`**:
- `processed`: fastest, ~400ms ahead, tapi bisa di-rollback
- `confirmed`: ~2 detik delay, sudah safe

Untuk **personal use**, pakai `confirmed`. Sub-detik latency tidak critical, tapi false-positive (rollback) bikin rugi waktu/fee.

### Per-DEX Subscription Strategy

| DEX | Program ID | Method |
|-----|-----------|--------|
| **Raydium AMM v4** | `675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8` | logsSubscribe, parse `initialize2` |
| **Raydium CLMM** | `CAMMCzo5YL8w4VFF8KVHrK22GGUsp5VTaW7grrKgrWqK` | logsSubscribe |
| **Raydium CPMM** | `CPMMoo8L3F4NbTegBCKVNunggL7H1ZpdTHKxQB5qKP1C` | logsSubscribe |
| **Pump.fun** | `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P` | logsSubscribe untuk `create` instruction |
| **PumpSwap** | `pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA` | logsSubscribe |
| **Meteora DLMM** | `LBUZKhRxPF3XUpBCjp4YzTKgLccjZhTSDM9YuVaPwxo` | logsSubscribe untuk `initialize_lb_pair` |
| **Meteora DBC** | (Dynamic Bonding Curve) | logsSubscribe |
| **Orca Whirlpools** | `whirLbMiicVdio4qvUfM5KAg6Ct8VwpYzGff3uctyCc` | logsSubscribe untuk `initialize_pool` |

### Helius Enhanced WebSocket

Kalau pakai Helius, gunakan **Enhanced Transactions** untuk filter di server-side:

```typescript
// Lebih efficient dari raw logsSubscribe
const subscription = await heliusWs.subscribe({
  accounts: ['675kPX9MHTjS2zt1qfr1NYHuzeLXfQM9H24wFSUt1Mp8'],
  type: 'enhanced',
  filter: { instructionName: 'initialize2' },
});
```

Mengurangi bandwidth & parsing di sisi client.

---

## Helius Webhooks (Alternative)

Daripada WebSocket persistent connection, bisa pakai webhook HTTP push.

**Pros**:
- Tidak perlu maintain WebSocket connection
- Helius retry kalau endpoint Anda down
- Lebih simple ops untuk personal deployment

**Cons**:
- Latency sedikit lebih tinggi (~200-500ms extra)
- Butuh public endpoint (tunnel via ngrok/Cloudflare untuk local dev)

Setup:
```bash
curl https://api.helius.xyz/v0/webhooks?api-key=YOUR_KEY \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "webhookURL": "https://yourdomain.com/webhook/helius",
    "transactionTypes": ["ANY"],
    "accountAddresses": ["RAYDIUM_PROGRAM_ID", "PUMP_PROGRAM_ID", ...],
    "webhookType": "enhanced"
  }'
```

Endpoint receiver (`services/webhook/`):

```typescript
fastify.post('/webhook/helius', async (req, reply) => {
  const txs = req.body as HeliusTransaction[];
  
  for (const tx of txs) {
    // Cepat-cepat ack, processing async
    enqueueRawTx(tx);
  }
  
  return reply.code(200).send({ ok: true });
});
```

**Recommendation untuk personal use**: WebSocket primary, webhook sebagai backup.

---

## Price Feed Strategy

TP/SL monitoring butuh price update terus-menerus.

### Option 1: Direct Pool State (Recommended)

Polling pool state untuk dapat current price:

```typescript
async function getPriceFromPool(poolAddress: string, dex: string): Promise<number> {
  switch (dex) {
    case 'raydium':
      const poolInfo = await raydiumSdk.getPoolInfo(poolAddress);
      return poolInfo.baseReserve / poolInfo.quoteReserve;
    case 'pump':
      return await pumpSdk.getCurrentPrice(poolAddress);
    // ...
  }
}
```

**Pro**: Real-time, accurate, no third-party dependency.
**Con**: 1 RPC call per pool per poll cycle.

### Option 2: Birdeye Aggregated Price

```typescript
const price = await birdeye.getPrice(mint);
```

**Pro**: Aggregated dari semua DEX, ada chart history, gratis.
**Con**: ~5-15 detik delay, tidak suitable untuk fast TP/SL.

### Recommendation

- **Open position monitoring**: Direct pool state (poll setiap 2 detik)
- **Historical analysis & charts**: Birdeye

---

## Holder Data

Untuk scam filter layer 3, butuh data holder.

### Option 1: Helius `getTokenLargestAccounts` + RPC

```typescript
const largest = await connection.getTokenLargestAccounts(mintPubkey);
// Returns top 20 holders by amount
```

Cukup untuk top-N analysis. Free, fast.

### Option 2: Bitquery GraphQL

Untuk analisis lebih dalam (jumlah holder, distribusi):

```graphql
query TokenHolders($mint: String!) {
  Solana {
    BalanceUpdates(
      where: {
        BalanceUpdate: { Currency: { MintAddress: { is: $mint }}}
      }
      limit: { count: 100 }
    ) {
      BalanceUpdate {
        Account { Address }
        PostBalance
      }
    }
  }
}
```

Free tier: 10K requests/month. Cukup untuk personal use.

### Option 3: Build Sendiri (Untuk Power Users)

Subscribe semua `Transfer` instructions untuk mint Anda track, maintain index sendiri.

Overkill untuk personal use.

---

## Data Storage

### Hot Path (Redis)

- Pool watch list (set of mints being monitored)
- Recent price cache (TTL 5 detik)
- Filter result cache (TTL 1 jam)
- Rate limit counters

### Warm Path (PostgreSQL)

- Tokens, Pools, Trades, Positions (Prisma models di [01-architecture.md](./01-architecture.md))
- Daily aggregates
- Filter decisions log

### Cold Path (Optional: S3/Local Files)

- Historical OHLCV untuk backtest
- Raw tx archive untuk audit

---

## Rate Limiting & Cost Control

### Helius Free Tier Limits

- 100 req/sec
- 100M credits/month
- WebSocket: 5 concurrent

Untuk personal use ini biasanya cukup, tapi **wajib track usage**:

```typescript
class RpcUsageTracker {
  async beforeCall(method: string) {
    await redis.incr(`rpc:${this.bucketKey()}`);
    const used = await redis.get(`rpc:${this.bucketKey()}`);
    
    if (used > this.softLimit) {
      log.warn('Approaching rate limit', { used });
    }
    if (used > this.hardLimit) {
      throw new Error('Rate limit safeguard hit');
    }
  }
}
```

### Cost Estimation (Personal Use)

Asumsi:
- 200 new pool events/hour
- 80% rejected di hard rules (no RPC needed)
- 20% lolos ke deeper checks (5 RPC calls each)
- 5 open positions, polled setiap 2 detik = 9000 calls/hour

Total: ~9200 RPC calls/hour = ~6.6M/month

Masih dalam free tier Helius (100M). Nyaman.

---

## Latency Budget

Untuk personal use, target end-to-end:

```
Pool created on-chain
    │
    ├─ WebSocket delivery: ~200-500ms
    │
[Pool event received]
    │
    ├─ Hard rules: ~10ms (no RPC)
    │
    ├─ On-chain checks: ~300-800ms (3-5 RPC calls)
    │
    ├─ Honeypot sim: ~200-400ms
    │
[Filter passed]
    │
    ├─ Risk engine: ~50ms (mostly DB)
    │
    ├─ Build tx + simulate: ~300-500ms
    │
    ├─ Submit + confirm: ~1500-3000ms
    │
[Position opened]
```

**Total**: 2.5–5 detik dari pool creation ke fill.

Ini kalah cepat dari competitive bots (sub-200ms), tapi cukup untuk personal sniping di multi-DEX dengan filter ketat.

---

## Reconnection & Resilience

### WebSocket Reconnect

```typescript
class ResilientWs {
  private reconnectAttempt = 0;
  
  async connect() {
    try {
      this.ws = new WebSocket(this.url);
      this.ws.onclose = () => this.handleDisconnect();
      this.ws.onerror = (e) => log.error('WS error', e);
      
      // Subscribe ulang setelah connect
      this.ws.onopen = () => {
        this.reconnectAttempt = 0;
        this.resubscribeAll();
      };
    } catch (e) {
      this.handleDisconnect();
    }
  }
  
  private async handleDisconnect() {
    const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempt), 30_000);
    this.reconnectAttempt++;
    
    log.warn(`WS disconnected, retry in ${delay}ms`);
    await sleep(delay);
    await this.connect();
  }
}
```

### Catch-Up After Disconnect

Saat WS reconnect, pool yang muncul saat disconnect bisa terlewat. Strategi:
- Setelah reconnect, query getProgramAccounts untuk pool baru dalam X slots terakhir
- Atau: terima kehilangan event (untuk personal use, OK)

---

## Pitfalls

- ❌ **Jangan pakai public RPC** untuk production — rate limited, no SLA
- ❌ **Jangan trust 1 RPC saja** — selalu siapkan fallback
- ❌ **Jangan parse log dengan regex naif** — pakai SDK parser
- ❌ **Jangan over-subscribe** — terlalu banyak WebSocket channel = bandwidth/parsing bottleneck
- ✅ **Cache aggressive**: Token metadata, pool info, filter result
- ✅ **Monitor connection health**: kalau no event > 30 detik di market jam sibuk, kemungkinan WS dead