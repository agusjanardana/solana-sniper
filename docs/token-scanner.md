# Token Scanner

## Purpose

Detect newly launched meme coins.

---

# Detection Sources

- Raydium
- Pump.fun
- Jupiter

---

# Detection Events

- New LP created
- Liquidity added
- Volume spike
- Migration event

---

# Scanner Responsibilities

- Subscribe websocket
- Parse liquidity events
- Store token metadata
- Trigger scam filter

---

# Important Metrics

- Liquidity
- Holders
- Market cap
- Volume
- LP age

---

# Anti Spam

Reject:
- Duplicate token
- Extremely low liquidity
- Blacklisted deployer