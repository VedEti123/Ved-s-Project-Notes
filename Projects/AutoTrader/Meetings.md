# AutoTrader — Log

## 2026-08-21
Set up this vault. Claude reviewed the AutoTrader repo and `HANDOFF.md` to populate Overview/Reference/Todo. Current state: Reddit + StockTwits signal consumers running in `DRY_RUN=1`; EarningsHub monitor and EarningsHub/TruthSocial consumers stopped/unstarted; Robinhood MCP connected but no orders ever placed.

## 2026-08-21 (later same day)
Stress-tested the Robinhood MCP connection: ~100 read-only calls (quotes, option chains, historicals) across the full watchlist in a conservative ramp over ~4 minutes, zero errors or throttling. Stopped deliberately short of finding the actual ceiling — real OAuth-linked brokerage account, and 100/4min already far exceeds this project's actual usage.

Started building the trade-decision side: scaffolded `TradingDecisionSource/` with a direct `robin-stocks` Robinhood login (`rh_client.py` + `rh_probe.py`), separate from the MCP connection since MCP only works in a live Claude session and this needs to run headless eventually. Explicitly did **not** wire it to the Foundry signal agent or order placement — decided to backtest the trading hypothesis (signal sentiment/confidence vs. realized price move) first, before any live capital or automated decision-making. `ROBINHOOD_CONFIG` credentials still need to be filled in and the connection verified via `rh_probe.py`.

Also found everything (Docker included) currently stopped — normal between-sessions state, not a regression. Updated `HANDOFF.md` and this vault (Overview/Todo/Reference) to reflect all of the above.
