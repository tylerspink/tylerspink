# Agentic Meme Coin Trading System — Planning Repository

> **Status: planning only. No code, no keys, no capital committed.**
> This repository currently contains the design brief for an autonomous meme coin
> trading system. Nothing here executes trades.

---

## What this is

A design for a system that autonomously trades Solana meme coins under a fixed,
machine-enforced risk mandate, with LLM agents doing research, strategy authoring
and post-trade review — and **deterministic code doing all execution**.

## Read in this order

| # | Document | What it answers |
|---|----------|-----------------|
| 01 | [Objective & Mandate](docs/01-objective-and-mandate.md) | What "make as much money as possible" has to become to be buildable |
| 02 | [Market Reality](docs/02-market-reality.md) | Who we're competing with, what the fees actually cost, what the base rates are |
| 03 | [System Architecture](docs/03-system-architecture.md) | The nine layers, the repo layout, the stack |
| 04 | [Data Layer](docs/04-data-layer.md) | Where signal comes from and how it's stored |
| 05 | [Strategy Portfolio](docs/05-strategy-portfolio.md) | Five candidate strategies, ranked, with falsification tests |
| 06 | [Risk Engine](docs/06-risk-engine.md) | Position sizing, circuit breakers, the thing that says no |
| 07 | [Custody & Security](docs/07-custody-and-security.md) | How the bot trades your money without being able to steal it |
| 08 | [Agent Design](docs/08-agent-design.md) | Where the LLM goes, and where it must never go |
| 09 | [Validation](docs/09-validation-and-backtesting.md) | How we know a strategy works before risking money |
| 10 | [Dashboard](docs/10-dashboard.md) | What you see and what you can press |
| 11 | [Roadmap](docs/11-roadmap.md) | Six phases, each with a gate that can fail |
| 12 | [Costs & Capital](docs/12-costs-and-capital.md) | The monthly burn and the bankroll ladder |
| 13 | [Legal, Tax & Compliance](docs/13-legal-tax-compliance.md) | Taxable events, what not to do |
| 14 | [Open Decisions](docs/14-open-decisions.md) | The nine things I need from you before Phase 1 |

Architecture decision records live in [`docs/adr/`](docs/adr/).

---

## The four claims this plan rests on

1. **The LLM must be out of the hot path.** Agents that reason for 2–10 seconds cannot
   win launch races against co-located bots that land in under 50ms. The LLM's edge is
   in *selection and review*, not *speed*. Execution is deterministic TypeScript/Rust
   with a hard latency budget.

2. **Fees, not strategy, are the first enemy.** A realistic round trip on a thin Solana
   meme pool costs **3–8%** all-in (venue fee + LP fee + priority fee + Jito tip +
   slippage both ways). Any strategy must clear that before it clears zero. This kills
   high-frequency scalping at small size and forces fewer, higher-conviction positions.

3. **The mandate must be bounded or it self-destructs.** "Maximize money" with no
   drawdown constraint has a mathematically optimal solution — bet everything, every
   time — whose long-run outcome is ruin with probability approaching 1. See
   [doc 01](docs/01-objective-and-mandate.md) for the reformulation.

4. **We are entering a contracting market.** Solana memecoin volume was ~$19B in
   September 2026, down 38% month-over-month, and memecoins fell from 40% to 16% of
   Solana spot volume across H1 2026. Less volume means thinner exits and worse fills.
   This is a headwind the plan has to price in, not ignore.

## The honest base rate

Published Dune analyses of pump.fun put the share of wallets that have realized
**$1,000 or more** in lifetime profit at roughly **1–5%**, and the share above $10,000
at well under 1%. Roughly 1–2% of launched tokens ever graduate their bonding curve.

That does not make the project pointless — it makes the *target* clear. We are not
trying to be an average participant. We are trying to build the small number of
structural advantages an average participant doesn't have: recorded-tape validation,
enforced position sizing, automated rug filtering, and a kill switch that fires before
a bad week becomes a terminal one.

If after the Phase 4 gate the system's measured expectancy is negative, the correct
action is to stop, and this plan says so in writing before any money is at stake.
