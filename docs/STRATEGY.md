# The Strategy · every rule, in plain words and in numbers

The strategy has three parts that always run in the same order: **story → zone → strat**. Most of the work is the first two. The entry itself is a small, mechanical last step.

---

## ① Story: which way to trade

### The two candles
- **Candle 1:** yesterday's daily candle (the last *closed* daily candle).
- **Candle 2:** the last 4H candle that *closed* before your session starts. The 4H candle the session opens in is still running, so it is never used.

The daily candle says what the previous day did. The 4H candle says what happened right before your session (for the New York open that is mostly London).

| Daily | 4H | Bias |
|---|---|---|
| green | green | **BUY only** |
| red | red | **SELL only** |
| different | different | **MIXED** → the story decides |

The bias is frozen at the first candle of each session window and used for the whole session. With a bias there is no guessing: every pullback is a buying opportunity (or a selling one), never a reason to flip.

### Mixed bias: tell the story
Default rule (`Tell the story`), checked in this order:
1. Price is pressed against a **daily ceiling** → SELL (a mixed day can sell from resistance).
2. Price is pressed against a **daily floor** → BUY.
3. Otherwise follow the **weekly staircase**: staircase up → BUY, staircase down → SELL.
4. Weekly is a box (or there is not enough history) → the **daily candle** wins, it is the stronger candle.

Other options: `Daily candle wins` or `Stand aside`.

### The staircase
- **Staircase up:** higher highs **and** higher lows.
- **Staircase down:** lower highs **and** lower lows.
- **Box:** anything else (chop).

Swing points use a strength of 2 candles on each side. The weekly staircase needs several months of history; on short intraday histories the indicator falls back to the daily staircase and the dashboard says `Daily story`.

The higher timeframe is the instruction manual for what the lower timeframe should build: start from the weekly, then the daily, then the session.

---

## ② Zone: where to trade

### Floors and ceilings (support and resistance)
- **Ceiling:** green candle(s) followed by red candle(s). The level sits on the highest **body** in that turn, not the wick.
- **Floor:** red candle(s) followed by green candle(s), on the lowest body.
- Daily and 4H: 1 candle on each side of the turn. 1H, 15m and chart timeframe: 2 candles on each side.
- The close is where price actually settled, so bodies are used, not wicks.

**Breaks:** a level only breaks when a candle of its **own** timeframe closes through it. A 5 minute close through a 15 minute level does not count. A daily level needs a daily close.

**Strength:** every time price wicks through a level but does not close through it, the level gets stronger. The dots on the label (`D CEILING •••`) count those failed attempts.

Levels within 0.15 ATR (of their own timeframe) of each other are merged. The 3 most recent levels per timeframe per side are kept.

### Positioning: are we at the top or the bottom?
The meter shows where price sits between the nearest unbroken daily floor (0%) and daily ceiling (100%).
- Buying low in that range is the best buy, the middle is fine, buying right under the ceiling is a gamble.
- **Bias BUY and price pressed against a daily ceiling** (within 0.25 daily ATR, or poking above it without a daily close) → no buys for the rest of the day. Only a daily close can confirm the break, so **come back tomorrow**. Same for sells at a daily floor.

### Clean zones
A clean zone is a stretch price already crossed in one direction without building a floor or ceiling on the way:
- 3 or more candles in one direction on the 15 minute chart,
- the move is at least 1.5 ATR,
- one small opposite candle is allowed as a pause; two opposite candles in a row build a level and end the move.

If price moved through an area cleanly before, it can usually move through it cleanly again: it is the path of least resistance. A zone stays active until a new 15m, 1H or 4H level forms inside it (then it is messy). By default an entry must be inside an active clean zone.

### Breakout or pullback
Look at the pre-session 4H candle (the same candle 2 from the story):
- It **ranged** → breakouts are taken straight away (`BREAKOUTS ON`).
- It **ran in the bias direction** (body at least 60% of the candle and size at least 1 ATR of the 4H) → the session usually wants better prices first (`WAIT FOR A PULLBACK`). For buys, the range must sit at least 38.2% back down inside that 4H candle; for sells, 38.2% back up.

That is how the strategy buys the highs on some days and the dips on others.

---

## ③ Strat: the trigger

Only inside the **first hour** of an enabled session (Asia 09:00 Tokyo, London 08:00 London, New York 08:30, NYSE 09:30 New York, power hour 15:00 New York: each one hour long and editable). The range itself may start before the window.

### 1. The range (structure)
- The last 3 to 20 candles before the breakout candle, capped by their **bodies**.
- At least **2 opposite candles** inside it (red candles for a buy setup, green for a sell setup): sellers tried and failed.
- Height between 0.5 and 4 ATR. The widest valid range is used, a bigger range holds more orders.
- A candle that dips under the range's earlier lows and closes back inside is marked as a **liquidity grab** (`LIQUIDITY GRAB ✓`): the spring is loaded.

### 2. The breakout candle
- Closes **outside** the range (above the top for buys, below the bottom for sells). A wick outside does not count.
- Is a convincing candle: body at least 50% of the candle (no doji).
- Is not bigger than **50% of the range height**. A bigger candle makes the stop too wide and price usually pulls back first.

The indicator then places a **stop order** at the breakout candle's high (buy stop) or low (sell stop). The stop loss goes under the breakout candle's low (over its high for sells).

### 3. The rubber band
Before the stop order may fill, a later candle has to dip back first: a wick against the trade of at least 0.1 ATR and at least 20% of that candle. It does not have to reach the range.
- This is the last chance for the other side. If they fail, price snaps back like a rubber band.
- A deeper wick is a bigger spring.

### 4. The entry
- The stop order fills when price trades through it **after** the dip.
- If the dip and the break happen inside the same candle, the candle must open below the order (above for sells) and the dip is assumed to come first.
- Price breaks the order with no dip first (a tiny "nipple" wick) → **skipped**. That break usually traps late buyers and comes back into the range.
- A candle **closes back inside the range** before the fill → setup cancelled. A close outside the range in the other colour is fine.
- Not filled within 6 candles → cancelled.

Max 2 trades per session, one at a time.

---

## ④ Exits

- **Stop loss:** under the breakout candle (optional buffer in ticks).
- **1:1:** when price reaches 1R, the stop moves to breakeven and 50% is banked (`Close % at 1R`). The trade can no longer lose.
- **A level before 1:1:** if price hits a level before 1R, the stop moves to breakeven there.
- **Target:** the first floor or ceiling (any timeframe) at or beyond 1R. If nothing is in the way within 3R, the target is 3R.
- **Too fast:** once the trade is up 1.5R or more and 3 straight candles push in the trade's direction, the indicator flags `⚡ TOO FAST · TAKE PROFIT` and (by default) closes the trade. The faster it moves, the sooner it usually pulls back.
- **Time:** anything still open after 240 minutes is closed.
- `1:1 only` mode closes the whole trade at 1R.

Results are counted in R (1R = the distance from entry to stop). With 50% banked at 1R and the rest stopped at breakeven, the trade counts as +0.5R.

---

## What is simulated and what is not

- All decisions are made on closed candles. Higher-timeframe values come from closed candles (`expr[1]` + `lookahead_on`): nothing repaints.
- Fills are simulated from candle highs and lows. Real fills can slip, especially at session opens.
- The on-chart record ignores fees, commissions and slippage.
- Sessions use the time zones you set (defaults: Tokyo, London, New York), so they follow daylight saving automatically.
