# Architecture

## High Level Flow

Solana Websocket
↓
Token Scanner
↓
Scam Filter
↓
Risk Engine
↓
Trade Decision
↓
Execution Engine
↓
Position Manager
↓
TP/SL Engine
↓
Dashboard + Notifications

---

# Services

## Token Scanner
Detect new tokens and liquidity events.

## Scam Filter
Analyze token risk.

## Risk Engine
Validate risk limits.

## Execution Engine
Execute buy/sell transaction.

## Position Manager
Track active positions.

## TP/SL Engine
Manage exits automatically.

## Notification Service
Send Telegram/Discord alerts.

## Dashboard
Realtime monitoring UI.

---

# Communication

Use:
- Websocket
- Redis Pub/Sub
- BullMQ jobs

---

# Realtime Requirements

- Low latency
- Fast websocket updates
- Retry mechanism
- Transaction confirmation handling

---

# Failure Handling

Must handle:
- RPC timeout
- websocket disconnect
- transaction failure
- duplicate execution
- liquidity removal