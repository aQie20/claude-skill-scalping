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
| Target | basket profit at TP **$100–1,000** (updated 18 Nov 09:02 MYT; earlier $300–1,000) |
| TP placement | **before** the next obstacle/zone — never through one |
| Risk:reward | basket profit at TP ≥ basket loss at SL (≥ 1:1), computed from **live bid/ask**, not candle closes |
| Cutloss | total risk ≤ 30% of balance |
| Min R:R (18 Nov 20:30) | **1:1** accepted (user) |
| Per-trade risk (18 Nov 19:39) | flexible: max **$500** per trade; SL by structure, then layers = most that fit ($50 per $1 per layer, 2–4 layers); SL needing > $5 with 2 layers → skip |
| Direction | never hold BUY and SELL at the same time |
| User edits | if the user changed SL/TP themselves, respect it; ask before overriding |

## 2. Before any entry: map the market

Use the `smc` skill (H4→H1→M30→M15 structure, S/R, OB, FVG, sweeps, PROMP2U L01–L07) to build a zone map: supply/resistance zones, demand/support zones, flipped levels, and the H1/H4 invalidation levels. Note the previous day's high/low/range. Refresh the map when price leaves it or structure changes.

## 2b. PRE-ENTRY CHECKLIST (user, 21 Nov 2026 — read before EVERY order or pending order)

Answer all six in the report. Any "no" → **no trade**, say which one failed.

1. **Break confirmed?** M15 (or M5) CLOSED clearly beyond the zone (≥ $0.5), not a wick, not an M1-only close. *(#1, #5)*
2. **Retest + structure?** Price came back to the broken level and printed an M1 LH (SELL) / HL (BUY) there. No chasing the breakout candle. *(#4b)*
3. **Signal still valid?** Closed-M1 trigger, and live price has NOT moved back past the trigger level nor run > $0.5 toward TP. *(#7, #11, #13)*
4. **SL right?** Beyond the real retest high/low + 0.5; distance ≤ $5 (2 layers); larger than the current M5 candle range; ≥ $2 from live price. *(#5, #12, #15)*
5. **Room & location?** At the EDGE of a zone, not mid-range; TP before the next obstacle; room to TP ≥ SL distance (R:R ≥ 1 from the likely fill). *(#12, #13)*
6. **Market calm?** Last 10–15 min M1/M5 swings smaller than the SL; not the chaotic NY open / broker daily close; no opposite open position. If I called it choppy, it IS choppy. *(#9, #12)*

After entry: when the written early-exit rule fires, exit — no reinterpretation. *(#6)*
Track record behind this list (18–20 Nov): all 5 wins followed it (model 4e); all 6 losses broke at least one item.

## 3. Entry discipline (hard-won — read every time)

1. **Zone ≠ signal.** Being in a zone is not a reason to enter.
2. **Trigger = a CLOSED M1 candle**: BUY = higher low + close above the previous M1 high; SELL = lower high + close below the previous M1 low. M5 must not oppose.
3. **No entries mid-range.** Only at/near a mapped zone.
4. **A range break counts only when the M15 candle CLOSES beyond the whole zone.** An M1 close below/above that the M15 closes back inside is a sweep (L04), not a breakout — the range is still alive, no breakout-retest trade (18 Nov 11:29: −$367).
4a. **No break by cents, no SL inside the retest.** The M5/M15 close must be clearly beyond the zone (~$0.5+), and on a breakout-retest the SL goes above the retest high (the candle that tagged the broken level) + buffer — not a minor M1 LH inside that zone. Fit layers to $500 (19 Nov 00:13: −$386).
4c. **Void signals and tight stops.** If price has moved back past the trigger candle's level, the signal is void — not a "better price". Never enter when the structural SL is < ~$2 from the live price: a normal spike can pass it before `modify_position` runs ("Invalid stops" → forced close, 19 Nov 02:30: −$292). After 3 losses in a row, stop for the session and tell the user.
4d. **Choppy market = no M1 triggers.** If the last 10–15 min show wide wicks both ways (moves larger than your planned SL), skip M1 signals until M5 closes settle; the SL must exceed the current M1 wick size (19 Nov 09:22: −$214, an hour after calling the market choppy).
4e. **Model setup (user-approved, 19 Nov 19:36 SELL, +$450.50 — keep this process; no volume filter):** M15 CLOSES clearly beyond the range (~$4) → wait for the retest of the broken level → M1 lower highs at the retest → closed-M1 trigger → SL above the real retest high +0.5 → TP before the next support; market calm (M1 wicks < SL). Skip every signal in chop until this lines up.
4b. **Breakouts: wait for the retest.** Never enter on the breakout candle itself; wait for price to come back to the broken level and reject it on M1. If no retest comes, let it go. (Chasing the break on 17 Nov doubled the risk for the same SL.)
5. **Obstacle check**: name every S/R between entry and TP. If a significant one sits before the $300 TP → no trade.
6. **Recompute R:R at the live ask/bid just before ordering.** By the time a check runs, price often has moved; a signal that was 1:2 at the close can be 1:0.8 now → skip it, and say so. Fills can slip further in a fast market (20 Nov 01:51: filled $1.3 past the checked bid, R:R 1.26 → 0.57); if price has already run > ~$0.5 toward TP so the likely-fill R:R < 1, skip.
7. Layer additions only on a clear structure shift at a pullback into the zone, never to average down blindly, and only if total risk stays ≤ 30%.

## 3b. Two entry methods (user, 18 Nov 13:31)

- **Pending orders** (sell/buy limit) at mapped zones with SL/TP attached — so 3-minute polling doesn't miss the move.
- **Instant one-click** market orders after a closed-M1 trigger in a zone.
- When a PO fills, watch M1/M5: if structure supports the trade (SELL: M1 LH/LL continue, M5 closes red), hold to TP. If structure clearly turns against it (SELL: M1 closes above its last lower high AND M5 closes green above the prior M5 high), exit early at the smallest loss possible — on a small M1 pullback or immediately if price nears SL. One noisy candle is not a reason to exit.
- **When the written early-exit condition fires, exit.** Don't reinterpret a closed candle as "just a wick" to justify holding (19 Nov 01:20: held at −$179, then full SL −$453).
- Count PO + open layers together for the max-10 and ≤30% risk limits.

### 3c. STOP orders at the trigger level (user-approved 20 Nov 16:34 MYT — keep the 3-min cron, no 1-min polling)

3-minute polling kept missing valid signals in fast markets (20 Nov: three valid triggers fired and ran between checks). Fix: once the setup is armed, let the broker execute the trigger.
- **Arm only after the 4e conditions are in place:** clear M5/M15 close beyond the zone → retest → BUY: a M1 higher low has formed at the retest (SELL: a M1 lower high).
- **Place** BUY STOP at the previous M1 high + ~0.3 (SELL STOP at the previous M1 low − ~0.3), 2 layers × 0.50 (two pending orders), with SL = beyond the HL/LH (retest low/high) ∓ 0.5 and TP before the next obstacle **attached in the same call**. Check before placing: SL distance from the stop price ≤ $5, R:R ≥ 1 from the stop price, stop price ≥ ~$0.5 from the live price.
- `place_pending_order` takes only 'BUY'/'SELL'; the MCP picks stop vs limit from price vs market (BUY above ask = buy stop). After placing, read `get_pending_orders_by_symbol` and confirm the type is a STOP and SL/TP are set; if not, cancel at once.
- **Cancel** the pending order when: price breaks the HL/LH (signal void), the structure changes so R:R < 1, the opposite setup appears, or it is unfilled after ~15 min (5 ticks).
- **Location + noise (20 Nov −$489):** arm only at the edge of a zone where the room to the next obstacle ≥ the SL distance — never mid-range under resistance; the SL must also exceed the current M5 candle range, not just M1 wicks.
- **MCP data can be stale:** if a candle high/low crossed the stop price but the order still shows "PLACED", or a cancel returns 10013 "Invalid request", call `get_deals` with `from_date` = today before reporting anything.
- When it fills, manage as a normal trade (no SL moves while trailing is on hold; early exit per 3b). Never have BUY and SELL pending/open at the same time.

## 4. Managing open trades (SL lessons)

> **TRAILING ON HOLD (user, 18 Nov 13:47 MYT):** the user is testing a trailing EA. Until the user says otherwise: set the initial SL/TP at entry only, then do NOT move SL (no breakeven, no protective SL, no trailing). If the SL/TP changes, assume the EA or user did it and respect it. The rest of this section is paused, except early exit when structure clearly turns against a PO fill (section 3b).

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

- Stay inside this skill: judge setups only by these rules, the live chart and this account's own results. Don't bring in other projects (user's instruction, 18 Nov). Demo results on one day prove little — say so when asked about edge.
- 3-minute polling misses candle closes between checks; report missed signals honestly rather than chasing them late.
