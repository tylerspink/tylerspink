# 02 — Market Reality

Everything in this plan is downstream of three facts: what a trade costs, who we are
trading against, and how fast the field is moving. This document establishes those
numbers so the strategy document has something to clear.

## 2.1 The venue landscape (as of September 2026)

**Chain: Solana.** Recommendation is Solana-first, and probably Solana-only for v1.
It has the deepest meme coin order flow, the best real-time data infrastructure
(Yellowstone gRPC / Geyser), sub-second finality, and fees low enough that a $100
position isn't eaten by gas. Base and BSC are real secondary markets but split our
engineering across two execution stacks for a fraction of the flow. Revisit at Phase 5.

**Launchpads and venues to index:**

- **pump.fun** — still the dominant bonding-curve launchpad (roughly 75–80% share of
  graduated tokens through 2025, under real competitive pressure since). Tokens sell
  ~800M supply along a bonding curve, then "graduate" at roughly $69k market cap.
  Since 2025 graduation migrates to **PumpSwap**, pump.fun's own AMM, rather than
  Raydium. As of June 2026 it also supports USDC-denominated curves.
- **LetsBonk** — took substantial launchpad share during 2025–26; must be indexed.
- **Bags, Believe, and successors** — smaller, but new launchpads appear continuously.
  The ingest layer must be *pluggable by program ID*, not hardcoded to pump.fun.
- **AMMs:** PumpSwap, Raydium (AMM v4 + CLMM), Meteora (DLMM), Orca (Whirlpools).
- **Aggregator:** Jupiter, for routing and quotes.

**Design consequence:** venue adapters are an interface, not a special case. Adding a
new launchpad should be one decoder module plus a config entry.

## 2.2 Market conditions — the headwind

This matters enough to state up front. The meme coin market we would be entering is
materially smaller than the one that produced the famous returns:

- Solana memecoin trading volume: **~$19B in September 2026, down 38% month-over-month**
  from ~$31B in August.
- Memecoins fell from **40% to 16%** of Solana spot volume across H1 2026.
- Solana network revenue fell **~87%** in H1 2026, driven by the collapse in memecoin
  priority fees and MEV tips.
- Speculative flow has been rotating elsewhere — prediction market volume roughly
  doubled to ~$4.3B over the same window.

**What this changes:**

1. **Thinner exits.** Lower volume means the depth you sell into is worse than the
   depth you bought into. Slippage models built on 2025 data will be optimistic.
2. **Fewer high-quality setups per day.** Strategy throughput assumptions must be
   conservative — plan for 2–10 qualified signals/day, not 100.
3. **Capacity ceiling drops.** The size at which our own orders move the market is
   lower than it was.
4. **It argues for building the pipeline venue-agnostic.** If the meta rotates to
   prediction markets or a different chain, a system whose data spine and risk engine
   are generic can follow. One hardcoded to pump.fun cannot. This is a real
   architectural requirement, not a hedge.

## 2.3 The cost of a round trip — the number that governs everything

Worked example: a $100 position in a token with $40k of pool liquidity.

| Cost component | Buy | Sell | Notes |
|---|---|---|---|
| Venue fee (pump.fun curve) | ~1.0% | ~1.0% | Bonding-curve phase; includes creator fee split |
| AMM LP fee (post-graduation) | 0.25–0.30% | 0.25–0.30% | PumpSwap / Raydium |
| Priority fee | $0.05–0.60 | $0.05–0.60 | Competitive conditions; higher during congestion |
| Jito tip | $0.10–1.00 | $0.10–1.00 | Min 1,000 lamports; real tips far higher when contested |
| Price impact | 0.3–2% | 0.3–3% | Scales with position ÷ pool depth |
| Adverse selection / latency | 0.5–3% | 0.5–5% | You are late to every move you didn't cause |

**Realistic all-in round trip: 3–8%.** On a $100 position that's $3–8 of cost before
the trade has any opinion.

Three consequences that shape the entire strategy set:

1. **Scalping is dead at our size.** A strategy targeting 2% moves has negative
   expectancy by construction. The minimum viable target move is roughly **15–20%**,
   which means holding periods of minutes-to-hours, not seconds.
2. **Fixed costs favor larger positions — pool depth caps how large.** Priority fees
   and tips are fixed per transaction, so they're a 1% tax on a $50 trade and a 0.05%
   tax on a $1,000 trade. But price impact scales the other way. The optimum for
   typical meme pool depth sits around **$200–800 per position**, which in turn implies
   a bankroll of roughly **$10k–40k** for the risk caps in [doc 06](06-risk-engine.md)
   to produce sensibly-sized trades. Below that, fixed costs dominate.
3. **Capacity is bounded by liquidity, not by conviction.** Rule: position ≤ 0.5% of
   pool liquidity. A $40k pool supports a $200 position. Scaling the bankroll 10x does
   not scale returns 10x; it forces us into either bigger tokens (different edge) or
   more concurrent positions (different risk profile).

## 2.4 Who we are actually competing against

Honest ranking of the field, fastest to slowest:

| Competitor | Latency | Their edge | Can we beat them? |
|---|---|---|---|
| Co-located snipers (ShredStream, SWQoS, Jito bundles) | <50ms | Raw speed, stake-weighted transaction priority | **No.** Not at any realistic budget. |
| Bundlers / dev-affiliated wallets | Same-block | They *are* the information | No. Detect and avoid. |
| MEV sandwich bots | Same-slot | Position in the block | Defend, don't compete |
| Copy-trade bot swarms | 0.5–3s | Mirroring known wallets | Partially — wallet *selection* is the differentiator |
| Telegram-call / KOL followers | 5–60s | Distribution | Yes, with better filtering |
| Discretionary humans | minutes | Judgment, narrative | Yes — this is where an LLM competes |

**The strategic conclusion:** do not build a launch sniper. We would lose every race,
and losing a launch race means buying the top from the bot that won it. Build strategies
whose decision window is **30 seconds to 30 minutes**, where analysis quality beats
reaction time, and where an LLM's ability to read narrative and context is an actual
asset rather than a latency liability.

## 2.5 The adversarial environment

Meme coin markets are adversarial in ways normal markets are not. The system must treat
every token as hostile until proven otherwise:

- **Honeypots** — buys succeed, sells revert. Detected via transfer-hook inspection and
  simulated sell before entry.
- **Mint authority not renounced** — dev can inflate supply to zero your position.
- **Freeze authority not renounced** — dev can freeze your token account. Fatal.
- **Token-2022 extension traps** — transfer fees, transfer hooks, permanent delegate,
  non-transferable. A whole class of newer traps that a naive SPL check misses.
- **Unlocked / unburned LP** — dev pulls liquidity. Check LP token burn or lock.
- **Bundled launches** — dev buys their own supply across many wallets in the launch
  block, then distributes into retail buyers. Detectable via same-slot buyer clustering.
- **Serial ruggers** — the same deployer wallet, funded from the same source, across
  dozens of failed tokens. A deployer-lineage graph is one of the cheapest real edges
  available and should be built in Phase 1.
- **Fake volume / wash trading** — inflated volume drawing in momentum bots. Detect via
  trade-size uniformity and counterparty wallet overlap.
- **Prompt injection via token metadata.** Token names, symbols, descriptions, and
  linked socials are **attacker-controlled text that our agents will read**. Some of it
  will contain instructions aimed at an LLM. This is a first-class threat, covered in
  [doc 07](07-custody-and-security.md).

## 2.6 Sources

- [Pump.fun market share and graduation mechanics — TradingView/Cointelegraph](https://www.tradingview.com/news/cointelegraph:9c3a24b10094b:0-how-pump-fun-captured-80-of-solana-memecoins-and-can-it-last/)
- [Pump.fun launchpad guide 2026 — DEXTools](https://www.dextools.io/tutorials/what-is-pump-fun-solana-memecoin-launchpad-2026)
- [Meme Launchpad 2.0: pump.fun and LetsBonk — bex.co](https://bex.co/blog/2026/04/22/meme-launchpad-2-pump-fun-letsbonk-anti-sniper-bonding-curve-professionalization)
- [21Shares: Solana H1 2026 network revenue fell 87% as memecoin fees collapsed](https://solanacompass.com/news/21shares-solanas-h1-2026-network-revenue-fell-87-as-memecoin-fees-collapsed)
- [Prediction markets double to $4.3B as Solana memecoin trading slumps — CryptoSlate](https://cryptoslate.com/new-degen-trenches-prediction-markets-double-volume-to-4-3b-as-solana-memecoin-trading-slumps/)
- [99.6% of pump.fun traders have not realized over $10,000 — Cointelegraph](https://cointelegraph.com/news/pump-fun-crypto-traders-majority-do-not-realize-profits-dune-data)
- [Only 0.76% of pump.fun wallets made $1,000 or more — crypto.news](https://crypto.news/only-0-76-of-pump-fun-wallets-made-1000-or-more-cn-research/)
- [Pump.fun trader profitability, April 2026 — CoinGecko Research](https://www.coingecko.com/research/publications/pump-fun-traders-are-making-a-comeback)
- [Solana trading infrastructure 2026: MEV, nodes, latency — Chainstack](https://chainstack.com/solana-trading-infrastructure-2026/)
- [Jito explained: bundles, tips and MEV in 2026 — Chainstack](https://chainstack.com/jito-explained-bundles-tips-mev-solana/)
- [MEV protection with Jito DontFront — solana.com](https://solana.com/developers/guides/advanced/mev-protection)
