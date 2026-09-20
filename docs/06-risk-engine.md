# 06 — Risk Engine

The risk engine is the component most worth over-building. Strategies will come and go;
the thing that keeps a bad week from being the last week is this.

**Core property: the risk engine can only ever reduce or reject.** There is no code path
in it that increases a position's size, raises a limit, or resumes a halt. Those
transitions require either a time-based reset or a human.

## 6.1 Position sizing

**Method: fractional Kelly, floored and capped.**

Full Kelly on meme coins is suicidal — the edge estimate has enormous error bars, and
Kelly is extremely sensitive to overestimated edge. We use **quarter-Kelly**, then apply
hard caps that bind in practice almost always.

```
kelly_f      = (p * b - (1 - p)) / b        # p = hit rate, b = win/loss payoff ratio
size_frac    = clamp(0.25 * kelly_f, 0, 0.02)
size_usd     = bankroll * size_frac * strategy_confidence * regime_multiplier
size_usd     = min(size_usd,
                   0.005 * pool_liquidity_usd,   # market impact cap
                   position_cap_usd)             # absolute cap, see ladder
```

Worked example: a strategy measuring p=0.25 with b=4.0 gives kelly_f = 0.0625;
quarter-Kelly = 1.6% of bankroll. That lands inside the 2% cap, so the cap doesn't
bind — which is what we want. If a strategy's measured stats imply more than 2%, the
cap binds and we take the smaller number, every time.

`regime_multiplier` scales all sizing down in unfavorable conditions (market-wide
drawdown, low volume, elevated priority fees, high recent slippage). Range 0.3–1.0.

**Kelly inputs are estimated from a rolling 100-trade window with a Bayesian prior
pulled toward zero edge.** A strategy with 12 trades does not get to claim a 60% hit
rate. This prevents the most common blowup: sizing up after a lucky streak.

## 6.2 Hard limits

| Limit | Value | On breach |
|---|---|---|
| Max single position | 2% of bankroll (4% high-conviction tier, requires 2 independent strategy confirmations) | Reduce to cap |
| Max position vs. pool liquidity | 0.5% | Reduce to cap |
| Max total deployed | 20% of bankroll | Reject new entries |
| Max concurrent positions | 8 | Reject new entries |
| Max per-strategy allocation | per [doc 05](05-strategy-portfolio.md) | Reject new entries for that strategy |
| Max correlated exposure (same narrative/deployer/theme) | 5% of bankroll | Reject |
| Min pool liquidity to enter | $25,000 | Reject |
| Max slippage tolerance | 8% (3% for exits on liquid pools) | Reject / abort tx |
| Max priority fee + tip | 1.5% of position value | Reject |
| Min expected move to justify entry | 15% | Reject |

## 6.3 Circuit breakers

Tiered, and each one is automatic. The distinguishing feature of a system that survives
is that the halt fires without anyone having to be awake.

| Trigger | Action | Reset |
|---|---|---|
| Daily realized P&L ≤ -8% of bankroll | Halt all new entries | 24h, automatic |
| Weekly realized P&L ≤ -15% | Halt all new entries | Manual, after written review |
| Drawdown from peak ≥ 25% | **Full stop.** Exit all, halt. | Manual only |
| 8 consecutive losing trades (any strategy) | Suspend that strategy | Manual review |
| Strategy drawdown ≥ 40% of its allocation | Suspend that strategy | Manual review |
| Realized slippage > 300 bps over model, 20-trade rolling | Halt entries | Auto once slippage normalizes |
| Transaction failure rate > 30% over 20 attempts | Halt entries | Auto |
| Slot lag > 20 | Halt entries (exits still run) | Auto |
| Safety-gate false negative detected (we bought a rug) | Halt, mandatory post-mortem | Manual |
| Any unrecognized transaction from the hot wallet | **Full stop + key rotation** | Manual |

**Exits are never halted.** Every circuit breaker stops *entries*. A system that can't
exit during a halt is worse than no system.

## 6.4 Exit policy

Exits get more design attention than entries, because entry mistakes cost fees and exit
mistakes cost positions.

**Default ladder** (per-strategy overrides allowed):
- +50%: sell 40% — this alone takes the trade to roughly break-even-or-better after costs
- +150%: sell 30%
- Remainder: trailing stop at 30% below the high-water mark
- Hard stop: -30% to -35% depending on strategy
- Time stop: strategy-specific (4h–48h), because a meme coin that hasn't moved is a
  meme coin that is quietly bleeding to fees and attention decay
- **Safety-degradation exit: immediate, full, priority lane** (S5 in doc 05)

**On stop-losses in illiquid markets.** A -30% stop does not guarantee a -30% outcome.
In a rug, liquidity vanishes and the realized exit may be -90% or unfillable. Therefore
the *position size*, not the stop, is the real risk control. We size assuming the stop
may fail entirely. This is why the 2% cap exists and why it is not negotiable.

## 6.5 The bankroll ladder

Capital scales only by passing gates. Never by feeling good about a week.

| Level | Bankroll | Max position | Promote when | Demote when |
|---|---|---|---|---|
| L0 Paper | $0 | — | 200 paper trades, positive modeled expectancy | — |
| L1 Micro | $250 | $10 | 100 live trades, expectancy > 0 @80% conf, DD < 25% | any breach |
| L2 | $1,000 | $40 | 100 trades at L1 meeting criteria | DD > 20% → back to L1 |
| L3 | $5,000 | $150 | 100 trades at L2 | DD > 20% → back to L2 |
| L4 | $15,000 | $400 | 100 trades at L3 + 2 months positive net of infra | DD > 20% → back to L3 |
| L5 | $40,000+ | $800 | Review with you, not automatic | — |

Demotion is automatic and immediate. Promotion requires a gate *and* a written review.
At L4 and above, position sizes finally reach the $200–800 range where fixed transaction
costs stop being a meaningful drag ([doc 02](02-market-reality.md) §2.3) — which is worth
naming plainly: **at L1 and L2 we are paying to learn, not expecting to earn.** The
purpose of those levels is to validate the machinery with real fills, real slippage and
real adverse selection, at a size where being wrong costs a few hundred dollars.

## 6.6 What the risk engine logs

Every decision, approved or rejected, with a structured reason code. The rejection log
is as valuable as the trade log: if the engine is rejecting 90% of signals, either the
strategies are bad or the limits are miscalibrated, and only the counterfactual tells us
which. Reviewed weekly.

## 6.7 Drawdown expectations — so a normal bad run isn't mistaken for failure

With 2% position sizing, a 25% hit rate, and a 4:1 payoff ratio, losing streaks of 10–15
trades are *routine*, not anomalous. That's a 20–30% drawdown from ordinary variance in a
system working exactly as designed.

Two implications:
1. The 25% max drawdown breaker will likely fire at some point even if the edge is real.
   That's intended — it forces a human look rather than a slow bleed.
2. **Do not intervene during a drawdown that is within statistical expectation.** The
   most common way a working system gets killed is the operator turning it off during
   normal variance, or worse, sizing up to "make it back." Both are forbidden by the
   ladder above, which is precisely why the ladder is mechanical.
