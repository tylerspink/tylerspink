# 05 — Strategy Portfolio

Five candidates, ranked by expected risk-adjusted return *given our latency profile*.
Each has a hypothesis, a falsification test, and an explicit reason it might not work.

**The filter applied to all of them:** viable at ~2s decision-to-landed latency, target
move ≥ 15% to clear the fee hurdle from [doc 02](02-market-reality.md), and mechanically
expressible as a deterministic rule so it can be replayed against the tape.

---

## S1 — Smart-Money Mirror *(build first)*

**Hypothesis.** A small set of wallets shows persistent, statistically distinguishable
skill. Their entries carry information. Mirroring a *filtered* subset with independent
risk management beats mirroring any one of them.

**Why it ranks first.** The latency requirement is 1–3 seconds, which we can meet. The
edge lives in wallet *selection* — a research problem, not a speed problem — and that's
exactly where our data pipeline and LLM review loop are advantaged over the thousands of
naive copy-trade bots that mirror whatever a Telegram channel tells them to.

**Signal.**
- Maintain a ranked wallet universe scored on ≥90 days of realized PnL, trade count,
  hit rate, median hold time, max drawdown, and **an explicit insider filter** (wallets
  whose profits come from being early to tokens deployed by wallets they funded are
  detected and excluded — their edge is not reproducible by us)
- Entry trigger: ≥ N qualified wallets (start N=2) buy the same mint within a
  T-second window (start T=180), token passes the safety gate
- Weight by the buying wallets' scores; require combined score above threshold

**Entry.** Market buy via Jupiter, tight slippage (≤ 8%), Jito bundle with `dontfront`.
**Exit.** Mirror-sell when ≥50% of the mirrored wallets exit; OR laddered TP at +50%
(sell 40%) / +150% (sell 30%) / trail the remainder at 30%; OR -35% stop; OR 6h time stop.
**Size.** 1.5–2% of bankroll, capped at 0.5% of pool liquidity.

**Falsification test.** Replay 60 days of tape. If the top-decile wallet cohort's
forward 1h return after entry is not materially above the all-token baseline *after*
modeled costs, the hypothesis is dead — no amount of tuning saves it.

**Why it might fail.** (a) Smart money's edge is partly *their* speed, and we enter 1–3s
later into a price that already moved. (b) Wallet skill may not persist — measure
period-over-period rank correlation before trusting it. (c) They get copied by everyone,
so their fills degrade and so do ours. Mitigation for (a) is the single most important
open research question: measure the decay curve of forward returns vs. entry lag.

---

## S2 — Graduation Momentum

**Hypothesis.** Bonding-curve graduation is a discrete, observable, scheduled-ish event
that filters out ~98% of tokens. Tokens that complete the curve have demonstrated real
buy pressure, and the migration moment produces a structural liquidity and attention
shift that plays out over minutes, not milliseconds.

**Why it ranks second.** The event is precisely detectable on-chain and the useful
window is minutes wide. We're not racing to be first into the new pool; we're trading
the continuation after the initial snipe chaos resolves.

**Signal.**
- Detect curve completion / migration to PumpSwap
- Wait out the snipe window (start with 30–90s; tune on tape)
- Require: post-migration price above migration price, unique buyer count rising,
  top-10 holder concentration below threshold, no bundle-cluster distribution detected,
  safety gate passed

**Entry.** Market buy after the settle window. **Exit.** TP ladder; -30% stop; 4h time stop.
**Size.** 1.5% of bankroll.

**Falsification test.** Distribution of forward 15m/1h/4h returns for graduated tokens,
conditioned on the filter set, net of costs. Needs a positive median, not just a positive
mean — a mean carried by one 50x in the sample is not a strategy.

**Why it might fail.** Graduation is the most-watched event in the ecosystem. Post-
migration price action may be entirely front-run. Also: graduation counts scale with
market activity, and activity is down 38% MoM — signal frequency may be too low to reach
statistical significance in reasonable time.

---

## S3 — Narrative / Meta Rotation *(the LLM's actual edge)*

**Hypothesis.** Meme coin flow clusters into short-lived narratives. Identifying an
emerging meta 30–120 minutes before it's widely priced, then buying the *best-constructed
token* within it, is a judgment task that an LLM with broad context does better than a
rule, and it operates on a timescale where our latency is irrelevant.

**Why it matters strategically.** This is the only strategy where we have an advantage
that isn't available to a well-funded competitor with faster machines. It's also the
hardest to validate.

**Signal.**
- Research agent runs every 15 minutes over news, X, Telegram, and on-chain token-naming
  clusters (a sudden burst of tokens sharing a theme *is itself* the signal)
- Agent emits a structured `NarrativeCandidate`: theme, evidence, confidence, expected
  duration, candidate mints
- Deterministic gate: each candidate mint must independently pass safety + liquidity +
  holder distribution, and the theme must show independent on-chain confirmation
  (multiple unrelated deployers, rising unique buyers across the cluster)

**Entry.** Basket of 2–4 tokens within the theme, half size each — narrative bets are
inherently correlated, so the basket *is* the position.
**Exit.** Theme-level: exit all when velocity decays below entry level, or at 24h.
Plus per-token TP ladder and -35% stop.
**Size.** Basket total ≤ 3% of bankroll. Counts as a single position for correlation caps.

**Falsification test.** Hardest to backtest honestly, because the agent's judgment isn't
replayable from tape alone. Approach: have the agent produce narrative calls *live* in
paper mode for 30 days, logged with timestamps, then evaluate forward returns. Do not
let it see post-hoc data. Accept the slower validation.

**Why it might fail.** LLMs are good at explaining narratives *after* they exist and poor
at timing them. High risk of confident, well-reasoned, wrong calls. Mitigation: the agent
can only nominate; deterministic on-chain confirmation is required before any entry, and
the basket cap bounds the damage.

---

## S4 — Survivor Momentum

**Hypothesis.** Tokens that survive 24–72 hours with sustained organic volume and
growing holder counts are a different, much better-behaved population than launch-day
tokens. Standard momentum/breakout logic works better here than anywhere else in meme
coins, and competition is lighter because it's unglamorous.

**Signal.** Age > 24h; liquidity > $50k and rising; holder count up > 20% over 24h;
volume/liquidity ratio in a healthy band (too high = wash trading); price breaking a
24h range on above-average volume; safety gate passed and re-verified.

**Entry.** Breakout confirmation, not anticipation. **Exit.** Trailing stop 25%;
-20% hard stop; 48h time stop.
**Size.** 2% of bankroll — the deepest liquidity in the portfolio supports the largest size.

**Falsification test.** Straightforward and cheap: classical momentum backtest on tape,
strict cost model. Run this one first because it validates fastest.

**Why it might fail.** Survivorship-bias trap — we're selecting on past success. The
backtest must include every token that met the criteria, including those that then died.
In a declining market, "survivors" may simply be slower bleeds.

---

## S5 — Liquidity-Event Reaction *(risk overlay, not a P&L strategy)*

Not intended to make money. It's a fast reflex that exits on adverse structural events:
LP unlock/removal, mint authority reappearing, top-holder distribution beginning, a
tracked insider wallet exiting, liquidity dropping below floor.

**Why it's listed as a strategy:** it needs the lowest latency in the whole system
(sub-second) and its own priority lane, because an exit that fires 5 seconds late on a
rug is worth nothing. Its P&L contribution shows up as reduced left-tail on every other
strategy. Build it in Phase 3 alongside the position manager.

---

## Explicitly rejected

| Strategy | Why not |
|---|---|
| **Launch sniping** (first block) | We lose to co-located bots with stake-weighted QoS. Losing means buying the top from whoever won. Non-viable at any budget we'd spend. |
| **MEV / sandwiching** | Requires validator-adjacent infrastructure; also predatory, and I'm not building it. |
| **Cross-DEX arbitrage** | Sub-100ms game, saturated by dedicated firms. |
| **Dev-affiliated / insider launches** | This is where much of the "smart money" profit actually comes from. It's market manipulation, it's the thing regulators prosecute, and it's out of scope. |
| **Leverage on meme perps** | Adds liquidation risk on top of an already-fat left tail. Revisit never. |

---

## Portfolio construction

Strategies are allocated a capital budget, not unlimited access to the bankroll.

| Strategy | Initial allocation | Max concurrent positions |
|---|---|---|
| S1 Smart-Money Mirror | 40% | 4 |
| S2 Graduation Momentum | 20% | 3 |
| S3 Narrative Rotation | 15% | 1 basket |
| S4 Survivor Momentum | 25% | 3 |

Total deployed capital is capped at **20% of bankroll** regardless of allocation
percentages; the remainder sits in SOL/USDC. Allocations reprice monthly based on
realized risk-adjusted contribution, and a strategy that breaches its own drawdown
budget is auto-suspended pending review ([doc 06](06-risk-engine.md)).

**Correlation warning.** These strategies are far more correlated than they look — in a
market-wide meme dump, all four lose simultaneously. The 20% deployment cap, not
diversification across strategies, is the actual protection.

## Build order

1. **S4** — fastest to validate, cheapest to backtest, proves the whole pipeline
2. **S1** — highest expected value, requires wallet reputation infrastructure
3. **S2** — needs precise migration detection
4. **S5** — must exist before size increases
5. **S3** — validate last; slowest and least certain
