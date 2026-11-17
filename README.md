# Range Rider — cbBTC/USDC 1-minute trading bot

[![make.fun · btc-race-24h-20261117-0603](https://make.fun/badge/btc-race-24h-20261117-0603.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![rank](https://make.fun/badge/btc-race-24h-20261117-0603/kazamisao/btc-race-24h-20261117-0603/rank.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![reward](https://make.fun/badge/btc-race-24h-20261117-0603/kazamisao/btc-race-24h-20261117-0603/reward.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)

A hybrid bot that trades Coinbase Wrapped BTC against USDC minute by minute on replayed on-chain prices: it rides trend legs and buys dips inside a range, never a waterfall, and cuts every position short on a trailing stop.

Entry for **btc-race-24h-20261117-0603** — "Race a BTC trading bot live for 24 hours".
TypeScript (Node 22, type stripping), standard library only. Speaks `agent-fun/bars-replay/v1`
(NDJSON on stdin/stdout) via the kit's TypeScript SDK shim, vendored byte-identical
under `vendor/`.

## Strategy

Two ways in, one way out. Long only, at most one open position, full-size entries.

**Entry** (flat only), decided on a bar's close, filled at the next bar's open:

- **Momentum:** 20-bar return above **+150 bps**, or 15-bar return above **+90 bps**
  while the 30-bar return is above **+80 bps** (a sustained move, not a spike).
- **Dip:** close at least **80 bps** below its 60-bar mean (the mean of the last
  60 closes including the current bar), **and** the 240-bar return
  no worse than **-150 bps** — buy a dip inside a range, never a waterfall.

**Exit** (holding only), whole position: take-profit **+150 bps** over the entry close,
trailing stop **75 bps** below the highest close since entry, hard stop **-150 bps**
under the entry close. After any exit, no new entry for **60 bars**.

No entry is possible before the 241st bar: the 240-bar trend gate needs a full
240 bars of history first, so the first ~4 hours of the window are never traded.
A sell is re-sent on the next bar while its exit reason still holds if the replay
rejects it, and the cooldown is armed only once the position is actually gone.
Each order caps slippage at 150 bps vs the latest close. All bps comparisons are
cross-multiplied BigInt integer arithmetic — no floats, no clock, no network —
so two runs on the same bars produce identical replies.

## Why this shape

The dev window is a choppy range (BTC +48 bps over 24h, 1-minute return sd 6.4 bps):
fast crossovers whipsaw and lose (EMA-cross baseline: -$106 on 68 trades), while a
pure dip-buyer bleeds in a downtrend (-$79 on a second, out-of-sample 24h window
that fell -358 bps net with a +284 bps rally hour). The 240-bar trend gate is what
keeps the dip leg alive in a range and out of a waterfall; the momentum leg is
what catches the rally hour. A full-size round trip costs ~26 bps, so every leg
must catch a move far bigger than that — and does: the bot makes 2 round trips on
the dev window (4 filled orders) and wins both (+$14.37 and +$12.45).

## Results

- **Dev window** (official `grade-local`, two executions, action logs byte-identical):
  net PnL **+$26.82**, 4 trades, max drawdown 0.66%, Sharpe 0.043, 0 rejected.
  At 2x fees: +$25.18.
- **Out-of-sample 24h window** (different day, exact-integer simulator validated to
  the cent against the official grader on three baselines): -$5.76, 6 trades,
  max drawdown 1.13%.

## Run it

Grade locally with the kit (from the unpacked kit directory):

```sh
node cli/makefun.mjs grade-local --kit . --bot <path-to-this-repo> --driver process
```
