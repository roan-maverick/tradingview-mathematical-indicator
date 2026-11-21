# The Mathematical Indicator

**A free TradingView indicator that runs a complete session-trading strategy and sends every step to your AI agent as clean JSON.**

Two candles pick the direction. Floors and ceilings sit on candle bodies. Trades only happen in the first hour of a session, inside clean zones, after a range, a strong breakout candle and one dip back. The indicator draws all of it on your chart, and your AI agent reads every step and tells you TAKE, WAIT or SKIP.

It works with any agent. It's built to shine with long-horizon reasoning models like **Claude Fable 5.2**.

Open source · MIT license · Pine Script v6 · No repainting

---

## The strategy in 3 steps

| Step | Rule | On your chart |
|---|---|---|
| **① Story** | Yesterday's daily candle + the last 4H candle before the session. Both green = **BUY only**. Both red = **SELL only**. Mixed = the story decides: a daily level, then the weekly staircase, then the daily candle. | A banner at every session open: `NYSE · ▲ BUY ONLY` |
| **② Zone** | Floors and ceilings on candle **bodies**. A level only breaks on a close of its **own** timeframe. No buys under a daily ceiling, no sells over a daily floor: come back tomorrow. Only trade inside **clean zones**. If the pre-session candle already ran, wait for a pullback. | Gold daily levels, 4H/1H/15m levels with strength dots, indigo `CLEAN ZONE` boxes |
| **③ Strat** | First hour only. A range with 2+ opposite candles → one strong candle closes outside it (no bigger than half the range) → a dip back (**the rubber band**) → a stop order at the breakout candle fires. Stop beyond the breakout candle. | Range box, `① BREAKOUT`, `BUY STOP`, `② RUBBER BAND ↺`, `③ ENTRY ▲` |
| **④ Exit** | 1:1 first: stop to breakeven and bank half. The rest rides to the next level. Price runs too fast in a straight line: take it. | Risk/reward boxes, `1R → BREAKEVEN`, `TARGET · 1H ceiling`, `⚡ TOO FAST`, the result in R |

Every rule and number: [docs/SPECIFICATION.md](docs/SPECIFICATION.md)

---

## What your agent receives

One JSON packet per event, the moment the candle closes:

- **Events:** `SESSION_BIAS`, `STOP_ORDER_PLACED`, `STOP_ORDER_CANCELLED`, `ENTRY`, `BREAKEVEN`, `TOO_FAST_TAKE_PROFIT`, `EXIT` (and optionally `BAR_CLOSE` on every candle)
- **Story:** the bias, both candles, the weekly staircase and the reason in words
- **Zone:** daily ceiling and floor, positioning (0 to 100%), the come-back-tomorrow flag, clean zone, breakout or pullback mode, nearest levels above and below
- **Setup:** range top and bottom, candle count, liquidity grab, breakout candle, rubber band status
- **Order and trade:** stop order price and stop loss, entry, 1R, target and what it is, breakeven, open R, result in R
- **Candle:** the candle that just closed, broken down (body, wicks, size versus ATR)
- **Record:** the chart's own trades, win rate and net R

Full field list: [docs/PACKET_SCHEMA.md](docs/PACKET_SCHEMA.md) · Example: [examples/sample_packet.json](examples/sample_packet.json)

---

## How it works

```
 TradingView chart                   Your AI agent                    You
 ┌─────────────────────┐  webhook   ┌──────────────────────────┐    ┌───────────────┐
 │ Mathematical        │ ─────────► │ reads the JSON packet    │ ─► │ signal:       │
 │ Indicator runs the  │  JSON per  │ checks story, zone,      │    │ TAKE / WAIT   │
 │ story, zone, strat  │  event     │ strat, sizes the trade   │    │ SKIP / MANAGE │
 └─────────────────────┘            └──────────────────────────┘    │ order, stop,  │
                                                                    │ target, why   │
                                                                    └───────────────┘
```

1. The indicator runs the full strategy on your TradingView chart, on closed candles.
2. One TradingView alert sends every event to your agent's webhook URL.
3. Your agent (Claude Code with Fable 5.2, an n8n flow, your own bot) reads it with the prompt in [agent/AGENT_PROMPT.md](agent/AGENT_PROMPT.md) and returns a signal: decision, confidence, order, stop, 1R, target, position size, the story/zone/strat checklist, and why it could fail.

---

## Quick start

### Step 1. Add the indicator to TradingView (1 minute)

1. Open [mathematical-indicator.pine](mathematical-indicator.pine) and copy all of it.
2. In TradingView, open the **Pine Editor** at the bottom of the chart.
3. Press **Ctrl + A** (Cmd + A on Mac) and **delete the default code first**. A script can hold only one `indicator()` line, so pasting under the template gives the error *"Scripts must contain one declaration statement"*.
4. Paste, click **Save**, then **Add to chart**.
5. Use a **15 minute** chart to learn it, drop to **5 minutes** once you know the setup.

Sessions are set in **🕐 02 · Sessions**: Asia, London and the 9:30 NYSE open are on by default.

### Step 2. Create one alert (1 minute)

1. Open the alert dialog (clock icon, or `Alt + A`).
2. **Condition:** `The Mathematical Indicator · 2-Candle Bias` → `Any alert() function call`.
3. **Notifications:** tick **Webhook URL** and paste your agent's URL (Step 3).
4. Leave the message box as it is. The indicator writes the message itself.
5. Click **Create**.

One alert covers one chart (one symbol on one timeframe). Repeat it for every chart you want your agent to watch.

> Webhook alerts need a paid TradingView plan with two-factor authentication turned on. On the Essential and Plus plans, alerts expire after 2 months, so recreate them when they do.

### Step 3. Connect your agent

Pick one.

#### Option A: Claude Code with Fable 5.2 (recommended)

```bash
git clone https://github.com/roan-maverick/tradingview-mathematical-indicator
cd tradingview-mathematical-indicator

# 1. Start the receiver (Python 3, no installs)
python3 examples/webhook_receiver.py --port 8080 --token YOUR_SECRET

# 2. In a second terminal, give it a public HTTPS address
cloudflared tunnel --url http://localhost:8080
```

Your webhook URL for Step 2 is:

```
https://<your-tunnel-address>/webhook?token=YOUR_SECRET
```

Every packet is saved to `packets/latest.json`, `packets/latest/<SYMBOL>_<TF>.json` and `packets/history.jsonl`.

Now open Claude Code in the same folder:

```bash
claude
```

and ask:

```
Use agent/AGENT_PROMPT.md as your instructions. Read packets/latest.json
and give me the signal for this event.
```

More things to ask:

```
Read packets/history.jsonl for today and walk me through the session:
the bias, every setup, why it was taken or skipped, and the result.

Read every file in packets/latest/ and tell me which chart has a stop
order working right now and which one is blocked for today.

Read the last 30 EXIT packets and tell me which part of the strategy
(story, zone or strat) explains most of my losses.
```

#### Option B: No-code (n8n, Make, Zapier)

1. Create a **Webhook** trigger and copy its URL into the TradingView alert.
2. Add an **AI** step (Claude or any LLM). Use the block in [agent/AGENT_PROMPT.md](agent/AGENT_PROMPT.md) as the system prompt and the webhook body as the user message.
3. Send the reply to Telegram, Discord, Slack or email.

#### Option C: Your own bot

Point the webhook at your server. The body is plain JSON with the schema `mathematical-indicator/v1`. Check the `schema` field, then use the fields directly or pass the packet to your model with the agent prompt.

---

## What a signal looks like

Example output from the agent prompt ([examples/sample_signal.json](examples/sample_signal.json)):

```json
{
  "event": "STOP_ORDER_PLACED",
  "decision": "TAKE",
  "side": "LONG",
  "confidence": 66,
  "order": { "type": "BUY_STOP", "price": 21116.6 },
  "stop": 21098.6,
  "take_profit_1r": 21134.6,
  "target": 21140.1,
  "position_size": 1,
  "position_unit": "MNQ contracts",
  "checklist": {
    "story": "PASS: yesterday's daily and the last 4H candle are both green, bias BUY only",
    "zone": "PASS: inside a clean zone, 54% between the daily floor and ceiling, not blocked today",
    "strat": "PASS so far: 14-candle range with a liquidity grab, strong breakout candle. Rubber band still pending"
  },
  "why_this_could_fail": "The breakout could be a fake-out into the 1H ceiling at 21140.1. With the daily staircase pointing down, sellers may defend that level."
}
```

The example files are illustrative, not live market data or a real trade.

---

## What you see on the chart

| Element | What it shows |
|---|---|
| **Dashboard** | Story, zone and strat in one card: the two candles, BIAS, positioning meter, daily ceiling and floor, mode, the trigger steps, the state, the next step, the open trade and the on-chart record |
| **Session banner** | At every session open: session, bias, both candles, staircase, mode |
| **Floors & ceilings** | Gold daily levels, red/green 4H and 1H, magenta/cyan 15m and chart levels. `•••` = failed breaks that made the level stronger. Broken levels fade |
| **Clean zones** | Indigo boxes where price moved without building a level |
| **Setups** | Neon range box (with `LIQUIDITY GRAB ✓`), glowing breakout candle, dashed `BUY STOP` / `SELL STOP` line, `② RUBBER BAND ↺` |
| **Trades** | Red risk box, green reward box, `1R → BREAKEVEN` line, `TARGET · source` tag, `🛡 RISK FREE`, `⚡ TOO FAST · TAKE PROFIT`, the result in R |
| **Skips** | Grey `✕` tags that say why a breakout was not taken (come back tomorrow, candle too big, wait for a pullback, no clean zone, nipple wick) |
| **Bias aura** | Cyan buyers wave and purple sellers wave around price, glowing in the bias color. Visual only |

### The dashboard

| Section | Shows |
|---|---|
| Header | THE MATHEMATICAL INDICATOR, market, timeframe, session live or waiting |
| ① Story | Yesterday's candle, the last 4H candle, the staircase (✔ agrees, ◐ disagrees), **BIAS**, why |
| ② Zone | Positioning meter, daily ceiling and floor with distance, clean zone, pre-session 4H, mode, today: open or come back tomorrow |
| ③ Strat | Range, breakout candle, rubber band, entry, **state** (looking, buy stop working, long active…), the next step |
| Active trade | Stop, 1R, target with R and source, open P&L meter |
| On-chart record | Trades, win rate, net R, average R, profit factor |

Switch **Mode** to Compact for the essentials only.

---

## The math

Every trade is measured in R (1R = the distance from entry to stop). Expectancy per trade:

```
E = (win rate × average win) − (loss rate × average loss)
```

At 1:1 a win pays +1R and a loss costs −1R, so the break-even win rate is 50%:

| Win rate at 1:1 | Per 10 trades | Per trade |
|---|---|---|
| 50% | 0R | 0.0R |
| 60% | +2R | +0.2R |
| 70% | +4R | +0.4R |

Breakeven at 1R and banking half there cut the average loss. The story, the zone filters and the first-hour window push the win rate. No strategy wins every trade: check your own numbers in the on-chart record and in paper trading.

---

## Settings

| Setting | Default | What it changes |
|---|---|---|
| Sessions | Asia 09:00 Tokyo, London 08:00, NYSE 09:30 New York | Which first-hour windows are traded |
| Max trades per session | 2 | One trade at a time |
| When the 2 candles disagree | Tell the story | Or daily candle wins, or stand aside |
| Levels | Daily, 4H, 1H, 15m, chart | Which floors and ceilings are drawn and used as targets |
| Entries must be inside a clean zone | On | Off takes breakouts anywhere |
| No buys at a daily ceiling | On | The come-back-tomorrow rule |
| Wait for a pullback if the pre-session ran | On | Pullback depth 38.2% |
| Range | 3 to 20 candles, 2+ opposite | How the structure is found |
| Breakout candle | Body ≥ 50%, size ≤ 50% of the range | No dojis, no oversized candles |
| Rubber band wick | ≥ 0.1 ATR and ≥ 20% of its candle | Tiny "nipple" wicks don't count |
| Exit plan | 1R → breakeven, bank 50%, rest to the next level | Or 1:1 only |
| Too-fast exit | ≥ 1.5R after 3 straight candles | Warn only, or warn and close |
| Send JSON packet | Setups and trade updates | Or every bar close, or off |

---

## FAQ

**I get "Scripts must contain one declaration statement".**
The default template was still in the editor. Press Ctrl + A, delete everything, then paste the script again.

**Does it repaint?**
No. Everything runs on closed candles, and higher-timeframe values come from candles that already closed.

**Which markets and timeframes?**
Any symbol on TradingView: futures, forex, gold, indices, crypto. Built for 5 to 15 minute charts.

**Why are there days with no trades?**
That is part of the strategy. Mixed days with no story, days pressed against a daily level, breakouts with no dip first and candles that are too big are all skipped on purpose. The grey `✕` tags tell you why.

**Can I use it on the free TradingView plan?**
Yes. The indicator and dashboard work on every plan. Sending packets to an agent needs webhook alerts, which need a paid plan.

**Does it place trades?**
No. It produces data and signals. Your agent can be connected to a broker if you choose, but this repository never places orders.

---

## Repository

```
tradingview-mathematical-indicator/
├── mathematical-indicator.pine   the indicator (Pine Script v6)
├── agent/AGENT_PROMPT.md         system prompt for your AI agent
├── docs/SPECIFICATION.md         every calculation and rule
├── docs/PACKET_SCHEMA.md         every field in the JSON packet
├── examples/webhook_receiver.py  receive packets locally (Python 3, no installs)
├── examples/sample_packet.json   example packet
├── examples/sample_signal.json   example agent signal
└── LICENSE                       MIT
```

---

## Credits

The rules follow a three-step "story, zone, strat" session strategy a trader shared publicly on YouTube. This is an independent, open-source implementation and is not affiliated with him.

## Disclaimer

This project is for education and research. It is not financial advice. The indicator follows fixed rules; it does not predict the future. Signals from any AI agent can be wrong. Test on paper first, size small, and never risk money you can't afford to lose.

## License

MIT. Use it, fork it, build on it.
