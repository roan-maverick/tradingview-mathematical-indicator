# Specification

Every calculation and rule in `mathematical-indicator.pine`, in the order the script runs them. Defaults are in brackets and every one of them is an input.

---

## 0. Processing rules

- All state changes happen on **closed** candles (`barstate.isconfirmed`). Nothing changes inside a live candle.
- Higher-timeframe values are read from the previous **closed** higher-timeframe candle: `request.security(…, expr[1], lookahead = barmerge.lookahead_on)`. This is the standard non-repainting pattern.
- `ATR` is `ta.atr(14)` on the chart unless a timeframe is named.
- One trade at a time. Max trades per session [2].

---

## 1. Data requests

| Timeframe | Input | Used for |
|---|---|---|
| Daily | Candle 1 [`D`] | Story candle 1, daily levels, daily ATR, daily staircase (fallback) |
| 4H | Candle 2 [`240`] | Story candle 2, 4H levels, pre-session check, 4H ATR |
| Intraday levels | [`60`] | 1H floors and ceilings |
| Entry-zone levels | [`15`] | 15m floors and ceilings |
| Weekly | Story timeframe [`W`] | The staircase |
| Clean zones | [`15`] | Clean moves (falls back to the chart timeframe if it is lower than the chart) |

A level timeframe that is not higher than the chart's is skipped; the chart's own levels cover it.

---

## 2. ① Story

### 2.1 The two candles
```
daily_dir = sign(close − open) of yesterday's closed daily candle
h4_dir    = sign(close − open) of the last closed 4H candle
```
At a session start the 4H candle the session opens in is still running, so the last closed one is the candle right before the session.

### 2.2 Staircase
Swing highs and lows with `ta.pivothigh / ta.pivotlow` [strength 2] on the story timeframe:
```
UP    if last swing high > previous swing high AND last swing low > previous swing low
DOWN  if last swing high < previous swing high AND last swing low < previous swing low
BOX   otherwise
```
`na` until two swing highs and two swing lows exist. When the weekly staircase is `na`, the daily staircase is used (`story.staircase_tf = daily`).

### 2.3 Bias
```
daily_dir = h4_dir ≠ 0                     → bias = daily_dir                 "2 candles agree"
MIXED, rule "Tell the story" (default):
    at a daily ceiling (and not a floor)   → SELL
    at a daily floor (and not a ceiling)   → BUY
    staircase UP / DOWN                    → BUY / SELL
    otherwise                              → daily_dir (if flat: h4_dir)
MIXED, rule "Daily candle wins"            → daily_dir
MIXED, rule "Stand aside"                  → NONE
```
The bias, both candles, the staircase and the reason are frozen on the first candle of each session window.

---

## 3. ② Zone

### 3.1 Floors and ceilings
On each timeframe, with `n` candles on each side of the turn [daily & 4H: 1, others: 2]:
```
ceiling: candles n..2n−1 back are green, candles 0..n−1 back are red   → price = highest body top of the 2n candles
floor:   candles n..2n−1 back are red,   candles 0..n−1 back are green → price = lowest body bottom of the 2n candles
```
- **Break:** only when a candle of the level's own timeframe **closes** beyond it (above a ceiling, below a floor).
- **Strength:** a candle of that timeframe that trades through the level without closing through it adds one touch (`•` on the label).
- **Merge:** a new level within 0.15 × that timeframe's ATR of an unbroken level of the same side and timeframe adds a touch instead.
- **Keep:** the newest 3 unbroken levels per timeframe per side and the newest broken one (faded, dotted).

| Timeframe | Look |
|---|---|
| Daily | gold, 2 px, glow, `D CEILING` / `D FLOOR` |
| 4H | red / green, 2 px, glow |
| 1H | red / green, dashed |
| 15m and chart | magenta / cyan, dotted |

### 3.2 Positioning
```
daily_ceiling = lowest unbroken daily ceiling at or above price
daily_floor   = highest unbroken daily floor at or below price
position_pct  = (close − daily_floor) / (daily_ceiling − daily_floor) × 100
at_daily_ceiling = price ≥ (any unbroken daily ceiling − 0.25 × daily ATR)
at_daily_floor   = price ≤ (any unbroken daily floor + 0.25 × daily ATR)
```
Because daily levels only break on a daily close, price above an unbroken daily ceiling during the day still counts as "at" it.

**Come back tomorrow:** inside a session window, bias BUY and `at_daily_ceiling` blocks buys until the next daily candle. Bias SELL and `at_daily_floor` blocks sells.

### 3.3 Clean zones
On the clean zone timeframe [15m], a move runs while candles keep the same direction:
```
same-direction candle    → count + 1, extend the extreme close
one opposite candle      → a pause, the move continues
two opposite in a row    → a level formed, the move ends
```
When a move ends it becomes a clean zone if it had at least 3 same-direction candles [`Min candles`] and moved at least 1.5 × ATR of that timeframe [`Min move`]. The zone spans from the open of the first candle to the extreme close.

A zone stays active until a new 4H, 1H or 15m level forms inside it (8% padding from its edges). The newest 8 zones are kept. With **Entries must be inside a clean zone** [on], the breakout candle's high (buys) or low (sells) must be inside an active zone.

### 3.4 Breakout or pullback
Using the same pre-session 4H candle:
```
ran = |close − open| ≥ 60% × (high − low)  AND  (high − low) ≥ 1.0 × 4H ATR
mode = PULLBACK if ran AND its direction = bias, else BREAKOUT
```
In PULLBACK mode a buy range must reach down to `4H high − 38.2% × (4H high − 4H low)`; a sell range must reach up to `4H low + 38.2% × (4H high − 4H low)`.

---

## 4. Sessions

| Session | Default window | Time zone | Default |
|---|---|---|---|
| Asia | 09:00–10:00 | Asia/Tokyo | on |
| London | 08:00–09:00 | Europe/London | on |
| New York | 08:30–09:30 | America/New_York | off |
| NYSE open | 09:30–10:30 | America/New_York | on |
| Power hour | 15:00–16:00 | America/New_York | off |

Each window is the first hour of that session. If windows overlap, the first one in the list wins. Breakout candles must close inside a window; the range may start before it. Time zones follow daylight saving automatically.

---

## 5. ③ Strat: the trigger

### 5.1 Range
For every closed candle, looking back `k = 1…20` candles [`Range: max candles`]:
```
top_k    = highest body top of candles 1..k back
bottom_k = lowest body bottom of candles 1..k back
opp_k    = red candles among them (green for sells)
stop growing when top_k − bottom_k > 4 × ATR
valid if k ≥ 3, opp_k ≥ 2, top_k − bottom_k ≥ 0.5 × ATR, and this candle closes beyond the range
```
The **largest** valid `k` is used.

**Liquidity grab:** some candle inside the range traded below the lowest low of the older range candles and closed back above it (mirrored for sells).

### 5.2 Breakout candle
All of:
- In the bias direction (green for BUY, red for SELL), body ≥ 50% of its range.
- Closes beyond the range (above the top for buys, below the bottom for sells).
- Its range (high − low) ≤ 50% of the range height.
- Session window live, session trade count below the limit, bias not NONE.
- Not blocked today, PULLBACK depth met (if in that mode), inside a clean zone (if required).

A failed check prints a grey `✕ reason` label: `AT DAILY CEILING · come back tomorrow`, `breakout candle > 50% of the range`, `pre-session ran · wait for a pullback`, `no clean zone here`.

When everything passes:
```
order price = breakout high (BUY STOP)  /  breakout low (SELL STOP)
stop loss   = breakout low − buffer     /  breakout high + buffer      [buffer 0 ticks]
```

### 5.3 Rubber band, fill, cancel
On each later closed candle while the order works:
```
wick      = min(open, close) − low  (buys)  /  high − max(open, close)  (sells)
wick_ok   = wick ≥ 0.10 × ATR  AND  wick ≥ 20% of the candle's range
hit       = high ≥ order price (buys)  /  low ≤ order price (sells)

if hit:
    dip already seen, or (wick_ok AND the candle opened before the order price)
        → FILL at max(open, order price) (buys) / min(open, order price) (sells)
    else → CANCEL "nipple wick · broke with no dip first"
else:
    wick_ok                                 → dip seen (② RUBBER BAND ↺)
    close back inside the range             → CANCEL "closed back inside the range"
    6 candles since the breakout            → CANCEL "stop order expired"
```
When the dip and the break happen inside the same candle, the dip is assumed to come first. If that candle also closes beyond the stop, the trade exits at the stop on the same candle.

---

## 6. ④ Trade management

At the fill:
```
risk   = |entry − stop|
1R     = entry ± risk
target = first unbroken level (any timeframe) at or beyond 1R in the trade's direction,
         capped at entry ± 3R ("3R cap · open space")
         "1:1 only" mode: target = 1R
pre_level = nearest level between entry and 1R (if any)
```
On each later closed candle, in this order:
1. **Stop:** low ≤ stop (buys) → exit at the stop, or at the open if it gapped through.
2. **1R:** first touch → bank 50% [`Close % at 1R`] and move the stop to entry. In `1:1 only` mode → exit everything at 1R.
3. **Pre-level:** touching a level before 1R moves the stop to entry.
4. **Target:** touch → exit the rest at the target.
5. **Too fast:** open result ≥ 1.5R and 3 straight candles in the trade's direction, each closing beyond the previous close → `⚡ TOO FAST · TAKE PROFIT`, exit at the close [auto-close on].
6. **Time:** still open 240 minutes after entry → exit at the close.

### Result in R
```
banked  = 0.5 if the 1R partial happened, else 0
rest    = 1 − banked
result  = banked × 1 + rest × (exit − entry) / risk          (sign flipped for shorts)
```
win > +0.05R, loss < −0.05R, breakeven otherwise.

---

## 7. On-chart record

Counts every closed trade on the loaded history: trades, wins, breakeven, losses, win rate, net R, average R, profit factor (gross wins / gross losses). No fees, commissions or slippage.

---

## 8. Visuals

| Element | Meaning |
|---|---|
| Bias aura | EMA(hlc3, 21) core ± 1.2 × ATR(21): cyan buyers wave above, purple sellers wave below, core colored by bias. Visual only, never part of a signal |
| Window tint | Background tint in the bias color during session windows |
| Session banner | Session, bias, the two candles, the staircase and the mode |
| Range box | Neon box over the range, `LIQUIDITY GRAB ✓` when present |
| Breakout candle | Glowing outline and `① BREAKOUT` |
| Stop order line | Dashed line and `BUY STOP` / `SELL STOP` tag |
| `② RUBBER BAND ↺` | The dip candle |
| `③ ENTRY` | Fill, red risk box, green reward box, `1R → BREAKEVEN` line, `TARGET · source` tag |
| `🛡 RISK FREE` | Stop moved to entry |
| `⚡ TOO FAST · TAKE PROFIT` | Too-fast exit |
| Result tag | `✓ +1.3R`, `✕ −1.0R`, `● +0.0R` with the reason |
| Candle colors | Breakout cyan / magenta, dip gold, entry green / red |

---

## 9. Dashboard

| Section | Rows |
|---|---|
| Header | Name, market, timeframe, session status |
| ① Story | Yesterday, last 4H candle, staircase (each with ✔ agrees / ◐ disagrees / ✕ flat), **BIAS**, why |
| ② Zone | Positioning meter, daily ceiling and floor with distance %, clean zone, pre-session 4H, mode, today (open or come back tomorrow) |
| ③ Strat | Range, breakout candle, rubber band, entry, **state**, next step, last skip |
| Active trade | Stop, 1R, target with R and source, open P&L meter |
| On-chart record | Trades, win rate, net R and average R, profit factor |

---

## 10. Alerts

- `alert()` sends the JSON packet described in [PACKET_SCHEMA.md](PACKET_SCHEMA.md) with `alert.freq_all`, only on closed candles.
- Three `alertcondition()` entries exist for simple notifications: stop order placed, entry, trade closed.
- Data Window values: session bias, state (0 looking, 1 stop order, 2 in trade), trade side.

<!-- Maintained by repo owner -->
