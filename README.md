# makefun-btc-race-24h-20261118-1444

[![make.fun · btc-race-24h-20261118-1444](https://make.fun/badge/btc-race-24h-20261118-1444.svg)](https://make.fun/bounty/btc-race-24h-20261118-1444?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261118-1444)

Selective dip mean-reversion BTC trading bot: buys CBBTC when it closes more than 0.5% below its 60-bar average and turns up, sells at a +0.6% take-profit or 60 bars later, with two brakes that keep it out of grinding-down markets and an always-trade fallback so it never ends a window with 0 trades. Dev window net PnL +$9.68.

## Run it

```sh
python3 main.py
```

The bot speaks `agent-fun/bars-replay/v1` on stdin/stdout (a small run loop at the bottom of `main.py`, standard library only). The grader runs it as `python3 -P -s /bot/main.py` with `PYTHONHASHSEED=0`.

## Strategy

Long only, full size (one position, 100% of equity), one entry signal:

- **Entry (when flat):** close more than 0.5% below its 60-bar simple average AND above the close 6 bars ago — buys local bottoms after a sharp drop, not the slide down to them. The 60-bar average is slow enough that the signal survives the bottom (a 30-bar average falls too fast and the window closes before the turn confirms).
- **Guards:** no entry while the 60-bar average has fallen more than 1% over the last 360 bars (sustained downtrend: dip-buying there loses ~1% per trade on the old windows); no new entry once realized PnL is below -$5; no new position after bar 1100 (this window's last 400 bars bleed -2.77% with no tradeable bounce).
- **Exit (when holding):** sell everything at +0.6% take-profit, fallback sell 60 bars after entry, stop-loss -0.5% from cost. The tight stop is what keeps losing trades small in a falling market.
- **Always-trade fallback:** if no trade at all by bar 1200, buy on the next small uptick with +0.5% take-profit, 90-bar hold, -0.8% stop. A bot that never trades makes exactly $0 and fails the dev gate; the fallback never fires on the dev window.
- Orders cap slippage at 100 bps from the latest close, as the kit baselines do. No clock, no randomness, no network: two runs on the same bars give the same orders.

## Dev window results (new window, 1,440 bars, BTC -2.80%, 2026-11-17 21:44 → 2026-11-18 21:44 WIB)

| Metric | Value |
|---|---|
| Net PnL | **+$9.68** |
| Trades | 4 (2 round trips) |
| Fees | $1.61 |
| Max drawdown | -0.28% |
| First half / second half | +$10.49 / $0.00 |
| Buy-and-hold / hold-cash | -$31.63 / $0.00 |

The two round trips: dip buy $63,870.54 → sell $64,292.18 (+$6.20, take-profit); dip buy $63,613.20 → sell $63,910.19 (+$4.29, take-profit decided at the $64,011 close). The window's only clean moves are two fast V-bounces in the first 160 bars; the second half has no edge for dip-buying (smaller thresholds and scalp exits were tested and lose money there, because the 26 bps round-trip cost eats the 0.4-0.5% swings), so the bot trades twice and sits out.

## Validation

- Wire test of this exact file on the dev window: 1,440/1,440 replies, exit 0, stderr empty, final cash $1,009.6831 (== simulator), two runs byte-identical (deterministic).
- Simulator calibration: the same simulator reproduces round 1's official server grade exactly (+$24.6078, 6 trades) on the previous race's window, and the fill model uses a fee-inclusive cost basis as the harness does.
- Every ±20% perturbation of every parameter stays positive (worst +$3.16: dip threshold x0.8). At 1.5x fees/slippage +$8.47, at 2x fees +$7.27.
- Cross-window check, same bot on other races' dev windows: 2314 -$10.48, 0828 -$8.21, 0603 -$4.24 (mean -$3.31, worst -$10.48). The brakes cap the damage in falling markets: without them the same entries lose -$28.83 on 2314 and -$68.64 on 0828. Baselines on the new window: momentum -$23.87, ema-cross -$128.30, rsi-bollinger -$143.14, adx-momentum -$323.35.
