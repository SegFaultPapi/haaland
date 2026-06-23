# Haaland — Sports Trading Agent for Polymarket

A semi-autonomous trading agent for soccer markets on Polymarket. Detects real edge by combining news, live market data, and LLM analysis — then executes or proposes trades via Telegram.

## The Problem

Polymarket traders in soccer markets operate blind: there's no fast way to detect edge using real context (news, injuries, lineups, price momentum, smart money). There's also no tool to analyze parlays with an honest probability breakdown per leg.

## How It Works (5 steps)

1. Scans active soccer markets on Polymarket via **Gamma API**
2. Fetches news from the last 48h for both teams via **GNews**
3. Pulls live market data: price, volume, spread, and large orders via **CLOB API**
4. **Claude Haiku** analyzes all context and computes a **composite score (0–100)** → `{signal, confidence, edge, score, reasoning}`
5. If score ≥ 75 → auto-executes. If 50–74 → asks user via Telegram. If < 50 → SKIP

## MVP Features

### Core engine
- Active soccer markets from Polymarket Gamma API
- Team news via GNews (free, 100 req/day)
- Claude Haiku analysis → structured JSON signal: `BUY YES / BUY NO / SKIP`
- Configurable mode: **semi-auto** (you approve) or **auto** (executes if score ≥ 75)
- Trade execution via `py-clob-client`
- Trade history in SQLite

### Market analytics (no extra APIs needed)
- Live price, 1h/24h change, and momentum from CLOB API
- Volume, liquidity, and bid/ask spread — auto-SKIP if spread > 10%
- Price snapshots every 30min in SQLite → detects sustained trends
- Smart money detection: flags single orders > $500 in the order book

### Composite score (0–100)

| Signal | Max points |
|---|---|
| LLM edge (prob vs market price) | 40 pts |
| Price momentum (confirms or contradicts signal) | 20 pts |
| Liquidity (tight spread = trustworthy price) | 20 pts |
| Smart money (whale buying = confirmation) | 20 pts |

### Parlay & free-query analysis

User writes natural language in Telegram (private or group chat):

```
/analyze Will Spain win the World Cup AND Mbappé scores in the final?
```

Bot response:

```
Parlay Analysis

① Spain wins World Cup
   Market price:     42%
   Agent estimate:   51%
   Edge:            +9% → slight BUY

② Mbappé scores in the final
   Market price:     38%
   Agent estimate:   29%
   Edge:            -9% → SKIP

─────────────────────
 Combined probability
   Market implied:   42% × 38% = 15.9%
   Agent estimate:   51% × 29% = 14.8%

 Final verdict: SKIP
   Parlay is roughly fairly priced.
   Leg 2 is slightly overpriced by the market.
   Better to play Leg 1 alone if anything.
```

Claude parses the free-text query → extracts teams/players/events → searches each leg in Gamma API → runs the same `analyst.py` per leg → multiplies estimated probabilities → compares against the combined market price.

### Telegram bot commands

Works in private chat **and** group chats:

| Command | Description |
|---|---|
| `/start` | Bot introduction and current status |
| `/markets` | Lists active soccer markets right now |
| `/signal [team]` | Full analysis + BUY/SELL/SKIP signal for that match |
| `/score [team]` | Composite score 0–100 with breakdown |
| `/whales` | Large orders detected in the last 24h |
| `/summary` | Daily P&L and executed trades |
| `/analyze [query]` | Free-text parlay or single event analysis |

> Trade execution is restricted to your private `chat_id`. Insights, signals, and `/analyze` are public in groups.

## Project Structure

```
haaland/
├── main.py          ← main loop (runs every X minutes)
├── markets.py       ← Gamma API → active soccer markets + free-text search
├── news.py          ← GNews → team news (last 48h)
├── market_data.py   ← CLOB API → price, volume, spread, smart money
├── analyst.py       ← Claude Haiku prompt → BUY/SELL/SKIP + reasoning
├── scorer.py        ← composite score 0–100
├── parlay.py        ← parlay parser + multi-leg analyzer (reuses analyst.py)
├── executor.py      ← py-clob-client
├── alerts.py        ← Telegram bot + all command handlers incl. /analyze
├── db.py            ← SQLite: trades + price snapshots
└── .env             ← POLYMARKET_KEY, CLAUDE_KEY, TELEGRAM_TOKEN, GNEWS_KEY
```

## Environment Variables

```env
POLYMARKET_KEY=
CLAUDE_KEY=
TELEGRAM_TOKEN=
GNEWS_KEY=
```

## Estimated Monthly Cost

| Component | Cost |
|---|---|
| Gamma API (Polymarket markets) | Free |
| CLOB API (price, volume, orders) | Free |
| GNews API (news) | Free |
| Telegram bot | Free |
| Claude Haiku (~600 analyses/month incl. `/analyze`) | ~$4–6 |
| Server (Railway free tier or local) | ~$0–5 |
| **Total** | **$4–11/month** |

## 3-Week Build Plan

| Week | Deliverable |
|---|---|
| 1 | Gamma API + GNews + CLOB market data running, output visible in console |
| 2 | Claude Haiku analysis, composite score, Telegram commands `/signal` `/score` `/whales` |
| 3 | `/analyze` parlay command, trade execution, SQLite history, 1-week paper trading |

## Not in MVP

- Web dashboard
- Monte Carlo / GBM (v2)
- Live in-match trading (high frequency)
- Full Smart Score system (Hashdive-style, requires blockchain indexer)
- Advanced stats (API-Football, xG, ELO)
- Multi-sport (NBA, UFC, etc.)
- Formal backtesting
- Full Kelly Criterion
- LLM + quant engine fusion
- Parlay history (v1.1)

## Success Criteria

The MVP works if: the agent generates at least 1 signal with real edge per match day, any user can write `/analyze` in a Telegram group and get an honest probability breakdown per leg, and trade execution on Polymarket works without manual intervention.

---

*Haaland — Sports Trading Agent for Polymarket · June 2026*
