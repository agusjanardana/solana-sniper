# 09 — Notifications

Notifikasi via Telegram + Discord. Untuk personal use, Telegram cukup. Discord sebagai opsi atau backup.

---

## Why Notifications Matter

Bot jalan 24/7 di VPS, Anda tidak bisa terus stare ke dashboard. Notifikasi harus:

1. **Critical alerts** sampai instant (Telegram push notif > email)
2. **Two-way control** — bisa stop bot dari notifikasi (Telegram command)
3. **Not noisy** — kalau setiap pool detected dikirim notif, Anda akan mute = miss real alerts

---

## Alert Categories

Klasifikasi event berdasarkan severity:

| Severity | Channel | Action |
|----------|---------|--------|
| **CRITICAL** | Telegram + sound | Bot stopped, balance drift, RPC down |
| **HIGH** | Telegram | Position closed with big loss, kill switch triggered |
| **NORMAL** | Telegram | Position opened/closed, daily summary |
| **LOW** | Logs only | Filter passes, pool detected, retry attempt |

---

## Telegram Setup

### Create Bot

1. DM @BotFather di Telegram
2. `/newbot` → ikuti instruksi
3. Simpan **bot token** (format: `123456:ABC-DEF...`)
4. Start chat dengan bot Anda
5. Cari chat ID: GET `https://api.telegram.org/bot<TOKEN>/getUpdates`
6. Simpan **chat_id**

### Environment

```env
TELEGRAM_BOT_TOKEN=123456:ABC-DEF...
TELEGRAM_CHAT_ID=-1001234567890
TELEGRAM_ADMIN_USER_IDS=12345678,87654321  # untuk authorized commands
```

### Why Group Chat (Not DM)?

- Bisa silent mute thread tertentu (less critical)
- Bisa tambah admin (kalau ada partner)
- History tetap saat ganti device

---

## Message Templates

### Position Opened

```
🟢 POSITION OPENED

Token: $BONK
Mint: DezX...B263
DEX: Raydium AMM v4

💰 Amount: 0.05 SOL
📊 Entry: $0.000023
🎯 TP: $0.0000345 (+50%)
🛡️ SL: $0.0000184 (-20%)

⏱️ Latency: 2.3s
🔗 Tx: <a href="https://solscan.io/tx/...">View</a>

Filter score: 78/100
Liquidity: $24,500
Holders: 47
```

### Position Closed (Win)

```
✅ POSITION CLOSED (WIN)

Token: $BONK
Hold time: 4m 12s

💰 P&L: +0.024 SOL (+48%)
📊 Exit reason: take_profit

Entry: $0.000023
Exit: $0.000034

🔗 <a href="https://solscan.io/tx/...">Exit Tx</a>

Today: +0.31 SOL (8W/3L)
```

### Position Closed (Loss)

```
🔴 POSITION CLOSED (LOSS)

Token: $RUGCOIN
Hold time: 1m 45s

💸 P&L: -0.010 SOL (-20%)
📊 Exit reason: stop_loss

🔗 <a href="https://solscan.io/tx/...">Exit Tx</a>

Today: +0.30 SOL (8W/4L)
```

### Critical: Kill Switch

```
🚨 KILL SWITCH TRIGGERED 🚨

Reason: consecutive_losses (5)
At: 2026-01-15 14:32:18 UTC

Trading STOPPED.
Manual intervention required.

Recent trades:
- $TOKEN1: -0.012 SOL
- $TOKEN2: -0.008 SOL
- $TOKEN3: -0.015 SOL
- $TOKEN4: -0.005 SOL
- $TOKEN5: -0.020 SOL

Reset with: /reset_killswitch
Or: pnpm cli kill-switch reset
```

### Critical: Balance Drift

```
🚨 BALANCE DRIFT DETECTED 🚨

Wallet: ABC...XYZ
Expected: 0.547 SOL
Actual: 0.312 SOL
Drift: -0.235 SOL

POSSIBLE UNAUTHORIZED ACCESS

Bot stopped automatically.
Investigate IMMEDIATELY.
```

### Daily Summary

```
📊 DAILY REPORT - 2026-01-15

Mode: LIVE

Trades: 12 (8W / 4L)
Win rate: 66.7%
P&L: +0.32 SOL ($45)
Best: +0.18 SOL ($BONK)
Worst: -0.05 SOL ($SCAM)

Avg hold: 4m 23s
Fees spent: 0.024 SOL

Filter stats:
• Pools scanned: 1,247
• Rejected: 1,193 (95.7%)
• Approved: 54
• Executed: 12

Top reject reasons:
• mint_authority_active: 489
• lp_not_burned: 312
• top10_concentrated: 198

Wallet balance: 0.847 SOL
```

---

## Implementation

### Telegram Service

```typescript
// packages/notify/src/telegram.ts
import { Bot, InlineKeyboard } from 'grammy';

export class TelegramNotifier {
  private bot: Bot;
  private chatId: string;
  
  constructor(token: string, chatId: string) {
    this.bot = new Bot(token);
    this.chatId = chatId;
    this.setupCommands();
  }
  
  async send(message: string, options?: { silent?: boolean; buttons?: InlineKeyboard }) {
    return this.bot.api.sendMessage(this.chatId, message, {
      parse_mode: 'HTML',
      disable_notification: options?.silent ?? false,
      reply_markup: options?.buttons,
      link_preview_options: { is_disabled: true },
    });
  }
  
  async sendCritical(message: string) {
    // Send with sound + emoji
    return this.send(`🚨 ${message}`, { silent: false });
  }
  
  private setupCommands() {
    this.bot.command('status', async (ctx) => {
      if (!this.isAdmin(ctx.from?.id)) return;
      const status = await getBotStatus();
      await ctx.reply(formatStatus(status));
    });
    
    this.bot.command('positions', async (ctx) => {
      if (!this.isAdmin(ctx.from?.id)) return;
      const positions = await db.position.findMany({ where: { status: 'OPEN' }});
      await ctx.reply(formatPositions(positions));
    });
    
    this.bot.command('stop', async (ctx) => {
      if (!this.isAdmin(ctx.from?.id)) return;
      await triggerKillSwitch('manual_telegram', String(ctx.from.id));
      await ctx.reply('🛑 Kill switch triggered. Bot stopped.');
    });
    
    this.bot.command('reset_killswitch', async (ctx) => {
      if (!this.isAdmin(ctx.from?.id)) return;
      await resetKillSwitch(String(ctx.from.id));
      await ctx.reply('✅ Kill switch reset. Bot will resume on next cycle.');
    });
    
    this.bot.start();
  }
  
  private isAdmin(userId?: number): boolean {
    if (!userId) return false;
    const admins = process.env.TELEGRAM_ADMIN_USER_IDS?.split(',') || [];
    return admins.includes(String(userId));
  }
}
```

### Command Reference

| Command | Auth | Action |
|---------|------|--------|
| `/status` | Admin | Bot status, mode, open positions count |
| `/positions` | Admin | List open positions dengan unrealized P&L |
| `/pnl` | Admin | P&L today/week/all-time |
| `/stop` | Admin | Trigger kill switch |
| `/reset_killswitch` | Admin | Reset kill switch |
| `/pause` | Admin | Pause new buys, tetap monitor positions |
| `/resume` | Admin | Resume normal operation |
| `/config` | Admin | Show current config (sanitized) |
| `/help` | All | List commands |

---

## Rate Limiting

Telegram API: 30 messages/sec ke chat berbeda, **1 message/sec ke chat yang sama**.

```typescript
import PQueue from 'p-queue';

const queue = new PQueue({ 
  interval: 1000, 
  intervalCap: 1 
});

async function send(message: string) {
  return queue.add(() => bot.api.sendMessage(chatId, message));
}
```

### Smart Batching

Untuk event yang sering (filter rejections), batch:

```typescript
class BatchedNotifier {
  private buffer: string[] = [];
  private timer?: NodeJS.Timeout;
  
  add(line: string) {
    this.buffer.push(line);
    if (this.buffer.length >= 10) {
      this.flush();
    } else if (!this.timer) {
      this.timer = setTimeout(() => this.flush(), 60_000);
    }
  }
  
  private flush() {
    if (this.buffer.length === 0) return;
    const msg = this.buffer.join('\n');
    notifier.send(msg, { silent: true });
    this.buffer = [];
    clearTimeout(this.timer);
    this.timer = undefined;
  }
}
```

---

## Alert Routing

Tidak semua event ke Telegram. Routing table:

```typescript
const routingRules = {
  'position.opened':    ['telegram'],
  'position.closed':    ['telegram'],
  'kill_switch':        ['telegram_critical', 'log'],
  'balance_drift':      ['telegram_critical', 'log'],
  'rpc_error':          ['log', 'discord_ops_channel'],
  'rpc_recovered':      ['log'],
  'filter.rejected':    ['log_only'],
  'filter.passed':      ['log_only'],
  'trade.failed':       ['telegram', 'log'],
  'daily.summary':      ['telegram'],
  'config.reloaded':    ['log', 'telegram'],
  'startup':            ['telegram', 'log'],
  'shutdown':           ['telegram', 'log'],
};
```

---

## Discord (Optional)

Setup webhook (lebih simple dari bot):

```bash
# Discord channel settings → Integrations → Webhooks → New webhook → copy URL
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...
```

### Discord Notifier

```typescript
export class DiscordNotifier {
  async send(message: string, options?: { color?: number; title?: string }) {
    await fetch(process.env.DISCORD_WEBHOOK_URL!, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        embeds: [{
          title: options?.title,
          description: message,
          color: options?.color ?? 0x5865F2,
          timestamp: new Date().toISOString(),
        }],
      }),
    });
  }
}
```

Color coding:
- 🟢 Win: `0x57F287` (green)
- 🔴 Loss: `0xED4245` (red)
- 🟡 Warning: `0xFEE75C` (yellow)
- ⚪ Info: `0x5865F2` (blurple)

---

## Email (Optional, for Daily Reports)

Untuk daily summary yang lebih panjang dengan chart, email lebih cocok dari chat.

Pakai service: Resend, Postmark, atau SES.

```typescript
import { Resend } from 'resend';
const resend = new Resend(process.env.RESEND_API_KEY);

await resend.emails.send({
  from: 'sniper@yourdomain.com',
  to: 'you@example.com',
  subject: 'Daily Report - 2026-01-15',
  html: renderDailyReport(stats),
});
```

---

## Notification Settings UI

Di dashboard, tambah halaman settings untuk control per-event:

```typescript
// Dashboard route: /settings/notifications
{
  events: {
    'position.opened': { telegram: true, discord: false },
    'position.closed.win': { telegram: true, discord: true },
    'position.closed.loss': { telegram: true, discord: true },
    'filter.rejected': { telegram: false, discord: false, log: true },
    'kill_switch': { telegram: true, discord: true, email: true },
  },
  quietHours: { 
    enabled: true, 
    from: '23:00', 
    to: '07:00',
    timezone: 'Asia/Jakarta',
    excludeCritical: true,  // critical tetap notify
  },
}
```

---

## Anti-Spam Safeguards

```typescript
class NotificationCircuitBreaker {
  private counts = new Map<string, number>();
  private resetTimer?: NodeJS.Timeout;
  
  constructor(private maxPerHour = 100) {
    this.resetTimer = setInterval(() => this.counts.clear(), 3600_000);
  }
  
  async send(type: string, message: string) {
    const count = (this.counts.get(type) ?? 0) + 1;
    this.counts.set(type, count);
    
    if (count > this.maxPerHour) {
      log.warn(`Notification rate limit hit for ${type}, dropping`);
      return;
    }
    
    if (count === this.maxPerHour) {
      // Send "going silent" notification once
      await this.notifier.send(
        `⚠️ Notification rate limit hit for "${type}". Going silent for this hour.`
      );
    }
    
    await this.notifier.send(message);
  }
}
```

Mencegah bot infinite-loop yang spam thousand notifs.

---

## Notification Audit Log

Setiap notification disimpan ke DB untuk audit:

```typescript
await db.systemEvent.create({
  data: {
    type: 'NOTIFICATION_SENT',
    severity: 'INFO',
    source: 'notify',
    message: messagePreview,
    context: { channel, eventType, recipient },
  }
});
```

Berguna saat: "kok saya tidak dapat notif posisi X?"

---

## Pitfalls

- ❌ **Spam semua event ke Telegram** — Anda akan mute, lalu miss critical alert
- ❌ **No rate limit** — bug bisa bikin 1000 message/menit, Telegram banned
- ❌ **Plaintext bot token di code** — bot bisa di-hijack
- ❌ **No admin check** — strangers bisa kontrol bot Anda
- ✅ **Test critical alerts** — minimum sekali jalan emergency stop test
- ✅ **Quiet hours** untuk non-critical alerts
- ✅ **Audit log** semua notification untuk troubleshoot