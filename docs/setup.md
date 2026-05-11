# Setup Guide

## Requirements

- Node.js 22+
- PostgreSQL
- Redis
- Solana RPC Provider

---

# Install

npm install

---

# ENV

TRADING_MODE=paper

RPC_URL=
PRIVATE_KEY=

DATABASE_URL=
REDIS_URL=

---

# Run Dashboard

cd apps/dashboard

npm run dev

---

# Run Bot

cd apps/bot-engine

npm run dev

---

# Recommended RPC

- Helius
- QuickNode
- Triton

Avoid public RPC for production.

---

# Wallet Setup

Create burner wallet ONLY.

Never use main wallet.