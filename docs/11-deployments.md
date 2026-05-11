# 11 — Deployment

Cara deploy bot ke VPS untuk 24/7 operation. Untuk personal use, simple & cost-effective.

---

## Architecture Recommendation

```
┌─────────────────────────────────────────────────────┐
│   VPS (Hetzner / DigitalOcean)                      │
│                                                      │
│   ┌─────────────┐   ┌─────────────┐                 │
│   │ Worker      │   │ Dashboard   │                 │
│   │ (Node.js)   │   │ (Next.js)   │                 │
│   └─────────────┘   └─────────────┘                 │
│   ┌─────────────┐   ┌─────────────┐                 │
│   │ PostgreSQL  │   │ Redis       │                 │
│   └─────────────┘   └─────────────┘                 │
│                                                      │
└─────────────────────────────────────────────────────┘
       │
       ├─ Cloudflare (DNS + Tunnel)
       │
       └─ User (you)
```

Semua di 1 VPS untuk personal use. Kalau growth, bisa pisah DB ke managed service nanti.

---

## VPS Recommendation

### Provider Comparison

| Provider | Tier | Specs | Cost/mo | Recommendation |
|----------|------|-------|---------|----------------|
| **Hetzner** | CPX21 | 3 vCPU, 4GB, 80GB SSD | ~$8 | ⭐ Best value |
| **Hetzner** | CPX31 | 4 vCPU, 8GB, 160GB SSD | ~$15 | Comfortable headroom |
| **DigitalOcean** | Basic | 2 vCPU, 4GB, 80GB | ~$24 | Good US/SG options |
| **Vultr** | Cloud Compute | 2 vCPU, 4GB | ~$24 | Many regions |
| **AWS Lightsail** | $20 plan | 2 vCPU, 4GB | $20 | Stable but pricey |

**Recommendation untuk personal use**: Hetzner CPX21 atau CPX31.

### Location

Pilih VPS terdekat dengan RPC provider Anda. Helius punya endpoint di:
- Frankfurt (EU)
- N. Virginia (US-East)
- Singapore (APAC)

Match VPS location. **Latency RPC ↔ VPS lebih penting dari latency VPS ↔ Anda**.

### OS

Ubuntu 22.04 LTS atau 24.04 LTS. Avoid:
- Debian (more conservative versions)
- CentOS/RHEL (lebih cocok untuk enterprise)
- Alpine (kecil tapi musl libc kadang trouble dengan Node native modules)

---

## Initial Server Setup

### 1. Create User

```bash
# Login as root pertama kali
ssh root@your-vps-ip

# Create non-root user
adduser sniper
usermod -aG sudo sniper

# Setup SSH key untuk user baru
mkdir -p /home/sniper/.ssh
cp ~/.ssh/authorized_keys /home/sniper/.ssh/
chown -R sniper:sniper /home/sniper/.ssh
chmod 700 /home/sniper/.ssh
chmod 600 /home/sniper/.ssh/authorized_keys

# Test SSH dari user baru
exit
ssh sniper@your-vps-ip
```

### 2. Harden SSH

```bash
sudo nano /etc/ssh/sshd_config
```

Edit:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 22  # atau ubah ke port custom, e.g., 2222
```

```bash
sudo systemctl restart sshd
```

### 3. Firewall (UFW)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp        # SSH (atau port custom)
sudo ufw allow 80/tcp        # HTTP (Caddy/Nginx)
sudo ufw allow 443/tcp       # HTTPS
# DON'T allow 5432 (Postgres) or 6379 (Redis) — bind to localhost only
sudo ufw enable
```

### 4. Fail2ban

```bash
sudo apt install fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Default jail untuk SSH sudah cukup.

### 5. Automatic Updates

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

---

## Install Dependencies

### Node.js (via nvm)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20
nvm alias default 20
node --version  # v20.x.x
```

### pnpm

```bash
npm install -g pnpm
pnpm --version
```

### PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib
sudo systemctl enable postgresql

# Create DB & user
sudo -u postgres psql

CREATE USER sniper WITH PASSWORD 'STRONG_PASSWORD_HERE';
CREATE DATABASE sniper OWNER sniper;
GRANT ALL PRIVILEGES ON DATABASE sniper TO sniper;
\q

# Bind to localhost only (default sudah, but verify)
sudo nano /etc/postgresql/16/main/postgresql.conf
# listen_addresses = 'localhost'

sudo systemctl restart postgresql
```

### Redis

```bash
sudo apt install redis-server
sudo systemctl enable redis-server

# Bind to localhost only (default)
# Verify: bind 127.0.0.1 ::1
sudo nano /etc/redis/redis.conf

# Set password
# requirepass YOUR_REDIS_PASSWORD

sudo systemctl restart redis-server
```

### Caddy (Reverse Proxy + Auto HTTPS)

Pilih Caddy over Nginx untuk personal use — auto-HTTPS dari Let's Encrypt out of the box.

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

---

## Deploy Application

### 1. Clone Repo

```bash
cd ~
git clone https://github.com/agusjanardana/solana-sniper.git
cd solana-sniper
```

### 2. Setup `.env`

```bash
cp .env.example .env
nano .env  # isi production values
chmod 600 .env
```

### 3. Setup `config.yaml`

```bash
cp config.example.yaml config.yaml
nano config.yaml
```

### 4. Install & Build

```bash
pnpm install --frozen-lockfile
pnpm run db:migrate:deploy
pnpm run build
```

### 5. Test Run

```bash
# Pastikan jalan di paper mode dulu!
NODE_ENV=production MODE=paper pnpm run start
# Ctrl+C setelah verify tidak crash
```

---

## Process Management: systemd

Bikin service untuk worker & dashboard.

### Worker Service

```bash
sudo nano /etc/systemd/system/sniper-worker.service
```

```ini
[Unit]
Description=Solana Sniper Worker
After=network.target postgresql.service redis-server.service
Requires=postgresql.service redis-server.service

[Service]
Type=simple
User=sniper
WorkingDirectory=/home/sniper/solana-sniper
Environment=NODE_ENV=production
EnvironmentFile=/home/sniper/solana-sniper/.env
ExecStart=/home/sniper/.nvm/versions/node/v20.x.x/bin/node apps/worker/dist/index.js
Restart=always
RestartSec=10

# Logging
StandardOutput=append:/var/log/sniper/worker.log
StandardError=append:/var/log/sniper/worker-error.log

# Resource limits
MemoryMax=2G
CPUQuota=200%

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/sniper/solana-sniper/logs /var/log/sniper

[Install]
WantedBy=multi-user.target
```

```bash
# Create log dir
sudo mkdir -p /var/log/sniper
sudo chown sniper:sniper /var/log/sniper

# Enable + start
sudo systemctl daemon-reload
sudo systemctl enable sniper-worker
sudo systemctl start sniper-worker

# Check status
sudo systemctl status sniper-worker
sudo journalctl -u sniper-worker -f
```

### Dashboard Service

```ini
# /etc/systemd/system/sniper-dashboard.service
[Unit]
Description=Solana Sniper Dashboard
After=network.target postgresql.service

[Service]
Type=simple
User=sniper
WorkingDirectory=/home/sniper/solana-sniper/apps/dashboard
Environment=NODE_ENV=production
Environment=PORT=3000
EnvironmentFile=/home/sniper/solana-sniper/.env
ExecStart=/home/sniper/.nvm/versions/node/v20.x.x/bin/pnpm start
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Watchdog Service

```ini
# /etc/systemd/system/sniper-watchdog.service
[Unit]
Description=Solana Sniper Watchdog
After=sniper-worker.service

[Service]
Type=simple
User=sniper
WorkingDirectory=/home/sniper/solana-sniper
ExecStart=/home/sniper/.nvm/versions/node/v20.x.x/bin/node apps/watchdog/dist/index.js
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
```

---

## Caddy Configuration

```bash
sudo nano /etc/caddy/Caddyfile
```

```caddyfile
# Replace with your domain
dashboard.yourdomain.com {
    reverse_proxy localhost:3000
    
    # Optional: basic auth as extra layer
    # basicauth {
    #     admin $2a$10$BCRYPT_HASH_HERE
    # }
    
    # Security headers
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "no-referrer"
    }
    
    # Log
    log {
        output file /var/log/caddy/dashboard.log
    }
}

# Webhook endpoint (Helius push)
webhook.yourdomain.com {
    reverse_proxy localhost:4000
}
```

```bash
sudo systemctl reload caddy
```

Caddy otomatis dapat cert Let's Encrypt. Verify: `https://dashboard.yourdomain.com` work dengan HTTPS.

---

## Alternative: Cloudflare Tunnel (No Public IP)

Lebih secure dari expose port langsung. VPS tidak butuh inbound port 80/443.

```bash
# Install cloudflared
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb

# Login
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create sniper

# Config
nano ~/.cloudflared/config.yml
```

```yaml
tunnel: <TUNNEL_ID>
credentials-file: /home/sniper/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: dashboard.yourdomain.com
    service: http://localhost:3000
  - hostname: webhook.yourdomain.com
    service: http://localhost:4000
  - service: http_status:404
```

```bash
# Run as service
sudo cloudflared service install
sudo systemctl start cloudflared
```

Plus: bisa pakai Cloudflare Access untuk auth (Zero Trust, gratis tier).

---

## Backups (Automated)

### Database Backup Script

```bash
# /home/sniper/scripts/backup-db.sh
#!/bin/bash
set -euo pipefail

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR=/home/sniper/backups
RETENTION_DAYS=30

mkdir -p $BACKUP_DIR

# Dump
PGPASSWORD="$DB_PASSWORD" pg_dump -h localhost -U sniper sniper \
  --format=custom --compress=9 \
  --file=$BACKUP_DIR/sniper-$DATE.dump

# Encrypt (optional but recommended)
if [ -n "${BACKUP_GPG_RECIPIENT:-}" ]; then
  gpg --encrypt --recipient $BACKUP_GPG_RECIPIENT \
    $BACKUP_DIR/sniper-$DATE.dump
  rm $BACKUP_DIR/sniper-$DATE.dump
fi

# Upload to remote (rclone)
if command -v rclone &> /dev/null; then
  rclone copy $BACKUP_DIR/sniper-$DATE.dump.gpg remote:sniper-backups/
fi

# Cleanup old local
find $BACKUP_DIR -name "sniper-*.dump*" -mtime +$RETENTION_DAYS -delete

echo "Backup complete: sniper-$DATE.dump"
```

```bash
chmod +x /home/sniper/scripts/backup-db.sh

# Crontab
crontab -e
# Daily at 03:00
0 3 * * * /home/sniper/scripts/backup-db.sh >> /var/log/sniper/backup.log 2>&1
```

### Config Backup

`.env`, `config.yaml`, dan `~/.config/solana/*.json` (wallet keys) — backup ke encrypted storage **manual**, jangan automate (avoid leakage).

---

## Monitoring

### Resource Usage

Install `htop`, `iotop`:
```bash
sudo apt install htop iotop nethogs
```

### Disk Space Alert

```bash
# /home/sniper/scripts/check-disk.sh
#!/bin/bash
USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
if [ $USAGE -gt 85 ]; then
  curl -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
    -d chat_id=$TELEGRAM_CHAT_ID \
    -d text="⚠️ Disk usage at ${USAGE}% on sniper VPS"
fi
```

Cron tiap jam.

### Log Rotation

```bash
sudo nano /etc/logrotate.d/sniper
```

```
/var/log/sniper/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 sniper sniper
    sharedscripts
    postrotate
        systemctl reload sniper-worker > /dev/null 2>&1 || true
    endscript
}
```

### Optional: Grafana + Prometheus

Overkill untuk personal use, tapi kalau mau:

```bash
# Run via Docker
docker run -d --name prometheus -p 9090:9090 prom/prometheus
docker run -d --name grafana -p 3001:3000 grafana/grafana
```

Worker expose metrics di `/metrics`, Prometheus scrape, Grafana dashboard.

---

## Update Deployment

### Workflow

```bash
# Di lokal: develop, test, commit, push
git push origin main

# Di VPS
ssh sniper@vps
cd ~/solana-sniper

# Pull
git pull origin main

# Install deps (kalau ada perubahan)
pnpm install --frozen-lockfile

# Migrate DB (kalau ada migration baru)
pnpm run db:migrate:deploy

# Build
pnpm run build

# Restart services (zero-downtime kalau perlu, restart simple untuk personal)
sudo systemctl restart sniper-worker
sudo systemctl restart sniper-dashboard

# Verify
sudo systemctl status sniper-worker
sudo journalctl -u sniper-worker -n 50
```

### Deploy Script

Automasi via script:

```bash
# scripts/deploy.sh
#!/bin/bash
set -euo pipefail

echo "🔄 Pulling latest..."
git pull origin main

echo "📦 Installing deps..."
pnpm install --frozen-lockfile

echo "🗃️ Migrating DB..."
pnpm run db:migrate:deploy

echo "🏗️ Building..."
pnpm run build

echo "🔄 Restarting services..."
sudo systemctl restart sniper-worker
sudo systemctl restart sniper-dashboard

echo "✅ Deployed!"
sudo systemctl status sniper-worker --no-pager
```

### Rolling Deployment (Advanced)

Kalau Anda butuh zero-downtime:
- Run 2 workers di port berbeda
- Restart satu-satu
- Pakai PM2 dengan cluster mode

Untuk personal use: 30 detik downtime saat deploy biasanya OK.

---

## Docker (Optional Alternative)

Lebih portable, harder to leak host. Tapi adds complexity.

```dockerfile
# Dockerfile.worker
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY apps/worker/package.json ./apps/worker/
COPY packages/*/package.json ./packages/
RUN npm install -g pnpm && pnpm install --frozen-lockfile
COPY . .
RUN pnpm run build --filter=worker

FROM node:20-alpine
WORKDIR /app
RUN addgroup -S sniper && adduser -S sniper -G sniper
USER sniper
COPY --from=builder --chown=sniper:sniper /app/apps/worker/dist ./dist
COPY --from=builder --chown=sniper:sniper /app/node_modules ./node_modules
CMD ["node", "dist/index.js"]
```

`docker-compose.yml`:

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: sniper
      POSTGRES_USER: sniper
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped
  
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redisdata:/data
    restart: unless-stopped
  
  worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    env_file: .env
    depends_on: [postgres, redis]
    restart: unless-stopped
  
  dashboard:
    build:
      context: .
      dockerfile: Dockerfile.dashboard
    env_file: .env
    ports: ["3000:3000"]
    depends_on: [postgres]
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:
```

```bash
docker-compose up -d
```

---

## Cost Estimate

| Item | Cost/month |
|------|------------|
| Hetzner CPX21 VPS | $8 |
| Domain (annual amortized) | $1 |
| Helius RPC (free tier OK initially) | $0 - $99 |
| Backup storage (B2/rclone) | $1 |
| **Total** | **~$10 - $109** |

---

## Pitfalls

- ❌ **Run as root** — bot compromise = system compromise
- ❌ **Open Postgres/Redis to internet** — even with password, lots of brute-force
- ❌ **No firewall** — anything goes
- ❌ **Auto-update package mid-trading** — package ganti tiba-tiba = bot crash
- ❌ **Single point of failure: no backup** — 1 disk fail = lose all history
- ✅ **Test full deploy procedure di staging** sebelum prod
- ✅ **Document recovery procedure** (kalau VPS crash, gimana restore?)
- ✅ **Keep build artifacts** untuk rollback cepat