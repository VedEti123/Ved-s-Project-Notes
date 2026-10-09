# AutoTrader — Reference

## External Services / APIs
| Service | Purpose | Auth |
|---|---|---|
| Kafka (local Docker) | Streams raw scraped text between scrapers and the signal-extraction consumers | container `AutoTrader`, no restart policy — start manually each session |
| Azure AI Foundry | LLM signal-extraction agent | `az login` / `DefaultAzureCredential` |
| Robinhood (via MCP) | Account data + trade execution (execution unused so far) | `claude mcp login robinhood --no-browser`, ~50 tools exposed. Session-bound to Claude Code — only usable interactively. Stress-tested: ~100 calls/~4min, zero rate-limit issues |
| Robinhood (direct, `robin-stocks`) | Same broker, second/separate auth path for `TradingDecisionSource/`'s standalone service (needs to run headless, MCP doesn't) | `ROBINHOOD_CONFIG` in `local_config.py` (username/password + optional `totp_secret` for non-interactive 2FA) — currently blank, not yet verified |
| Reddit API | Source scraping | OAuth via `praw` |
| Truth Social | Source scraping | session-token scraping via `truthbrush` |
| StockTwits | Source scraping | public, no auth |
| Earnings Hub / SavvyTrader | Live earnings call audio | Cognito auth (`pycognito`) + WebSocket |

## Watchlist
AAPL, MSFT, AMZN, PLTR, META, NVDA, TSLA, INTC, NFLX, JPM, GS, C, LMT, MA, SHEL

## Glossary
- **Signal** — structured output of the signal-extraction agent: `{symbol, sentiment, confidence, topic, key_quote, reasoning}`
- **DRY_RUN** — env flag; when `1`, the pipeline runs end-to-end but does not spend on live agent calls
- **HANDOFF.md** — living source-of-truth doc inside the AutoTrader repo itself; check it for the most current state before assuming this vault note is up to date
