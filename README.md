# ◆ The Mathematical Indicator · 2-Candle Bias Strategy

**A free TradingView indicator that runs a complete intraday strategy for you, and streams every step as JSON so an LLM (Claude Fable 5.2, Grok, any model) can turn it into a 24/7 trading bot.**

Two candles pick the direction. Floors and ceilings are drawn on candle bodies. Trades only happen in the first hour of a session, inside clean zones, after a range, a strong breakout candle and one dip back. Stop under the breakout candle, 1:1 first, breakeven, then the next level.

Pine Script® v6 · overlay · no repainting · MIT licensed · works on futures, forex, gold, indices and crypto (5 to 15 minute charts).

---

## The strategy in plain words

| Step | What it does | What you see on the chart |
|---|---|---|
| **① Story** | Yesterday's daily candle + the last 4H candle before your session. Both green = **only BUY**. Both red = **only SELL**. They disagree = **MIXED**: the story decides (daily level, then the weekly staircase, then the daily candle). | A banner at every session open: `NYSE · ▲ BUY ONLY`, the two candles, and the mode |
| **② Zone** | Floors and ceilings on candle **bodies**. A level only breaks when a candle of its **own** timeframe closes through it; wicks that fail make it stronger (`•••`). No buys under a daily ceiling, no sells over a daily floor: come back tomorrow. Only trade inside **clean zones** (stretches price already crossed without stopping). If the pre-session 4H candle already ran in the bias direction, wait for a pullback. | Gold daily levels, red/green 4H and 1H levels, dotted 15m and chart levels, indigo `CLEAN ZONE` boxes |
| **③ Strat** | Inside the first hour: a range with 2+ opposite candles → one strong candle closes outside it (not a doji, not bigger than half the range) → the next candle dips back (**the rubber band**) → a stop order at the breakout candle's high (low for sells) fires. Stop loss under the breakout candle. | Neon range box, `① BREAKOUT`, `BUY STOP 21,345.25`, `② RUBBER BAND ↺`, `③ ENTRY ▲` |
| **④ Exit** | At 1:1 the stop moves to breakeven and part of the trade is banked. The rest rides to the next level. If price runs too fast in a straight line, take it. | Risk/reward boxes, `1R → BREAKEVEN`, `TARGET · 1H ceiling`, `⚡ TOO FAST · TAKE PROFIT`, result in R |

Full rules with every number: **[docs/STRATEGY.md](docs/STRATEGY.md)**

---

## Install (2 minutes)

1. Open [`mathematical-indicator.pine`](mathematical-indicator.pine) and copy everything.
2. In TradingView open the **Pine Editor** (bottom of the chart), paste, **Save**, then **Add to chart**.
3. Use a **15 minute** chart to learn it, drop to **5 minutes** once you know the setup.
4. Open the settings (⚙) → **🕐 02 · Sessions**: switch on the sessions you trade (Asia, London and the 9:30 NYSE open are on by default). It only hunts setups in the first hour of each.

That's it. The dashboard (top right) tells you the bias, where price sits, and what the indicator is waiting for next.

---

## The dashboard

```
◆ THE MATHEMATICAL INDICATOR
STORY · ZONE · STRAT
Market        NQ1!                    TF 5m
Session       ● NYSE · LIVE (first hour)
① STORY · THE 2 CANDLES
Yesterday     ▲ Green                   ✔
Last 4H candle ▲ Green                  ✔
Weekly story  ▲ Staircase up            ✔
BIAS          ▲  BUY ONLY
Why           2 candles agree
② ZONE · WHERE TO TRADE
Positioning   ██████░░░░          58% mid
Daily ceiling 21,512.25   +0.84%
Daily floor   21,104.00   −1.12%
Clean zone    Inside one                ✔
Mode          BREAKOUTS ON
Today         Open for trades           ✔
③ STRAT · THE TRIGGER
Range         7 candles · liquidity grab ✔
Breakout candle closed outside the range ✔
Rubber band wick waiting for the dip    ◐
Entry         buy stop 21,345.25        ◐
State         ◆  BUY STOP WORKING
Next          Need a dip (wick) before price breaks 21,345.25
ON-CHART RECORD
Trades · Win rate · Net R · Profit factor
```

The record is this chart's own history with no fees or slippage. Use it to study the setup, not as a promise.

---

## Alerts (for your LLM / 24/7 bot)

Every step is sent as one JSON alert: `SESSION_BIAS`, `STOP_ORDER_PLACED`, `STOP_ORDER_CANCELLED`, `ENTRY`, `BREAKEVEN`, `TOO_FAST_TAKE_PROFIT`, `EXIT`. Each one carries the full story (bias, the 2 candles, staircase), the zone (daily ceiling/floor, positioning, clean zone, mode), the setup, the order and the trade.

Create **one** alert on the chart: condition **The Mathematical Indicator → Any alert() function call**, then paste a webhook URL if you want the alerts sent somewhere. Give this indicator to Claude Fable 5.2, Grok or any LLM and ask it to **"Build a 24/7 Trading Bot"** around these alerts. Paper trade for at least two weeks before you connect real money.

---

## The math (why the rules are built this way)

The edge of any strategy is its expectancy per trade:

```
E = (win rate × average win) − (loss rate × average loss)        measured in R (1R = your risk)
```

At 1:1 every win pays +1R and every loss costs −1R, so the break-even win rate is 50%:

| Win rate at 1:1 | Result per 10 trades | Expectancy per trade |
|---|---|---|
| 50% | 0R | 0.0R |
| 60% | +2R | +0.2R |
| 70% | +4R | +0.4R |

Breakeven stops and banking part at 1R exist to cut the average loss, and the filters (2 candles, daily levels, clean zones, first hour only) exist to push the win rate up. No strategy wins every trade. Check your own numbers in the on-chart record and in paper trading.

---

## Key settings

| Group | Setting | Default |
|---|---|---|
| 02 Sessions | Asia 09:00 Tokyo · London 08:00 · NYSE 09:30 New York (first hour each) | on |
| 02 Sessions | Max trades per session | 2 |
| 03 Story | Mixed bias rule | daily level → weekly staircase → daily candle |
| 04 Levels | Daily & 4H turn: 1 candle each side · 1H, 15m & chart: 2 each side | |
| 05 Clean zones | 3+ candles one way, move ≥ 1.5 ATR, entries must be inside one | on |
| 06 Positioning | "At" a daily level = within 0.25 daily ATR · block that direction for the day | on |
| 07 Breakout or pullback | Pre-session 4H ran (body ≥ 60%, size ≥ 1 ATR) → pullback ≥ 38.2% first | on |
| 08 Trigger | Range 3–20 candles, 2+ opposite · breakout body ≥ 50% · breakout ≤ 50% of range · wick ≥ 0.1 ATR and ≥ 20% of its candle · order lives 6 candles | |
| 09 Exits | 1R → breakeven, bank 50%, rest to the next level (cap 3R) · too fast: ≥ 1.5R after 3 straight candles | |

---

## Honest notes

- Everything runs on closed candles and higher-timeframe data comes from closed candles, so signals do not repaint.
- Entries are simulated from candle data: when the dip and the break happen inside the same candle, the dip is assumed to come first.
- The weekly staircase needs several months of history. On short intraday histories the dashboard falls back to the daily staircase and says so.
- Built for education and research. Not financial advice. Trading involves risk of loss.

## Credits

The rules follow a three-step "story → zone → strat" session strategy a trader shared publicly on YouTube. This is an independent, open-source implementation and is not affiliated with him.

## Files

```
mathematical-indicator.pine      the indicator (Pine Script v6)
docs/STRATEGY.md                 every rule, in plain words and in numbers
```

MIT License.
