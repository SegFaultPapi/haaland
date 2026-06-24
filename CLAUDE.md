# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**Haaland** is a semi-autonomous trading agent for soccer prediction markets on Polymarket. It monitors active markets, fetches team news, analyzes price momentum and order flow, sends the combined context to Claude Haiku, and either auto-executes a trade or asks for Telegram approval based on a composite score (0–100).

The codebase does not exist yet — only docs are present. All Python files listed below are to be created.

## Stack

- **Python 3.11+**, async (`asyncio`), no threads
- **Claude Haiku** via Anthropic SDK for analysis (`analyst.py`) and parlay parsing (`parlay.py`)
- **Polymarket Gamma API** — market discovery
- **Polymarket CLOB API + py-clob-client** — prices, order book, trade execution
- **GNews API** (free tier, 100 req/day) — team news
- **python-telegram-bot** (async mode) — all user interaction
- **SQLite** — trades, price snapshots, signals log (no WAL mode; single writer)
- **APScheduler or asyncio loop** — periodic market scanning

## Running

```bash
pip install -r requirements.txt
cp .env.example .env   # fill in keys
python main.py
```

## Module Responsibilities

| File | What it does |
|---|---|
| `main.py` | Main loop — scans markets every `SCAN_INTERVAL_MIN` (default 30min), orchestrates the full pipeline |
| `markets.py` | Gamma API: `get_active_soccer_markets()`, `search_market(query)` — filters by soccer/football tag |
| `news.py` | GNews: `get_team_news(team, hours=48)` — **in-memory cache with 1h TTL is mandatory** to stay under 100 req/day limit |
| `market_data.py` | CLOB API: `get_price()`, `get_orderbook()`, `detect_whales(threshold=500)`, `get_spread()`, `get_price_history()` — computes momentum vs SQLite snapshots |
| `analyst.py` | Sends structured context to Claude Haiku, returns `{signal, confidence, estimated_probability, llm_edge_score, reasoning}` as JSON — must handle invalid JSON by falling back to SKIP |
| `scorer.py` | Combines analyst output + market data into composite score 0–100: LLM edge (40pts) + momentum (20pts) + liquidity (20pts) + smart money (20pts) |
| `parlay.py` | Parses free-text `/analyze` queries via Claude Haiku → N legs → calls `analyst.py` per leg → multiplies probabilities for combined verdict |
| `executor.py` | `place_order(market_id, side, amount_usdc)` via `py-clob-client`; validates `MODE` before executing; logs to SQLite; notifies via `alerts.py` |
| `alerts.py` | `python-telegram-bot` async handlers for all commands; restricts trade execution buttons to `OWNER_CHAT_ID`; handles inline keyboard callbacks |
| `db.py` | SQLite schema + CRUD for `trades`, `price_snapshots`, `signals_log` tables |

## Scoring and Trade Logic

```
score ≥ 75  → auto-execute (only in MODE=auto)
score 50–74 → send to OWNER_CHAT_ID with approve/reject inline buttons (semi-auto)
score < 50  → silent SKIP, log only
spread > 10% → auto-SKIP regardless of score
```

## Environment Variables

```env
POLYMARKET_KEY=        # Polygon wallet private key
CLAUDE_API_KEY=        # Anthropic API key
TELEGRAM_TOKEN=
GNEWS_KEY=
MODE=semi-auto         # "auto" | "semi-auto"
TRADE_SIZE_USDC=5
SCORE_AUTO_THRESHOLD=75
SCORE_ALERT_THRESHOLD=50
SCAN_INTERVAL_MIN=30
WHALE_THRESHOLD=500
MAX_SPREAD_PCT=10
OWNER_CHAT_ID=         # Only this chat ID can execute trades
GROUP_CHAT_ID=         # Optional
```

## analyst.py Prompt Contract

The prompt must instruct Claude Haiku to return **only** a JSON object with no markdown or extra text:

```json
{
  "signal": "BUY_YES" | "BUY_NO" | "SKIP",
  "confidence": "HIGH" | "MEDIUM" | "LOW",
  "estimated_probability": 0.0–1.0,
  "llm_edge_score": 0–40,
  "reasoning": "2–3 sentence max"
}
```

## SQLite Schema

```sql
CREATE TABLE trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    market_id TEXT NOT NULL, market_name TEXT,
    signal TEXT NOT NULL,           -- BUY_YES, BUY_NO
    composite_score INTEGER, entry_price REAL, amount_usdc REAL,
    status TEXT DEFAULT 'OPEN',     -- OPEN, WON, LOST, VOID
    pnl_usdc REAL, executed_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    resolved_at DATETIME, tx_hash TEXT, reasoning TEXT
);

CREATE TABLE price_snapshots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    market_id TEXT NOT NULL, price_yes REAL, price_no REAL,
    volume_24h REAL, spread_pct REAL,
    captured_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE signals_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    market_id TEXT NOT NULL, market_name TEXT, signal TEXT,
    composite_score INTEGER, llm_edge_pts INTEGER,
    momentum_pts INTEGER, liquidity_pts INTEGER, whale_pts INTEGER,
    skip_reason TEXT,
    action_taken TEXT,  -- AUTO_EXECUTED, SENT_TO_USER, SKIPPED_BY_SCORE, SKIPPED_BY_USER, SKIPPED_HIGH_SPREAD
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## Key Constraints

- **No threads** — use `asyncio` throughout; `python-telegram-bot` async mode + `main.py` loop must share one event loop
- **SQLite single writer** — `main.py` and `alerts.py` must use a shared connection or async queue; no WAL mode
- **GNews rate limit** — 100 req/day; `news.py` cache is not optional; alert if >80 req consumed
- **Claude Haiku rate** — max ~20 calls/hour to stay within $4–6/month budget; enforce in `analyst.py`
- **Telegram group safety** — filter out `/command@other_bot` patterns; `/summary` and all trade buttons are private-only
- **Trade execution** — run `executor.py` in DRY RUN mode until paper trading is validated (≥50 signals logged, ≥55% edge accuracy)

## Telegram Commands

| Command | Who | What |
|---|---|---|
| `/start` | anyone | Status and intro |
| `/markets` | anyone | Active soccer markets |
| `/signal [team]` | anyone | Full BUY/SELL/SKIP analysis |
| `/score [team]` | anyone | Composite score with 4-component breakdown |
| `/whales` | anyone | Orders >$500 detected in last 24h |
| `/analyze [query]` | anyone | Free-text single or multi-leg parlay analysis |
| `/summary` | owner only | Daily P&L and executed trades |
