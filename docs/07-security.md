# 07 — Security

Untuk personal-use sniper bot, security mistakes = kehilangan dana langsung. Section ini wajib dibaca **sebelum** Anda touch live trading.

---

## Threat Model

Untuk personal use, ancaman utama:

1. **Self-rug** — leak private key (commit ke git, share di screenshot)
2. **Malicious dependencies** — npm package punya backdoor
3. **RPC provider compromise** — server intercept tx
4. **Honeypot tokens** — beli tapi tidak bisa jual (lihat [02-scam-filter](./02-scam-filter.md))
5. **Bug bot sendiri** — infinite loop trade, drain wallet
6. **Phishing** — fake DEX UI, fake "telegram support"

---

## Wallet Strategy

### Always Burner Wallet

**Burner wallet** = wallet baru yang HANYA untuk sniper bot. Tidak punya NFT mahal, tidak punya staking, tidak punya identity.

```bash
# Generate burner via Solana CLI
solana-keygen new --outfile ~/.config/solana/sniper-burner.json
```

### Tiered Wallet Setup

```
[Cold Wallet (Ledger)]
    │ withdraw weekly
    │ deposit weekly profit
    ▼
[Hot Wallet (Phantom, manual)]
    │ top-up burner ke jumlah trading
    ▼
[Burner Wallet (bot)]
    │ trade
    ▼
[Profits drained back to Hot weekly]
```

**Rule**: Burner wallet TIDAK PERNAH menyimpan > 2-3x ukuran position max.

Contoh:
- Max position size: 0.1 SOL
- Max concurrent: 3
- Burner balance target: 0.5-0.7 SOL (cukup untuk operasi + buffer)

Sisa profit pindah ke hot wallet, lalu cold wallet, **manual**.

### Why?

Kalau private key burner leak (developer mistake, malware, RPC exploit), kerugian max = isi burner. Tidak menyentuh main holdings.

---

## Private Key Storage

### ❌ NEVER

- Commit ke git (bahkan di branch lain)
- Simpan di Slack/Discord
- Screenshot
- Email ke diri sendiri
- Plaintext di Google Drive/Dropbox
- Sebut di prompt ChatGPT/Claude

### ✅ Acceptable for Personal Use

| Method | Pro | Con |
|--------|-----|-----|
| **`.env` file (gitignored)** | Simple | Plaintext on disk |
| **OS keychain** (macOS Keychain, Linux libsecret) | Encrypted at rest | Code lebih kompleks |
| **Hardware wallet** (Ledger via CLI) | Paling secure | Awkward untuk bot |
| **Encrypted file + passphrase** | Encrypted at rest | Butuh input passphrase |
| **HSM/KMS** (AWS KMS, GCP KMS) | Production-grade | Overkill personal |

**Recommended personal setup**:
1. `.env` di-gitignore (verify dengan `git check-ignore -v .env`)
2. File permission 600: `chmod 600 .env`
3. `.env` di home directory, BUKAN di project root (lebih sulit accidental copy)

```bash
# .env.example (commit ini, BUKAN .env)
SOLANA_RPC_URL=https://...
WALLET_PRIVATE_KEY_PATH=~/.config/solana/sniper-burner.json
# Atau base58:
# WALLET_PRIVATE_KEY_BASE58=...
TELEGRAM_BOT_TOKEN=...
DATABASE_URL=...
```

### .gitignore

```gitignore
# Secrets
.env
.env.*
!.env.example
*.key
*.pem
keypairs/
wallets/

# Solana
.anchor/
target/

# Node
node_modules/
.next/
dist/

# DB
*.sqlite
*.db
prisma/migrations/dev/

# Logs
logs/
*.log

# OS
.DS_Store
Thumbs.db

# IDE
.vscode/
.idea/
```

---

## Pre-Commit Hooks

Pakai `husky` + `lint-staged` + custom check:

```json
// package.json
"husky": {
  "hooks": {
    "pre-commit": "lint-staged && npm run check-secrets"
  }
},
"scripts": {
  "check-secrets": "node scripts/check-secrets.js"
}
```

```javascript
// scripts/check-secrets.js
const { execSync } = require('child_process');

const staged = execSync('git diff --cached --name-only').toString().split('\n');

const patterns = [
  /PRIVATE_KEY/i,
  /SECRET_KEY/i,
  /[1-9A-HJ-NP-Za-km-z]{87,88}/, // base58 ed25519 private key
  /api[_-]?key.*['"][a-zA-Z0-9]{20,}['"]/,
];

for (const file of staged) {
  if (!file) continue;
  const content = require('fs').readFileSync(file, 'utf-8');
  for (const pattern of patterns) {
    if (pattern.test(content)) {
      console.error(`❌ Possible secret in ${file}`);
      process.exit(1);
    }
  }
}
```

Plus pakai [`gitleaks`](https://github.com/gitleaks/gitleaks) sebagai backup.

---

## Dependency Security

NPM ecosystem rentan ke supply chain attack (sudah banyak kasus crypto-stealing packages).

### Practices

1. **Lock file commit**: `pnpm-lock.yaml` selalu di git
2. **Audit reguler**: `pnpm audit` weekly
3. **No auto-update**: pin exact version (`"@solana/web3.js": "1.95.4"`, bukan `"^1.95.4"`)
4. **Review baru install**: `npm-package-readme` extension, cek download count & maintainer
5. **Pakai pnpm strict mode**: tidak install ghost dependencies

### Suspicious Patterns

Audit dependencies untuk:
- Package yang baru dipublish < 6 bulan tapi punya akses ke `fs`, `child_process`, `http`
- Maintainer dengan profil baru / no GitHub history
- Package dengan typo dari popular package (`solanaa-web3.js`)
- Package dengan minified install script

### Isolated Runtime

Untuk extra safety, jalankan bot di:
- VPS terpisah (DigitalOcean, Hetzner)
- Container (Docker) dengan limited capability
- Firewall rules: outbound hanya ke RPC + DB + Telegram

```dockerfile
FROM node:20-alpine
WORKDIR /app

# Non-root user
RUN addgroup -S botuser && adduser -S botuser -G botuser
USER botuser

COPY --chown=botuser:botuser package.json pnpm-lock.yaml ./
RUN npm install -g pnpm && pnpm install --frozen-lockfile

COPY --chown=botuser:botuser . .

CMD ["node", "apps/worker/dist/index.js"]
```

---

## RPC Provider Trust

RPC provider bisa **theoretically**:
- Lihat tx Anda sebelum di-submit ke network
- Front-run tx Anda (jarang, reputasi-damaging untuk provider)
- Log private key Anda (kalau Anda kirim plaintext ke RPC)

### Mitigation

1. **Never send private key via RPC** — sign tx locally, kirim hanya signed tx
2. **Use reputable provider** — Helius, Quicknode, Triton sudah terbukti
3. **Verify TLS** — pastikan https, check cert
4. **Don't expose RPC key in client-side code** — kalau dashboard, proxy via backend

```typescript
// ❌ Wrong: private key never goes here
// connection.requestSomething({ privateKey })

// ✅ Right: sign locally, submit signed
const tx = await buildTx(...);
tx.sign([wallet.payer]);
const sig = await connection.sendTransaction(tx);
```

---

## Bot-Side Safeguards

### Sanity Checks Before Submit

Sebelum tx benar-benar di-submit, cek:

```typescript
function preSubmitSanity(tx: Transaction, params: TradeParams): boolean {
  // 1. Amount tidak melebihi config
  if (params.amountSol > config.maxPositionSizeSol * 1.1) {
    log.error('Amount exceeds config limit', { params });
    triggerKillSwitch('safety_amount_exceeded');
    return false;
  }
  
  // 2. Recipient address adalah token mint yang valid
  if (!isValidMint(params.mint)) {
    return false;
  }
  
  // 3. Total daily SOL spent tidak melebihi daily limit
  if (todaySolSpent() + params.amountSol > config.dailyMaxSolSpent) {
    return false;
  }
  
  // 4. Wallet balance cukup, dengan buffer untuk fee
  const balance = await getBalance();
  if (balance < params.amountSol + 0.01) {
    log.warn('Insufficient balance');
    return false;
  }
  
  return true;
}
```

### Watchdog Process

Process terpisah yang monitor main bot:

```typescript
// watchdog.ts
setInterval(async () => {
  // Cek apakah bot heartbeat masih hidup
  const lastHeartbeat = await redis.get('bot:heartbeat');
  const age = Date.now() - parseInt(lastHeartbeat);
  
  if (age > 60_000) {
    await sendTelegram('🚨 BOT HEARTBEAT LOST');
    await triggerKillSwitch();
  }
  
  // Cek balance drift
  const balance = await getBalance();
  const expected = await getExpectedBalance();
  if (Math.abs(balance - expected) > 0.05) {
    await sendTelegram(`🚨 BALANCE DRIFT: ${balance} vs ${expected}`);
    await triggerKillSwitch();
  }
}, 30_000);
```

### Manual Kill Switch

CLI command yang bisa di-jalankan dari mana saja:

```bash
# Lokal
pnpm cli kill

# Remote (kalau VPS): SSH + jalanin
ssh vps "cd ~/sniper && pnpm cli kill"

# Lebih ekstrim: matikan service
ssh vps "sudo systemctl stop sniper-bot"
```

Telegram bot dengan command `/emergency_stop` juga bagus.

---

## Operational Security (OpSec)

### Don't Talk About It

- Jangan share strategi spesifik di grup Telegram/Discord publik
- Jangan share screenshot trade dengan address visible
- Jangan tweet "lol my bot just made +500%" — bikin target

Personal sniper success = quiet success.

### Geographic & Legal

- Cek regulasi di yurisdiksi Anda — di sebagian negara, automated trading crypto perlu compliance
- Tax reporting: jaga record semua trade untuk pajak
- KYC: kalau profit besar dan masuk ke exchange, siap dengan source-of-funds explanation

### Backup & Recovery

- **Seed phrase burner wallet**: tulis tangan, simpan di tempat aman (BUKAN di komputer)
- **Database backup**: pg_dump harian ke encrypted cloud
- **Config backup**: commit (tanpa secrets) ke git private repo

### Incident Playbook

Kalau curiga compromised:

1. **Immediate**: matikan bot (`pnpm cli kill`)
2. **Drain burner wallet**: transfer semua SOL & token ke hot wallet baru
3. **Revoke**: revoke semua RPC keys di provider dashboard
4. **Audit**: cari source compromise — leaked .env? compromised dep? compromised VPS?
5. **Rebuild**: wallet baru, RPC key baru, VPS baru kalau perlu
6. **Document**: post-mortem di docs/incidents/

---

## Pre-Live Checklist

Sebelum first live trade, **semua** harus ✅:

- [ ] Wallet baru (burner) dibuat khusus untuk bot
- [ ] Seed phrase burner ditulis tangan, disimpan offline
- [ ] `.env` di-gitignore & verified via `git check-ignore`
- [ ] Pre-commit hook untuk secret detection aktif
- [ ] Bot jalan di environment terisolasi (VPS atau container)
- [ ] Firewall rules outbound limited
- [ ] RPC keys hanya di server, tidak di client/repo
- [ ] Daily loss limit & kill switch tested
- [ ] Manual kill switch tested (CLI atau Telegram)
- [ ] Watchdog process aktif
- [ ] Telegram alerts work
- [ ] Database backup automated
- [ ] Paper trade ≥ 1 minggu sukses
- [ ] Maximum capital di burner = 2-3x position size
- [ ] Tahu cara emergency drain wallet

---

## Pitfalls

- ❌ **"Saya akan secure-kan nanti"** — security debt = wallet drain
- ❌ **Sharing config.yaml** dengan param "rahasia" Anda — kalau ketahuan strategi, di-counter
- ❌ **Run as root** — kalau bot compromised, full system compromised
- ❌ **Skip backup** — DB corrupt = lose trade history
- ✅ **Treat the bot like a small bank** — security mindset
- ✅ **Document everything** — kalau ada bug, audit trail penting