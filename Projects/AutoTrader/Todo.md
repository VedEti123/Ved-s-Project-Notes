# AutoTrader — Todo

- [ ] Fill in `ROBINHOOD_CONFIG` (username/password/totp_secret) in `local_config.py`, then run `rh_probe.py` to verify the standalone Robinhood connection
- [ ] Design and run a backtest of the trading hypothesis (signal sentiment/confidence vs. realized price move) — the explicit gate before connecting the Foundry agent or order placement to `TradingDecisionSource/`
- [ ] Bring Docker/Kafka back up and decide which scrapers/consumers to restart (everything is currently stopped)
- [ ] Decide on flipping `DRY_RUN` off for Reddit + StockTwits consumers
- [ ] Restart EarningsHub live monitor (stopped since last host reboot)
- [ ] Start EarningsHub + TruthSocial signal consumers (never started)
- [ ] Extend `dataScraper.py` with yfinance fundamentals/news
- [ ] Evaluate Robinhood historicals as a replacement for yfinance
