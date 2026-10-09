# AutoTrader

## Purpose
A real-time financial data pipeline: scrapes multiple sources (earnings call transcripts, Truth Social, Reddit, StockTwits), streams the raw text through Kafka, and runs an LLM (Azure AI Foundry) signal-extraction agent over it to produce structured trading signals — `{symbol, sentiment, confidence, topic, key_quote, reasoning}`.

There is **no autonomous order execution**. A Robinhood MCP connection exists (read + trade-capable) but is explicitly execution-untested, and gated by a standing safety rule:

> No order is ever placed without an explicit, real-time request from Ved.

Watchlist: AAPL, MSFT, AMZN, PLTR, META, NVDA, TSLA, INTC, NFLX, JPM, GS, C, LMT, MA, SHEL.

## Status
Actively in progress, mid-buildout (as of 2026-08-21).
- **Everything is currently stopped**, including Docker itself (`docker ps` can't reach the daemon) — nothing auto-restarts after a host reboot, so this is the normal between-sessions state, not a problem. Last time it was up: Reddit + StockTwits signal consumers were running in `DRY_RUN=1` (pipeline wired end-to-end, not yet spending on live agent calls).
- EarningsHub live monitor and the EarningsHub/TruthSocial signal consumers: also stopped (the latter two have never been started at all, dry-run or otherwise).
- Robinhood MCP connected and explored, zero orders ever placed. Since then, stress-tested for rate limits: ~100 calls in ~4 min, zero errors or throttling — not a practical concern at this project's actual usage volume.
- **New this session**: `TradingDecisionSource/` scaffolded — a *separate*, direct Robinhood login (via `robin-stocks`, its own credentials) rather than the MCP connection, because MCP only works inside a live Claude session and this needs to run headless eventually. Deliberately **not** wired to the Foundry signal agent or order placement yet — explicit decision to backtest the trading hypothesis (does signal sentiment/confidence actually correlate with realized price moves?) before any live capital or automated decision-making touches it. `ROBINHOOD_CONFIG` credentials are still blank, connection not yet verified end-to-end.
- Repo recently cleaned up: removed dead `TwitterSource/`, stale scripts; scraped CSVs moved out of the tree into gitignored `data/`.

Next steps already identified in the project's own `HANDOFF.md`: fill in `ROBINHOOD_CONFIG` + verify the standalone connection, design/run the backtest (the gate on connecting anything to `TradingDecisionSource/`), bring Docker back up and decide what to restart, decide on flipping dry-run off, restart the EarningsHub monitor, extend `dataScraper.py` with yfinance fundamentals/news, consider Robinhood historicals replacing yfinance.

## Stack
Python ≥3.14, managed via `uv` (`pyproject.toml` / `uv.lock` are source of truth — `requirements.txt` is stale/legacy).

Key dependencies:
- `av` + `faster-whisper` — live audio capture/transcription (earnings calls)
- `azure-ai-projects` / `azure-identity` — Azure AI Foundry signal-extraction agent (`DefaultAzureCredential` / `az login`)
- `praw` — Reddit OAuth
- `truthbrush` — Truth Social scraping
- `pycognito` — Earnings Hub auth
- `beautifulsoup4` — scraping
- `robin-stocks` — Robinhood

## Architecture
- `dataScraper.py` — shared ticker watchlist + scraping utilities
- `EarningsHubSource/` — reverse-engineered SavvyTrader product; captures live earnings-call audio via HLS/PyAV, transcribes with faster-whisper
- `TruthSocialSource/`, `RedditSource/`, `StockTwitsSource/` — one scraper + one Kafka signal-consumer per source, all sharing the root-level `signal_agent.py` (`SignalAgent` wrapper) rather than duplicating agent logic
- `TradingDecisionSource/` — direct Robinhood login (`robin-stocks`, separate from the MCP connection) + probe script. Connection layer only — not yet wired to the signal agent or order placement, pending a backtest of the trading hypothesis
- `KafkaSetup/` — topic manager, producer/consumer helpers; Kafka runs in local Docker (container `AutoTrader`, **no restart policy — must be started manually each session**)
- `start_scrapers.py` / `stop_scrapers.py` — manage the 3 raw scrapers as background processes
- `.claude/skills/` — two custom Claude Code skills: `ticker-price-snapshot`, `rsi-macd-signals`

See [[Projects/AutoTrader/Reference|Reference]] for external services/auth details and [[Projects/AutoTrader/References/TradingAgents|TradingAgents]] for the multi-agent design this project is drawing inspiration from.

## Notable Notes
- All secrets live in a gitignored `local_config.py` — nothing hardcoded (verified via grep by the research pass).
- The project's own `HANDOFF.md` is unusually detailed and kept current — treat it as living source of truth in the repo itself, this note is the vault-side summary of it.
- **Standing safety rule**: never place a live order without Ved's explicit real-time go-ahead, regardless of what the pipeline signals.
