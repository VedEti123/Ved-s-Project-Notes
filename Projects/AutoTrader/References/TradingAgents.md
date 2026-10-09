# TradingAgents (reference for AutoTrader)

Open-source multi-agent LLM framework (Tauric Research, built on LangGraph) that simulates a trading firm's org structure to reach a buy/sell/hold decision. Research-only per its own disclaimer, not for live trading. Backed by an arXiv paper (2412.20138) and a "Trading-R1" report.

Located at `/Users/vedeti123/Documents/TradingAgents`. Used here as design inspiration, not a dependency of AutoTrader.

## Stack
- Python 3.13, `pyproject.toml`/`uv.lock`, pip package or Docker.
- Orchestration: LangGraph.
- Pluggable LLM providers (OpenAI, Gemini, Claude, Grok, DeepSeek, Qwen, GLM, MiniMax, OpenRouter, Ollama, Azure OpenAI) — config picks a "deep-think" model and a separate "quick-think" model per run.
- Data vendors: yfinance (default) or Alpha Vantage, configurable per data category or per tool.
- Persistence: SQLite for LangGraph checkpoints, Markdown decision log for memory/reflection.

## Architecture / Pipeline
1. **Analyst team** — fundamentals, market (technicals like MACD/RSI), news, sentiment, social media analysts each independently report on a ticker.
2. **Researcher team** — bull vs. bear researcher debate the analyst reports.
3. **Research manager** — synthesizes the debate.
4. **Trader** — converts research into a proposed trade (timing/size).
5. **Risk management** — aggressive/conservative/neutral personas debate the proposed trade.
6. **Portfolio manager** — final approve/reject call.

## Ideas Worth Borrowing for AutoTrader
- **Decompose the decision, not just the pipeline**: analyst → debate → trade proposal → risk debate → final approval, instead of one monolithic "decide" prompt. Keeps each agent's context narrow and each output independently inspectable/loggable — relevant since AutoTrader's `signal_agent.py` is currently a single shared extraction step.
- **Structured debate as a primitive**: bull/bear and aggressive/conservative/neutral roles force counterarguments to surface before a decision is committed, rather than just aggregating sentiment signals.
- **Persistent decision-log + reflection loop**: after each run, fetch the ticker's realized return (raw + alpha vs. a regional benchmark index), write a short reflection, and feed recent same-ticker/cross-ticker lessons back into the next run's decision prompt. A cheap self-improving loop without fine-tuning — could plug into AutoTrader's Robinhood execution path later to actually learn from outcomes.
- **Config-driven LLM tiering**: cheap/fast model for routine steps, stronger model reserved for the final reasoning step, to control cost. AutoTrader currently uses one Azure AI Foundry agent for all signal extraction — this is a candidate cost optimization.
- **Vendor abstraction with per-tool override**: data vendor defaults set per category, overridable per tool — lets data sources be swapped without touching agent logic. Relevant to AutoTrader's own planned yfinance/Robinhood-historicals swap (see [[Projects/AutoTrader/Todo|Todo]]).
- **Checkpoint/resume via LangGraph state persistence**, opt-in, resumable per ticker — worth considering if AutoTrader's pipelines grow long-running.
- **Region-aware benchmarking**: maps ticker suffix (`.NS`, `.T`, `.L`, etc.) to the correct regional index for alpha calc instead of hardcoding SPY.
