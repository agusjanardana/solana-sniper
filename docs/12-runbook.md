# 12 — Runbook

Operational playbook saat ada masalah. Penting: saat panik, Anda tidak ingin baca dokumentasi dari awal. Runbook = step-by-step instruction.

---

## Quick Commands

Cheat sheet untuk situasi umum:

```bash
# Status bot
sudo systemctl status sniper-worker
sudo journalctl -u sniper-worker -f
sudo journalctl -u sniper-worker -n 100

# Emergency stop
pnpm cli kill
# atau via Telegram: /stop
# atau via systemd:
sudo systemctl stop sniper-worker

# Reset kill switch
pnpm cli kill-switch reset --note "investigated and resolved"

# Restart
sudo systemctl restart sniper-worker

# Wallet balance check
pnpm cli wallet balance
solana balance $WALLET_ADDRESS --url $RPC_URL

# Drain wallet (emergency)
pnpm cli wallet drain --to $HOT_WALLET_ADDRESS

# Database status
psql $DATABASE_URL -c "SELECT COUNT(*) FROM \"Trade\" WHERE \"createdAt\" > NOW() - INTERVAL '1 hour';"

# RPC health check
curl -s -X POST $SOLANA_RPC_URL \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getHealth"}'
```

---

## Incident Playbooks

### 🚨 Incident: Bot Stopped Trading Unexpectedly

**Symptoms**: Tidak ada notifikasi entry selama > 1 jam di market jam sibuk.

**Diagnosis**:

```bash
# 1. Cek service status
sudo systemctl status sniper-worker

# 2. Kalau running, cek heartbeat
redis-cli get bot:heartbeat
# Bandingkan dengan current time

# 3. Cek log
sudo journalctl -u sniper-worker -n 200 --no-pager

# 4. Cek kill switch
psql $DATABASE_URL -c "SELECT * FROM \"KillSwitchEvent\" WHERE triggered = true AND \"resolvedAt\" IS NULL;"

# 5. Cek WebSocket status (dari log)
grep "WS_DISCONNECT\|WS_RECONNECT" /var/log/sniper/worker.log | tail -20

# 6. Cek RPC health
curl -s -X POST $SOLANA_RPC_URL \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getHealth"}'
```

**Resolution paths**:

| Cause | Action |
|-------|--------|
| Kill switch triggered | Investigate cause → `pnpm cli kill-switch reset` |
| Service crashed | `sudo systemctl restart sniper-worker` → review logs |
| WS disconnected | Should auto-reconnect; kalau tidak, restart |
| RPC down | Switch ke fallback RPC, atau wait |
| Memory leak (OOM) | Restart, monitor memory growth, file bug |
| Disk full | Cleanup old logs/backups |

---

### 🚨 Incident: Balance Drift Detected

**Symptoms**: Telegram alert "BALANCE DRIFT" atau dashboard banner red.

**Immediate Actions** (dalam 5 menit pertama):

```bash
# 1. STOP everything
pnpm cli kill --reason "balance_drift_investigation"

# 2. Snapshot current state
solana balance $WALLET_ADDRESS --url $RPC_URL
psql $DATABASE_URL -c "SELECT * FROM \"WalletBalance\" ORDER BY \"recordedAt\" DESC LIMIT 5;"

# 3. List recent transactions
solana transaction-history $WALLET_ADDRESS --url $RPC_URL --limit 20
# atau via Solscan: https://solscan.io/account/$WALLET_ADDRESS
```

**Investigation**:

```sql
-- Trades yang tracked di DB
SELECT "createdAt", side, "amountInSol", "txSignature", status
FROM "Trade"
WHERE "createdAt" > NOW() - INTERVAL '24 hours'
ORDER BY "createdAt" DESC;

-- Compare dengan on-chain history dari Solscan
-- Cari tx yang tidak ada di DB → suspicious
```

**Decision tree**:

```
On-chain tx exists tapi tidak di DB?
├─ YES → Possible unauthorized access
│        → Drain wallet ke safe address NOW
│        → Rotate semua keys (RPC, DB, wallet)
│        → Audit access logs
│
└─ NO → DB out of sync atau bug accounting
         ├─ Reconcile DB dengan on-chain history
         ├─ File bug report
         └─ Reset kill switch setelah fix
```

**Drain Wallet Procedure**:

```bash
# Sangat hati-hati, double check destination address
pnpm cli wallet drain \
  --to "YOUR_SAFE_HOT_WALLET_ADDRESS" \
  --keep-sol 0.01 \
  --include-tokens
# Akan minta konfirmasi 2x
```

---

### 🚨 Incident: Stuck Position (Can't Sell)

**Symptoms**: Posisi sudah hit SL tapi tidak exit, atau exit tx kept failing.

**Diagnosis**:

```bash
# 1. Cek position di DB
psql $DATABASE_URL -c "SELECT * FROM \"Position\" WHERE id = '$POSITION_ID';"

# 2. Cek attempt logs
psql $DATABASE_URL -c "
SELECT * FROM \"Trade\" 
WHERE \"positionId\" = '$POSITION_ID' AND side = 'SELL'
ORDER BY \"createdAt\" DESC;
"

# 3. Common reasons:
# - Slippage too tight (price moved too fast)
# - Pool drained (rugged)
# - Honeypot (can't sell, transfer tax 100%)
# - RPC timeout
```

**Resolution**:

```bash
# Option 1: Retry with higher slippage
pnpm cli position exit \
  --id $POSITION_ID \
  --slippage-bps 5000 \
  --priority-fee high

# Option 2: Manual swap via Jupiter UI
# Open https://jup.ag/swap → connect burner wallet → swap

# Option 3: Cek if honeypot (cek transfer tax di Rugcheck)
# Kalau YES → mark as loss, write off

# Option 4: Mark position as closed manually (last resort)
pnpm cli position close-manual \
  --id $POSITION_ID \
  --exit-reason "honeypot_unsellable" \
  --realized-pnl-sol -0.05
```

---

### 🚨 Incident: RPC Provider Down

**Symptoms**: Banyak `RPC_ERROR` di log, fill rate drop drastis.

**Diagnosis**:

```bash
# Cek health
curl -s $PRIMARY_RPC -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"getHealth"}'

# Cek status page provider
# Helius: https://status.helius.dev/
# Quicknode: https://status.quicknode.com/

# Cek di Solana status
# https://status.solana.com/
```

**Resolution**:

```bash
# Option 1: Force switch ke fallback
pnpm cli rpc switch --to fallback

# Option 2: Pause new buys, tetap monitor positions
pnpm cli pause-buys

# Option 3: Full stop kalau both RPC down
pnpm cli kill --reason "rpc_outage"
```

**Post-incident**: Tambah fallback RPC ke-3 kalau sering kena.

---

### 🚨 Incident: Database Corruption / Down

**Symptoms**: Worker crash dengan "connection refused" atau "relation does not exist".

**Diagnosis**:

```bash
# Cek service
sudo systemctl status postgresql

# Cek disk
df -h
# Kalau / penuh, ini biasanya cause-nya

# Cek log
sudo tail -100 /var/log/postgresql/postgresql-16-main.log
```

**Resolution**:

```bash
# Disk penuh?
sudo apt-get clean
sudo journalctl --vacuum-size=500M
# Cleanup old backups
find ~/backups -mtime +7 -delete

# Service crashed?
sudo systemctl restart postgresql

# Data corrupted?
# 1. Stop bot
sudo systemctl stop sniper-worker

# 2. Try repair
sudo -u postgres pg_resetwal /var/lib/postgresql/16/main

# 3. Kalau tidak bisa, restore dari backup
pg_restore -h localhost -U sniper -d sniper /home/sniper/backups/sniper-LATEST.dump

# 4. Restart bot
sudo systemctl start sniper-worker
```

---

### 🚨 Incident: Excessive Losses (Strategy Broken?)

**Symptoms**: Banyak loss berurutan, daily loss limit hit, kill switch nyala.

**DON'T**:
- ❌ Reset kill switch dan lanjut trading "berharap recovery"
- ❌ Naikkan position size untuk "balas dendam"
- ❌ Loosen filter untuk "lebih banyak peluang"

**DO**:

```bash
# 1. Tetap stop, analisis dulu
pnpm cli kill --reason "investigating_losses"

# 2. Run analysis
pnpm cli analyze --period today --type losses
```

**Investigation checklist**:

- [ ] Apakah ada perubahan market kondisi? (e.g., extreme volatility hari ini)
- [ ] Apakah ada perubahan config recent? (cek git log)
- [ ] Apakah filter terlalu loose? (cek scam_score rata-rata trade yang loss)
- [ ] Apakah TP/SL salah setting? (cek hold time distribution)
- [ ] Apakah ada bug execution? (cek slippage actual vs target)
- [ ] Apakah RPC lag bikin late entry? (cek latency P95)

**Actions berdasarkan finding**:

| Finding | Action |
|---------|--------|
| Market kondisi extreme | Stop trading hari ini, resume besok |
| Filter loose | Switch ke `strict` profile, run paper trade 1 hari |
| TP terlalu tinggi | Backtest dengan TP lebih rendah, evaluate |
| Execution lag | Check RPC, mungkin switch provider |
| Bot bug | File issue, run paper mode sambil fix |

---

### 🚨 Incident: Suspected Compromise

**Symptoms**: 
- Login attempts unknown di dashboard
- Wallet transactions yang tidak dilakukan bot
- File modified yang Anda tidak edit
- Strange process di VPS

**Immediate Actions (in order)**:

```bash
# 1. Stop everything
sudo systemctl stop sniper-worker sniper-dashboard cloudflared

# 2. Disconnect VPS from network if possible
sudo ufw default deny outgoing
sudo ufw reload
# (kecuali Anda butuh SSH untuk recovery)

# 3. Drain burner wallet IMMEDIATELY
# Pakai computer Anda lokal, BUKAN VPS
# Pakai wallet baru dengan key baru
solana transfer $SAFE_ADDRESS ALL --keypair $BURNER_KEYPAIR \
  --url https://api.mainnet-beta.solana.com

# 4. Rotate all secrets
# - New wallet (new seed phrase)
# - New RPC keys (Helius dashboard)
# - New DB password
# - New SSH keys
# - New Telegram bot token

# 5. Audit
last -20  # login history
who       # current logged in
ps aux    # running processes
sudo netstat -tlnp  # listening ports
sudo cat /var/log/auth.log | tail -200  # auth log

# 6. Forensics (kalau penting)
# - Snapshot VPS image
# - Copy logs ke safe location
# - Document timeline

# 7. Rebuild
# Anggap VPS compromised. Setup fresh VPS, restore DB dari clean backup.
```

**Post-incident**:
- Write post-mortem: `docs/incidents/YYYY-MM-DD-compromise.md`
- Review apa yang leaked / accessed
- Hardening review

---

## Routine Maintenance

### Daily (Manual Review, 5 minutes)

```bash
# Quick health check
ssh sniper@vps "sudo systemctl status sniper-worker sniper-dashboard | grep Active"

# Today's P&L
pnpm cli stats today

# Open positions sanity
pnpm cli positions list
```

Atau via dashboard `/`.

### Weekly (15 minutes)

- [ ] Review trade list, cari pattern (best & worst)
- [ ] Cek filter reject reasons distribution
- [ ] Backtest week dengan current config, compare to live
- [ ] Withdraw profit kalau bankroll naik > 50% dari baseline
- [ ] Update dependency yang ada security patch: `pnpm audit fix`

### Monthly (30 minutes)

- [ ] Full backup ke offline storage
- [ ] Rotate RPC keys
- [ ] Review server logs untuk anomaly
- [ ] Check VPS resource trend (CPU/RAM/disk growth)
- [ ] Test backup restore procedure (di staging)
- [ ] Update OS: `sudo apt update && sudo apt upgrade`
- [ ] Review & update docs based on operational learnings

### Quarterly (1-2 hours)

- [ ] Full strategy review: backtest, paper, live alignment
- [ ] Major dependency updates (Node version, etc)
- [ ] Disaster recovery drill: restore from backup to fresh VPS
- [ ] Tax/accounting reconciliation

---

## Post-Mortem Template

Setiap incident, tulis post-mortem ke `docs/incidents/YYYY-MM-DD-summary.md`:

```markdown
# Incident: [Brief title]

**Date**: 2026-01-15
**Duration**: 14:23 - 14:58 UTC (35 min)
**Severity**: High / Critical
**Author**: [you]

## Summary
[1-2 paragraf: apa yang terjadi]

## Timeline
- 14:23 — First alert: balance drift detected
- 14:24 — Bot auto-stopped
- 14:25 — I noticed Telegram alert
- 14:28 — Investigation started
- ...
- 14:58 — Bot resumed

## Root Cause
[Apa penyebab sebenarnya]

## Impact
- Loss: 0.05 SOL
- Downtime: 35 min
- Trades missed: ~3-5

## What Went Well
- Auto-detection worked
- Kill switch fired
- Alerting timely

## What Went Wrong
- [Apa yang lambat / salah]

## Action Items
- [ ] [Concrete fix 1]
- [ ] [Concrete fix 2]
- [ ] [Process improvement]

## Lessons Learned
[Insight untuk next time]
```

---

## Emergency Contacts

Tulis di tempat aman (NOT in repo):

```
RPC Provider Support:
- Helius: https://discord.gg/helius
- QuickNode: support@quicknode.com

VPS Support:
- Hetzner: https://www.hetzner.com/support

Critical recovery wallet (cold):
- Address: [your cold wallet]
- Seed phrase location: [physical location]

Backup access:
- Cloud storage: [provider]
- 2FA backup codes: [physical location]
```

---

## Pitfalls

- ❌ **Panic and bypass procedures** — runbook ada untuk dipakai, bukan dilewati saat panik
- ❌ **Reset kill switch tanpa investigasi** — itu yang bikin loss makin besar
- ❌ **Skip post-mortem** — kalau tidak document, akan repeat mistakes
- ❌ **Trust on-chain data 100% kalau RPC suspect** — cross-check minimum 2 sources
- ✅ **Practice procedures sebelum butuh** — drill emergency stop, drain wallet, dll
- ✅ **Keep runbook updated** — setiap incident, refine playbook