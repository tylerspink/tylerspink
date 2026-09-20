# 11 — Roadmap

Six phases. Each ends in a **gate that can fail**, and failing a gate means stopping or
reworking, never "proceeding anyway." Calendar estimates assume part-time work with me
doing the building; they are estimates, and the gates are what actually govern.

---

## Phase 0 — Decisions & Setup *(week 1, no code)*

**Deliverables**
- Answers to the nine questions in [doc 14](14-open-decisions.md)
- Private implementation repository created
- Accounts: RPC/Geyser provider, data APIs, VPS, secret manager, monitoring
- Wallets: cold hardware wallet, treasury multisig (Squads), hot keypair generated *on
  an offline machine*
- Capital plan confirmed, written down
- Threat model reviewed and signed off
- CPA contacted about crypto trading tax treatment

**Gate:** all nine decisions made; funds in the treasury multisig; **$0 on the hot wallet**.

---

## Phase 1 — Data Spine *(weeks 2–4)*

The foundation. Nothing downstream is trustworthy without this, so it gets built first
and properly.

**Deliverables**
- Geyser/LaserStream ingest with reconnect, replay, gap detection
- Venue decoders: pump.fun, PumpSwap, Raydium, Meteora, LetsBonk
- **Tape writer** — append-only archive to object storage
- Postgres + Timescale schema, migrations
- Enrichment v1: safety scoring (authorities, Token-2022 extensions, LP state, honeypot
  simulation), holder analytics, bundle detection, deployer lineage
- Wallet reputation engine (backfilled over ≥90 days)
- Data quality monitors + alerting
- Read-only dashboard: system health, token feed, safety scores

**Gate:** 7 consecutive days of tape with <1% slot gaps · safety scoring correctly
classifies a hand-labeled set of 50 known rugs and 50 known survivors · wallet reputation
produces a stable ranking (period-over-period rank correlation measured and reported,
whatever it says) · p99 ingest lag < 200ms.

---

## Phase 2 — Simulation & Research *(weeks 4–6)*

**Deliverables**
- Replay engine (deterministic, clock-gated)
- Fill simulator with latency, impact, fees, failure rate, adverse selection
- Strategy framework — pure evaluators + versioned configs
- S4 (Survivor Momentum) implemented and backtested
- S1 (Smart-Money Mirror) implemented and backtested
- Research notebooks: forward-return distributions, wallet skill persistence, the
  **entry-lag decay curve** (the key open question from [doc 05](05-strategy-portfolio.md))
- Backtest reporting with bootstrap CIs

**Gate:** ≥1 strategy passing in-sample → out-of-sample → walk-forward from
[doc 09](09-validation-and-backtesting.md) §9.4, with ≥200 simulated trades and positive
expectancy net of the full cost model.

**If this gate fails:** we stop here. Total spend to this point is roughly $200–600 and
6 weeks, and we've answered the actual question — is there a rule-expressible edge
available at our latency — for a very small fraction of the bankroll. That's the cheap
fork [doc 01](01-objective-and-mandate.md) §1.5 is designed to reach.

---

## Phase 3 — Execution & Risk *(weeks 6–8)*

**Deliverables**
- Signer service with full policy enforcement, isolated
- Executor: Jupiter quoting, priority fee oracle, Jito bundles with `dontfront`,
  pre-flight simulation, confirmation tracking, idempotency
- Risk engine: sizing, all hard limits, all circuit breakers
- Position manager: TP ladders, stops, trailing, time stops
- S5 safety-degradation exit on its own priority lane
- Crash recovery: on-chain reconciliation at boot
- Dashboard: positions, trades, **kill switch**, alerting
- Paper trading against live data

**Gate:** 30 days paper trading · expectancy within 30% of backtest · zero unhandled
infra incidents in the final 7 days · kill switch and panic-exit tested under load ·
crash recovery rehearsed by killing the process with open simulated positions.

---

## Phase 4 — Micro Live *(weeks 8–12)* — first real money

**$250 bankroll. $10 maximum position.** Deliberately small enough that total loss is
irrelevant and large enough that fills, slippage and adverse selection are real.

**Deliverables**
- Live trading at L1
- Fill calibration: real vs. simulated, weekly
- Tax lot tracking from the very first trade
- Nightly post-mortem agent (A4)
- Weekly review process, written

**Gate:** ≥100 live trades · expectancy > 0 at 80% confidence (bootstrap) · max drawdown
< 25% · realized fills within 150 bps of simulation · zero security incidents.

**This is the real decision point.** Everything before it is construction; this is the
first evidence. If it fails, [doc 01](01-objective-and-mandate.md) §1.3's kill criteria
apply.

---

## Phase 5 — Scale Ladder *(weeks 12–24)*

Climb the bankroll ladder in [doc 06](06-risk-engine.md) §6.5: L2 $1k → L3 $5k → L4 $15k.
Each rung requires 100 trades meeting criteria. Demotion is automatic on a 20% drawdown.

**Deliverables**
- S2 and S3 implemented and put through the full validation pipeline
- Agents A1, A2, A3 enabled (L3+, where their cost is defensible)
- Portfolio allocation across strategies, repriced monthly
- Full dashboard
- Monthly strategy review, agent A/B scorecards

**Gate per rung:** as above. Real gates, real demotion.

---

## Phase 6 — Ongoing *(month 6+)*

Steady state: research loop producing and retiring strategies, monthly allocation
review, quarterly infrastructure review, continuous edge-decay monitoring. Optional
expansions, each requiring its own business case: second chain, adjacent venues
(prediction markets are where the speculative flow has been rotating — see
[doc 02](02-market-reality.md) §2.2), Rust hot path if measurement justifies it.

---

## Critical path and what can be parallelized

```
Phase 0 ─▶ Phase 1 (ingest ─▶ tape ─▶ enrichment) ─▶ Phase 2 (replay ─▶ strategies)
                                                         │
                       Phase 3 (signer + executor) ──────┤  can start during Phase 2
                                                         ▼
                                                     Phase 4 ─▶ Phase 5
```

The signer and executor can be built in parallel with strategy research — they have no
dependency on which strategy wins. The dashboard trails throughout.

**Dependencies that cannot be parallelized:** the tape must exist before honest
backtesting; the risk engine must exist before real money; 100 trades must accumulate
before any scaling decision, and no amount of engineering makes that faster.
