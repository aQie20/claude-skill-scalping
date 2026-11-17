# claude-skill-scalping

A Claude Code skill for running and monitoring XAUUSD scalping on a **MetaTrader 5 demo account** through a MetaTrader MCP server (M1/M5/M15), with layered 0.50-lot entries, broker-side SL/TP, trailing protection and a recurring 3-minute monitor.

## Install

Copy the `scalping/` folder into `~/.claude/skills/`. It triggers on requests such as "run trade", "pantau trade", "scalping", or `/scalping`.

## Requirements

- Claude Code with a MetaTrader MCP server (`get_account_info`, `place_market_order`, `modify_position`, `get_candles_latest`, ...)
- An MT5 **demo** account (the skill refuses to trade if the account is not demo)
- Optional: an `smc` analysis skill for zone mapping

## Disclaimer

Educational / demo use only. Not financial advice. The strategy has no backtested edge.
