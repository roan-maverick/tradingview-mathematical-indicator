# Packet schema: `mathematical-indicator/v1`

The indicator sends one JSON packet per event, on candle close. With **Send JSON packet = Every bar close** it also sends a `BAR_CLOSE` packet on every closed candle that has no other event. Missing values are sent as `null`. Prices are rounded to the symbol's tick size.

Full example: [`examples/sample_packet.json`](../examples/sample_packet.json)

## Events

| `event` | Sent when | What an agent does with it |
|---|---|---|
| `SESSION_BIAS` | First candle of every enabled session window | State the bias and the plan for the session |
| `STOP_ORDER_PLACED` | A breakout candle closed outside a valid range | Check story, zone and strat; TAKE or SKIP the stop order |
| `STOP_ORDER_CANCELLED` | Close back inside the range, break with no dip first, or the order expired | Stand down; the reason is in `trade.note` |
| `ENTRY` | The stop order filled after the rubber band dip | Manage: stop, 1R, target, size |
| `BREAKEVEN` | Price reached 1R, or a level before 1R | Move the stop to entry, bank the partial |
| `TOO_FAST_TAKE_PROFIT` | Trade up ≥ 1.5R after 3 straight candles in its direction | Take profit (an `EXIT` follows when auto-close is on) |
| `EXIT` | Stop, breakeven, target, too fast or time exit | Record the result: `trade.result_r` |
| `BAR_CLOSE` | Every other closed candle (only in "Every bar close" mode) | Context only |

## Top level

| Field | Type | Description |
|---|---|---|
| `schema` | string | Always `"mathematical-indicator/v1"` |
| `event` | enum | See the table above |
| `tag` | string | The "Packet tag" input, for routing several charts |
| `symbol` | string | Exchange prefix and ticker, e.g. `CME_MINI:NQ1!` |
| `timeframe` | string | Chart timeframe code: `5`, `15`, … |
| `time` | string | Candle close time, ISO 8601 UTC |
| `state_code` | string | One-line summary: `MATH\|TICKER\|TF\|SESSION\|BIAS\|MODE\|STATUS` |
| `candle` | object | The candle that just closed |
| `session` | object | Session window |
| `story` | object | ① The two candles and the bias |
| `zone` | object | ② Levels, positioning, clean zones, mode |
| `setup` | object | ③ The range and the breakout candle |
| `order` | object | The stop order |
| `trade` | object | The trade |
| `record` | object | The chart's own record |

## `candle`

| Field | Type | Description |
|---|---|---|
| `open`, `high`, `low`, `close` | number | Prices |
| `direction` | enum | `bull`, `bear`, `flat` |
| `body_pct` | number | Body as % of the candle's range |
| `upper_wick_pct`, `lower_wick_pct` | number | Wicks as % of the range |
| `range_atr` | number | Candle range divided by ATR(14) |

## `session`

| Field | Type | Description |
|---|---|---|
| `name` | enum | `Asia`, `London`, `New York`, `NYSE`, `Power Hour` (the last session that started) |
| `live` | bool | Inside a first-hour window right now |
| `trades` | int | Trades taken in this session |
| `max_trades` | int | Limit per session |

## `story`

| Field | Type | Description |
|---|---|---|
| `bias` | enum | `BUY`, `SELL`, `NONE`. Frozen at the session's first candle |
| `daily_candle` | enum | Yesterday's daily candle: `UP`, `DOWN`, `FLAT` |
| `h4_candle` | enum | The last closed 4H candle before the session: `UP`, `DOWN`, `FLAT` |
| `staircase` | enum | `UP` (higher highs and lows), `DOWN`, `BOX`, `UNKNOWN` |
| `staircase_tf` | enum | `weekly`, or `daily` when the chart has too little weekly history |
| `reason` | string | Why the bias is what it is, e.g. `2 candles agree`, `Mixed → at a daily ceiling → SELL` |

## `zone`

| Field | Type | Description |
|---|---|---|
| `mode` | enum | `BREAKOUT`, or `PULLBACK` when the pre-session 4H candle already ran in the bias direction |
| `daily_ceiling`, `daily_floor` | number or null | Nearest unbroken daily levels above and below price |
| `position_pct` | number or null | Where price sits between them: 0 = floor, 100 = ceiling |
| `at_daily_ceiling`, `at_daily_floor` | bool | Price is pressed against a daily level (within 0.25 daily ATR, or through it without a daily close) |
| `blocked_today` | bool | The bias direction is done for today: come back tomorrow |
| `in_clean_zone` | bool | Price is inside an active clean zone now |
| `next_ceiling`, `next_floor` | number or null | Nearest unbroken level above / below price, any timeframe |
| `next_ceiling_tf`, `next_floor_tf` | string or null | Their timeframe: `D`, `4H`, `1H`, `15m`, or the chart's |

## `setup`

Filled while a setup is live and on the packets that close it. Otherwise `side` is `NONE` and the numbers are `null`.

| Field | Type | Description |
|---|---|---|
| `side` | enum | `LONG`, `SHORT`, `NONE` |
| `range_top`, `range_bottom` | number or null | The range, measured on candle bodies |
| `range_candles` | int or null | Candles in the range |
| `liquidity_grab` | bool | A candle swept the range's earlier lows (highs for sells) and closed back inside |
| `in_clean_zone` | bool | The breakout happened inside a clean zone |
| `breakout_high`, `breakout_low` | number or null | The breakout candle |
| `rubber_band` | bool | The dip after the breakout has happened |

## `order`

| Field | Type | Description |
|---|---|---|
| `active` | bool | A stop order is working right now |
| `type` | string | Always `STOP` |
| `action` | enum | `BUY`, `SELL`, `NONE` |
| `price` | number or null | The stop order price (breakout candle high for buys, low for sells) |
| `stop_loss` | number or null | Beyond the breakout candle, plus the tick buffer |
| `cancel_after_candles` | int | Order lifetime |

## `trade`

Filled while a trade is open and on its `EXIT` packet.

| Field | Type | Description |
|---|---|---|
| `status` | enum | `FLAT`, `ORDER_WORKING`, `OPEN`, `CANCELLED`, `CLOSED` |
| `entry` | number or null | Fill price |
| `stop` | number or null | Current stop (equals `entry` after breakeven) |
| `risk_per_unit` | number or null | \|entry − stop\| at entry |
| `take_profit_1r` | number or null | 1R price |
| `target` | number or null | Final target |
| `target_source` | string or null | What the target is: `1H ceiling`, `D floor`, `3R cap · open space`, `1:1` |
| `breakeven` | bool | The stop sits at entry |
| `close_pct_at_1r` | number | % banked at 1R (0 in `1:1 only` mode) |
| `open_r` | number or null | Open result in R |
| `exit_price` | number or null | `EXIT` only |
| `result_r` | number or null | `EXIT` only: result of the whole trade in R, partial included |
| `note` | string | Exit or cancel reason |

## `record`

The indicator's own record on this chart's history. No fees, commissions or slippage.

| Field | Type | Description |
|---|---|---|
| `trades`, `wins`, `breakeven`, `losses` | int | Counts (breakeven = result within ±0.05R) |
| `win_rate` | number or null | wins / trades × 100 |
| `net_r` | number | Sum of all results in R |
| `note` | string | Always `this chart's history, no fees` |

<!-- Maintained by repo owner -->
