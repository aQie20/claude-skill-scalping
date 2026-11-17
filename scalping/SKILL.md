---
name: scalping
description: Run and monitor live XAUUSD scalping on the user's MetaTrader 5 DEMO account through the metatrader MCP (M1/M5/M15), opening layered 0.50-lot trades with broker-side SL/TP, trailing protection, and a recurring 3-minute monitor. Use this whenever the user asks to trade, scalp, "run trade", "entry", "pantau trade", "buka posisi", "tutup posisi", set layers/cutloss/target profit on their MT5 demo, or invokes /scalping — even if they only say "trade gold sekarang" or "teruskan pemantauan". Not for backtesting or EA development (use the GoldADX/mt5 skills for that).
---

# Scalping — XAUUSD live demo (MT5)

The user (Malay speaker, no trading background) wants Claude to trade their MT5 **demo** account like its own, scalping gold on M1/M5/M15. Reply in plain Bahasa Melayu; explain jargon briefly. Be honest about losses and about mistakes — this skill exists because of mistakes made on 17 Nov 2026, and the user values candour over looking good.

## 0. Safety gate (every order, no exceptions)

Executing trades is only acceptable because the account holds no real money. So:
- Call `get_account_info` right before **every** `place_market_order`. If `account_type` is not `"demo"`, stop, place nothing, and tell the user.
- Never leave a position without SL and TP. `place_market_order` cannot set them, so immediately after each fill call `get_positions_by_symbol` (real fill price) and `modify_position`. If the modify fails, close that position at once.
- Symbol on this broker is `XAUUSD.vx` (verify with `get_symbols` group `*XAU*` if it ever fails). 0.50 lot = **$50 per $1** move; 1 pip = 0.10 USD.
- Server time = MYT − 6h (verify against the hook's MYT time).

## 1. User's trading rules (current defaults — the user may change them; latest instruction wins)

| Rule | Value |
|---|---|
| Lot | 0.50 per layer |
| Layers | open 2–4 at entry (choose the fewest that reach the target); max 10 total |
| Target | basket profit at TP ≥ **$300** (up to $1,000) |
| TP placement | **before** the next obstacle/zone — never through one |
| Risk:reward | basket profit at TP ≥ basket loss at SL (≥ 1:1), computed from **live bid/ask**, not candle closes |
| Cutloss | total risk ≤ 30% of balance |
| Direction | never hold BUY and SELL at the same time |
| User edits | if the user changed SL/TP themselves, respect it; ask before overriding |

## 2. Before any entry: map the market

Use the `smc` skill (H4→H1→M30→M15 structure, S/R, OB, FVG, sweeps, PROMP2U L01–L07) to build a zone map: supply/resistance zones, demand/support zones, flipped levels, and the H1/H4 invalidation levels. Note the previous day's high/low/range. Refresh the map when price leaves it or structure changes.

## 3. Entry discipline (hard-won — read every time)

1. **Zone ≠ signal.** Being in a zone is not a reason to enter.
2. **Trigger = a CLOSED M1 candle**: BUY = higher low + close above the previous M1 high; SELL = lower high + close below the previous M1 low. M5 must not oppose.
3. **No entries mid-range.** Only at/near a mapped zone.
4. **Breakouts: wait for the retest.** Never enter on the breakout candle itself; wait for price to come back to the broken level and reject it on M1. If no retest comes, let it go. (Chasing the break on 17 Nov doubled the risk for the same SL.)
5. **Obstacle check**: name every S/R between entry and TP. If a significant one sits before the $300 TP → no trade.
6. **Recompute R:R at the live ask/bid just before ordering.** By the time a check runs, price often has moved; a signal that was 1:2 at the close can be 1:0.8 now → skip it, and say so.
7. Layer additions only on a clear structure shift at a pullback into the zone, never to average down blindly, and only if total risk stays ≤ 30%.

## 4. Managing open trades (SL lessons)

- Initial SL: beyond the invalidating swing/sweep wick + ~0.5 buffer.
- **Protective SL must sit outside the current range**: beyond BOTH the entry AND the nearest M1 swing of the range + 0.5. An SL inside the range gets taken by noise (17 Nov: three trades stopped by $0.1–0.4 wicks).
- Trail only after price **leaves** the range and prints a new swing (new LH for sells / HL for buys); then SL = that swing ± 0.5. Never loosen an SL.
- If the user chose a far TP, protect at entry ± buffer outside the range — don't use a tight SL as a second take-profit.
- MT5 via this MCP cannot partially close; tell the user to do partials manually (double-click position → volume → Close), then re-check the remaining lots have SL/TP.

## 5. Monitoring loop

When the user says to monitor ("pantau", "teruskan"), create a recurring `CronCreate` job (`*/3 * * * *`) whose prompt restates: the demo gate, the rules table, the entry discipline, the SL rules, and the **current zone map + active setups with exact prices and invalidation levels**. Delete and recreate the job whenever rules or the zone map change (cron prompts can't be edited). Tell the user it is session-only and expires after 7 days; broker-side SL/TP protect trades if the session ends.

Each tick: `get_account_info` → `get_positions_by_symbol` → candles M1(8–15)/M5(4–6) (M15/H1 when structure questions arise) → act → report.

## 6. Reporting format

Per tick: 2–4 lines in Malay — price, floating P/L (and equity), action taken (or "Tiada entry" + the reason). On entry/exit, add a small table (entry, SL, TP, $ at risk, $ at target) and the running day result vs the starting balance. When a trade closes, verify with `get_account_info` balance rather than guessing.

When asked "kenapa", answer with the actual structure prices and own any mistake plainly; then record new lessons in memory (`feedback-live-entry-discipline.md`) and fold them into this skill and the cron prompt.

## 7. Honesty notes

- This scalping approach has **no backtested edge** (GoldADX project tests found none). Say so when relevant; demo results on one day prove little.
- 3-minute polling misses candle closes between checks; report missed signals honestly rather than chasing them late.
