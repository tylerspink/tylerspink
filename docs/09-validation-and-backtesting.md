# 09 — Validation & Backtesting

Most crypto trading bots fail not because the strategy was bad but because the backtest
lied. This document is about not being lied to.

## 9.1 The four ways a meme coin backtest lies

1. **Survivorship bias.** Historical data from public APIs mostly covers tokens that
   still exist. The 98% that died and were delisted are missing, so the sample is the
   winners. *Fix: our own tape, which records every token from first observation,
   including the ones that went to zero an hour later.*

2. **Look-ahead bias.** Using information that wasn't available at decision time — the
   most common form here is reading a token's *current* safety verdict when evaluating a
   trade from last week. *Fix: point-in-time, append-only feature snapshots; the replay
   engine physically cannot access rows with `observed_at > simulation_clock`.*

3. **Fill fantasy.** Assuming you filled at the mid price. On a thin pool with a 1–2 second
   latency and competing bots, you did not. *Fix: the fill simulator below.*

4. **Overfitting.** With hundreds of tunable parameters and a few hundred trades, any
   strategy can be made to look profitable in-sample. *Fix: strict train/validation/test
   splits, walk-forward analysis, and a parameter budget.*

## 9.2 The replay engine

The core validation tool, built in Phase 1–2, before any strategy work.

```
tape (object storage) ──▶ replay engine ──▶ strategy evaluator ──▶ intents
                              │                                       │
                     simulation clock                                 ▼
                     (no future access)                        fill simulator
                                                                      │
                                                                      ▼
                                                             simulated positions
                                                                  + P&L
```

Requirements:
- Deterministic: same tape + same config = byte-identical results. Non-negotiable; it's
  what makes A/B comparison of configs meaningful.
- Clock-gated: features resolve as of simulation time only.
- Fast: 30 days of tape in under 10 minutes, so iteration is cheap.
- Replays *all* observed tokens, not a curated list.

## 9.3 The fill simulator

Where most backtests quietly cheat. Every simulated fill applies, in order:

1. **Latency penalty.** Sample from our *measured* end-to-end latency distribution
   ([doc 03](03-system-architecture.md) §3.5) and advance the clock. Fill at the price
   that actually existed then — including any move caused by other bots reacting to the
   same event.
2. **Price impact.** Compute against actual pool reserves at that slot via the AMM
   invariant. Not a flat assumption.
3. **Fees.** Venue fee + LP fee + priority fee + Jito tip, using the *observed* fee
   levels from that slot, not today's.
4. **Failure rate.** Some transactions don't land. Sample from measured failure rates,
   including fee loss on failures.
5. **Adverse selection.** When the signal event itself moves the price, we're trading
   against it. Modeled from observed post-event price paths.

**Calibration loop:** once live, compare every real fill against what the simulator
would have predicted. A persistent gap means the simulator is optimistic, and every
backtest built on it is invalid until it's corrected. This calibration is checked
weekly and is a Phase 4 gate condition.

## 9.4 The validation pipeline

```
  idea
   │
   ├─▶ [1] IN-SAMPLE BACKTEST       days 1-60 of tape
   │       gate: positive expectancy net of full cost model, ≥200 trades
   │
   ├─▶ [2] OUT-OF-SAMPLE            days 61-90, parameters frozen
   │       gate: expectancy ≥ 50% of in-sample, same sign
   │
   ├─▶ [3] WALK-FORWARD             rolling 30d train / 7d test across full tape
   │       gate: positive in ≥60% of test windows
   │
   ├─▶ [4] PAPER TRADING            live data, simulated fills, ≥30 days
   │       gate: within 30% of backtest expectancy; no infra incidents 7 days
   │
   ├─▶ [5] MICRO LIVE               L1, $250, real money, ≥100 trades
   │       gate: expectancy > 0 @80% confidence; DD < 25%; fills within 150bps of sim
   │
   └─▶ [6] LADDER                   scale per doc 06 §6.5
```

A strategy failing any gate goes back to the idea stage. It does not get "one more
parameter tweak and a retry" — that's how overfitting enters through the back door.
**A parameter budget:** each strategy gets at most 8 tunable parameters, and every retune
against the same data costs a multiple-comparison penalty applied to the required
expectancy threshold.

## 9.5 Statistics — how many trades before we believe anything

Meme coin returns are extremely fat-tailed: most trades are small losses, a few are
large wins. Mean-based statistics converge slowly and mislead badly.

- **Minimum sample: 100 trades.** Below that, nothing is knowable.
- **Bootstrap confidence intervals**, not t-tests. The normality assumption is violated
  so severely that a t-test is actively misleading.
- **Report the median trade alongside the mean.** If the mean is positive and the median
  is deeply negative, the result rests on one or two outliers, and outliers do not repeat
  on schedule.
- **Report expectancy net of everything:** fees, slippage, failed transactions, *and*
  amortized infrastructure cost. A strategy earning $200/month is losing money at a
  $400/month burn.
- **Deflated Sharpe ratio** or an equivalent multiple-testing correction, because we will
  test many strategies and the best of 20 random strategies looks good by construction.

## 9.6 What gets measured continuously in production

| Metric | Why |
|---|---|
| Expectancy per trade, rolling 100 | The headline number |
| Hit rate and payoff ratio, separately | Tells you *how* expectancy changed |
| Median and mean trade return | Outlier dependence |
| Max drawdown and time-to-recovery | Against the 25% breaker |
| Realized vs. simulated fill | Simulator validity |
| Realized vs. quoted slippage | Execution quality and edge crowding |
| Transaction success rate and latency p50/p99 | Infra health |
| Per-strategy attribution | Allocation decisions |
| Rejected-signal counterfactual P&L | Whether risk limits are too tight |
| Cost ratio: fees+infra ÷ gross profit | The hurdle rate, tracked as a first-class metric |

## 9.7 The most likely way this goes wrong

Stated so we can watch for it specifically: a strategy validates cleanly on 90 days of
tape, passes paper trading, goes live, and produces nothing — because the edge was in
the *past regime* and the market changed, or because our own participation and the
crowding of similar bots removed it.

The defense isn't a better backtest. It's (a) the bankroll ladder, so discovering this
costs $250 rather than $25,000, and (b) continuous measurement of expectancy decay, so
a dying strategy gets de-allocated early rather than after it has given back a quarter's
gains.
