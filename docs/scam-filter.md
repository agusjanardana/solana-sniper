# Scam Filter Engine

## Goal

Reduce rug pull probability.

---

# Rules

## Liquidity

Reject if:
- liquidity < $5,000

---

## Holder Count

Reject if:
- holders < 50

---

## Top Holder

Reject if:
- top holder > 25%

---

## Mint Authority

Reject if enabled.

---

## Freeze Authority

Reject if enabled.

---

## Volume Analysis

Detect:
- fake volume
- suspicious swaps
- bot wash trading

---

# Risk Score

0–100

Example:
- 0–20 = safe
- 20–50 = medium risk
- 50+ = high risk

---

# External APIs

- Rugcheck
- Birdeye
- Dexscreener