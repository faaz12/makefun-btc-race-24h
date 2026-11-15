# btc-race-24h — dip-scale mean-reversion bot

Entry for the make.fun competition **btc-race-24h-20261115-0828**
("Race a BTC trading bot live for 24 hours", $250 prize pool).
Trades Coinbase Wrapped BTC (cbBTC) against USDC on 1-minute bars, long
only, starting from $1,000 paper USDC, over the `agent-fun/bars-replay/v1`
protocol. Python 3.13, standard library only.

## Strategy

Multi-hour dislocation mean reversion with scale-in and a trailing stop:

- **Enter** when the close is at least **200 bps below** where it was
  **240 minutes ago** (the 4-hour return on this pool mean-reverts —
  its autocorrelation is about −0.4), confirmed by a 14-bar RSI under 45.
- **Scale in** in three equal tranches of one third of starting equity as
  the drop deepens past **200 / 300 / 400 bps** vs 240 minutes ago, so
  the average cost sits near the middle of the decline instead of at its
  top.
- **Exit** at **+400 bps** over the position's average cost, or on a
  **trailing stop 100 bps** below the highest close since entry (the stop
  also cuts a position that keeps falling, freeing cash to re-enter
  lower), or after 1,440 bars. 30-bar cooldown after an exit.
- **Costs matter:** a full-size round trip pays ~40 bps on this pool
  (4 bps fee + 2 bps slippage + ~15 bps of price impact each side at the
  dev window's depth), so entries only fire on 200+ bps dislocations.

The bot is a pure function of the bars seen so far: no wall clock, no
network, no randomness — two runs on the same window produce identical
replies.

## Files

- `main.py` — the bot (entrypoint, run as `python3 -P -s /bot/main.py`)
- `vendor/makefun_bars.py` — byte-for-byte copy of the kit's Python SDK
  shim (excluded from authorship counts per the rules)

## Local dev-window grade

Graded on the kit's published dev window (1,440 one-minute bars) with an
exact re-implementation of the harness's integer fill model
(`analysis/replay.py`, validated against the official grader's
buy-and-hold figure of −$44.01). The official `grade-local` command
could not run this Python bot here (its process driver is Node-only and
this machine has no Docker), so the replay above is the local grade.

| Metric | Value |
|---|---|
| Net PnL | **+$23.37 USDC** |
| Return | +2.34% |
| Max drawdown | 2.12% |
| Trades | 16 |
| Fees | $2.66 |
| Net PnL at 2x fees | +$19.16 |
| Determinism | two runs, byte-identical action logs |

For reference, every kit baseline loses on the same window
(buy-and-hold −$44.01, momentum −$40.92, ema-cross −$134.95,
rsi-bollinger −$57.37, adx-momentum −$86.60) — the dev day fell ~4%,
so the gate is passed by trading the bounces, not by holding.

## Honest caveats

- The profit concentrates in one episode: the day's deep afternoon
  decline and V-shaped recovery (bars 720–1080, +$23.61 of the +$23.37).
  The first half of the window is roughly flat (−$4.85). The edge is
  real (4-hour mean reversion) but a steadily-trending race day would
  pay much less, and a single 24-hour window is a small sample.
- Parameters sit in the middle of a broad positive plateau
  (TP 350–500 bps × trailing stop 80–150 bps all clear +$10 on dev),
  not at a knife-edge optimum.
