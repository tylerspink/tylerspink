# 01 — Objective & Mandate

## 1.1 The problem with the stated goal

The brief is: *trade autonomously with one goal — make as much money as possible.*

As an objective function for an autonomous system, that is unbuildable, for a specific
reason. An agent maximizing terminal wealth with no variance penalty and no ruin
constraint has a dominant strategy: concentrate the entire bankroll into the
highest-expected-value position available, every time. Repeated over N trades with any
per-trade probability of total loss `p`, survival probability is `(1-p)^N`. On meme
coins, where a single honeypot, freeze-authority trap, or liquidity pull is a -100%
outcome, `p` is not small. Over a few hundred trades, ruin is effectively certain.

Any system built to literally maximize money will therefore eventually take a bet that
ends the project. The constraint is not squeamishness — it is that an unbounded mandate
and a surviving bankroll are mutually exclusive.

## 1.2 The reformulated mandate

**Primary objective.** Maximize the geometric growth rate of the bankroll (log wealth),
subject to hard constraints:

- Maximum peak-to-trough drawdown: **25%** (auto-halt, human restart required)
- Maximum single-position risk: **2%** of bankroll (4% for a designated high-conviction tier)
- Zero tolerance for *catastrophic* events: key compromise, unbounded position size,
  runaway trade loop, unauthorized withdrawal destination

Log-wealth maximization is not a softer goal than "make as much money as possible" —
over any horizon longer than a few dozen trades it is the version that *actually*
produces the most money, because it is the only one that survives to keep compounding.

**Secondary objectives, in priority order:**

1. **Explainability.** Every trade traces to a named strategy, a timestamped signal, a
   risk decision, and a fill record. An unattributable P&L is a bug.
2. **Capital preservation during regime change.** The system should degrade to "hold
   SOL/USDC and do nothing" gracefully, not thrash.
3. **Throughput of validated strategies.** The durable asset is the research pipeline,
   not any one strategy. Meme coin edges decay in weeks.

## 1.3 Success criteria — defined before any money moves

These are the numbers we will be judged against. They are deliberately written now,
while it costs nothing to be honest.

| Horizon | Criterion | Value |
|---|---|---|
| Phase 4 gate (first real money) | Net expectancy per trade over ≥100 live trades | > 0 at 80% confidence (bootstrap) |
| Phase 4 gate | Max drawdown during those 100 trades | < 25% |
| Phase 5 | Monthly return net of **all** costs incl. infra | > 0 in 3 of any 4 consecutive months |
| Phase 5 | Realized slippage vs. quoted | within 150 bps of model |
| Always | Catastrophic events | zero |

**Kill criteria — the conditions under which we stop rather than scale:**

- 100 live trades completed with negative expectancy at 80% confidence → halt, full review
- Drawdown breaches 25% from peak → halt, human decision required to restart
- Two consecutive months where infra cost exceeds gross trading profit → the strategy
  does not clear its own hurdle rate; descope or stop
- Any key compromise or unauthorized transaction → full stop, rotate everything, post-mortem
- Realized fills persistently 300+ bps worse than simulation → the model is wrong;
  back to Phase 2

## 1.4 What "autonomous" will actually mean

Full autonomy is granted in stages, not at the start. The permission ladder:

| Level | Agent may | Requires |
|---|---|---|
| L0 | Observe, score, alert | — |
| L1 | Paper-trade | — |
| L2 | Execute pre-approved strategies at micro size | Phase 4 gate passed |
| L3 | Size up within the risk engine's caps | 100 profitable trades at L2 |
| L4 | Allocate capital *between* proven strategies | 3 strategies live at L3 |
| L5 | Author and deploy a *new* strategy | Backtest + paper gate + **your explicit approval** |

Note the asymmetry that runs through the whole design: **agents may always reduce risk
autonomously (halt, exit, de-size) and may never increase it without passing a gate.**
Stopping is a safe action; starting is not.

## 1.5 What I expect the outcome distribution to look like

Stated plainly, because a plan that hides this is worse than useless:

- **Most likely (~50%):** the system works mechanically, trades cleanly, and produces a
  return somewhere between -30% and +50% annualized on the deployed bankroll before
  infra costs — i.e. after infra, roughly a wash. The real product is the pipeline.
- **Good case (~25%):** one or two strategies (most likely smart-money mirroring and
  narrative rotation) show a durable edge worth 3–10% monthly at small size, decaying
  as they're crowded out, replaced by the research loop.
- **Bad case (~25%):** no strategy clears the fee hurdle, the Phase 4 gate fails, and
  the correct move is to stop. This is a *successful* outcome of the process, reached
  for roughly $500–1,500 in infra and test capital rather than for the whole bankroll.

The leverage is entirely in how cheaply and quickly we can reach that fork.
