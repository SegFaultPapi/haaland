# MVP Scope: Haaland — Sports Trading Agent for Polymarket

## Core Problem
Polymarket traders have no fast way to detect edge in soccer markets using real context (news, injuries, lineups, price momentum, and smart money) — they operate blind or manually. They also have no tool to analyze parlays and get honest probability breakdowns per leg.

## MVP Flow (5 steps)
1. Agent scans active soccer markets on Polymarket via Gamma API
2. Fetches news from the last 48h for both teams via GNews (free)
3. Pulls live market data: price, volume, spread, and large orders via CLOB API
4. Claude Haiku analyzes all context and computes a **composite score (0–100)** → outputs `{signal, confidence, edge, score, reasoning}` as JSON
5. If score ≥ 75 → auto-executes via CLOB API. If 50–74 → asks user via Telegram. If <50 → SKIP

## MVP Features

### Core engine
- Fetch active soccer markets from Polymarket Gamma API
- Team news search via GNews API (free, 100 req/day)
- Claude Haiku analysis → structured JSON signal: `BUY YES / BUY NO / SKIP`
- Configurable mode: **semi-auto** (you approve) or **auto** (executes if score ≥ 75)
- Trade execution via `py-clob-client` (from base repo)
- Trade history in SQLite

### Market analytics (no extra APIs needed)
- Live price, 1h/24h change, and momentum from CLOB API
- Volume, liquidity, and bid/ask spread — auto-SKIP if spread > 10%
- Price snapshots every 30min in SQLite → detects sustained trends
- Smart money detection: flags single orders > $500 in the order book

### Composite score (0–100)
Combines four signals into one number to decide action:

| Signal | Max points |
|---|---|
| LLM edge (prob vs market price) | 40 pts |
| Price momentum (confirms or contradicts signal) | 20 pts |
| Liquidity (tight spread = trustworthy price) | 20 pts |
| Smart money (whale buying = confirmation) | 20 pts |

### Parlay & free-query analysis
User writes natural language in Telegram (private or group):

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

️ Final verdict: SKIP
   Parlay is roughly fairly priced.
   Leg 2 is slightly overpriced by the market.
   Better to play Leg 1 alone if anything.
```

How it works under the hood:
- Claude parses the free-text query → extracts teams/players/events
- Gamma API search for each leg as a separate market
- Same `analyst.py` runs per leg (no new logic)
- Combined probability = product of all leg estimates
- Final verdict compares combined market price vs agent estimate

Works for 1 to N legs. No extra APIs needed.

### Telegram bot with commands
Works in private chat **and** group chats:

| Command | Description |
|---|---|
| `/start` | Bot introduction and status |
| `/markets` | Lists active soccer markets right now |
| `/signal [team]` | Full analysis + BUY/SELL/SKIP signal for that match |
| `/score [team]` | Composite score 0–100 with breakdown |
| `/whales` | Large orders detected in the last 24h |
| `/summary` | Daily P&L and executed trades |
| `/analyze [query]` | Free-text parlay or single event analysis |

> Trade execution is restricted to your private `chat_id`. Insights, signals, and `/analyze` are public in groups.

## NOT in MVP
- Web dashboard
- Monte Carlo / GBM (v2)
- Live in-match trading (high frequency)
- Full Smart Score system (Hashdive-style, requires blockchain indexer)
- Advanced stats (API-Football, xG, ELO)
- Multi-sport (NBA, UFC, etc.)
- Formal backtesting
- Full Kelly Criterion
- LLM + quant engine fusion
- Saving parlay history (v1.1)
- User-specific parlay tracking per wallet (v1.1)

## ️ Project Structure
```
haaland/
├── main.py          ← main loop (runs every X minutes)
├── markets.py       ← Gamma API → active soccer markets + free-text search
├── news.py          ← GNews → team news (last 48h)
├── market_data.py   ← CLOB API → price, volume, spread, smart money
├── analyst.py       ← Claude Haiku prompt → BUY/SELL/SKIP + reasoning
├── scorer.py        ← composite score 0–100
├── parlay.py        ← parlay parser + multi-leg analyzer (reuses analyst.py)
├── executor.py      ← py-clob-client (from base repo)
├── alerts.py        ← Telegram bot + all command handlers incl. /analyze
├── db.py            ← SQLite: trades + price snapshots
└── .env             ← POLYMARKET_KEY, CLAUDE_KEY, TELEGRAM_TOKEN, GNEWS_KEY
```

## Monthly Cost
| Component | Cost |
|---|---|
| Gamma API (Polymarket markets) | Free |
| CLOB API (price, volume, orders) | Free |
| GNews API (news) | Free |
| Telegram bot | Free |
| Claude Haiku (~600 analyses/month incl. /analyze queries) | ~$4–6 |
| Server (Railway free tier or local) | ~$0–5 |
| **Total** | **$4–11/month** |

## 3-Week Build Plan
| Week | Deliverable |
|---|---|
| 1 | Gamma API + GNews + CLOB market data running, output visible in console |
| 2 | Claude Haiku analysis, composite score, Telegram commands `/signal` `/score` `/whales` |
| 3 | `/analyze` parlay command, trade execution, SQLite history, 1-week paper trading |

## Success Criteria
**The MVP works if:** the agent generates at least 1 signal with real edge per match day, a user can ask `/analyze` in a Telegram group and get an honest probability breakdown per leg, and trade execution works on Polymarket without manual intervention.

---

## Final Test
- [x] **30-second test:** A bot that reads soccer news, checks if Polymarket's price is wrong, scores the opportunity 0–100, lets anyone in a Telegram group ask about any match or parlay, and can execute trades automatically.
- [x] **Focus test:** 6 critical feature groups, all powered by the same 2 APIs (Gamma + CLOB) plus GNews and Claude.
- [x] **Problem test:** Solves ONE problem — detecting and explaining real edge in soccer prediction markets, for both automated trading and on-demand analysis.

---

*Haaland — Sports Trading Agent for Polymarket · June 2026*