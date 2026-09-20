# 04 — Data Layer

Signal quality is the binding constraint on this system. Execution is an engineering
problem with a known solution; knowing *what to buy* is not.

## 4.1 Real-time on-chain data

**Primary: Yellowstone gRPC (Geyser).** Push-based streaming of accounts,
transactions, blocks and slots directly from validator infrastructure — the standard
for Solana bots. Candidate providers:

| Provider | Product | Notes |
|---|---|---|
| Helius | LaserStream | Yellowstone-compatible, adds historical replay, auto-reconnect, multi-region. The replay feature is directly useful for our tape. |
| Triton One | Yellowstone gRPC | Reference implementation maintainers |
| Chainstack / dRPC / OrbitFlare | Yellowstone endpoints | Competitive pricing |

**Recommendation:** start on Helius (developer tier) for LaserStream's reconnect and
replay semantics, keep a second provider configured for failover, and only move to a
business tier when Phase 3 latency measurements justify it. Provider choice is behind
an interface — we should never be locked in.

**Subscriptions to establish:**
- Transactions mentioning launchpad program IDs (pump.fun, LetsBonk, others)
- Account updates for bonding curve accounts of watched tokens
- Pool creation + swap events on PumpSwap, Raydium, Meteora, Orca
- Transactions from the tracked smart-money wallet set
- Balance changes on our own wallets (reconciliation)

**Not using:** polling `getSignaturesForAddress`. It is seconds behind and rate-limited.

## 4.2 The tape — non-negotiable

Every ingested event is written, **before processing**, to an append-only archive:
zstd-compressed protobuf, partitioned by hour, in object storage.

This single decision is what separates a real system from a hopeful one:

- It is the **only** way to backtest honestly. Aggregated OHLCV from a public API has
  survivorship bias baked in (dead tokens get delisted), no order-book depth, and no
  information about *when we would actually have known* something.
- It lets us replay a strategy change against the exact conditions of last Tuesday.
- It is the forensic record when a trade goes wrong.

Cost is trivial — on the order of a few GB/day compressed, dollars per month.
Build it in Phase 1, before any strategy work. **If we skip this, every backtest
number the project produces afterwards is fiction.**

## 4.3 Enrichment & safety data

**Compute in-house** (cheaper, faster, and it's our edge):
mint/freeze authority state, Token-2022 extension inspection, LP burn/lock, holder
distribution, same-slot bundle detection, deployer lineage, wallet PnL reputation.

**Third-party, for cross-check only** — never as the sole gate, since these are
rate-limited, lag, and can be gamed:
- **RugCheck** — Solana token risk scanning; usable as a second opinion
- **Birdeye** — Solana-native analytics, token/wallet data, decent API
- **DexScreener / GeckoTerminal** — broad pair coverage, free tiers, good for discovery
  and for a sanity check on price

Policy: a token must pass **our** safety gate. A third-party green light never
overrides a local red flag; a third-party red flag is always disqualifying.

## 4.4 Off-chain signal

This is where the LLM has a genuine, defensible edge, and it's the least commoditized
input in the stack.

| Source | Use | Difficulty |
|---|---|---|
| X/Twitter | KOL mentions, narrative emergence, engagement velocity | High — API is expensive; needs careful account-quality weighting |
| Telegram | Call channels, alpha groups | Medium — userbot client, high noise, many are paid shills |
| News / events | The "why now" behind a meta (a celebrity, a headline, a meme) | Medium — RSS + LLM classification |
| Google Trends / TikTok | Mainstream attention lead indicator | Low value at our timescale |
| Farcaster / on-chain social | Less saturated than X | Low volume |

**The discriminating feature is not mention count — it is mention *velocity* weighted by
account quality, plus the derivative of that.** Raw counts are trivially manipulated.

**Hard rule:** all of this is attacker-controlled text. It enters the system as *data*
in a structured field, is never concatenated into an instruction, and never
short-circuits the safety gate. See [doc 07](07-custody-and-security.md).

## 4.5 Storage model

```
Postgres + TimescaleDB
  tokens                 mint, venue, deploy slot, deployer, metadata, first_seen
  token_safety           point-in-time safety snapshots (append-only, never updated)
  pools                  pool address, venue, token pair, created slot
  trades_raw       [HT]  every observed swap: slot, wallet, side, amounts, price
  candles_1s/1m    [HT]  continuous aggregates derived from trades_raw
  wallets                reputation: realized pnl, hit rate, hold time, last active
  signals                every strategy signal emitted, taken or not (critical: we must
                         be able to evaluate the trades we *didn't* take)
  intents                strategy output pre-risk
  risk_decisions         approve/reduce/reject + reason code (the veto audit log)
  orders                 submitted transactions, quoted vs realized, fees, latency
  positions              open + closed, entry/exit legs, realized pnl, fees, tax lots
  agent_runs             every LLM call: prompt hash, model, tokens, cost, output
  events                 system event log for the dashboard timeline

Redis
  hot features, quote cache, rate limiters, dedup keys, Streams as the message bus

Object storage
  tape/YYYY/MM/DD/HH/*.pb.zst    the raw event archive
```

Two schema notes that matter more than they look:

1. **`signals` records rejected signals too.** Without the counterfactual we cannot tell
   whether the risk engine is protecting us or strangling us.
2. **`token_safety` is append-only, point-in-time.** A backtest that reads *today's*
   safety verdict for a trade made last week is look-ahead bias, and it is the most
   common way a meme coin backtest lies to you.

## 4.6 Data quality monitors

Silent data degradation is the most dangerous failure in the system, because the bot
keeps trading on stale inputs. Continuous checks, all alerting:

- Slot lag: our processed slot vs. chain head (alert > 5 slots, halt entries > 20)
- Gap detection: missing slots in the tape
- Price cross-check: our computed price vs. Jupiter quote (alert > 2% divergence)
- Feature staleness: age of every cached feature, with per-feature TTLs
- Enrichment failure rate
- Quoted vs. realized fill divergence, tracked as a rolling distribution

## 4.7 Sources

- [Complete 2026 guide to Solana streaming and Yellowstone gRPC — Triton](https://blog.triton.one/complete-guide-to-solana-streaming-and-yellowstone-grpc/)
- [Helius LaserStream](https://www.helius.dev/laserstream)
- [Helius Yellowstone gRPC docs](https://www.helius.dev/docs/grpc)
- [Solana trading infrastructure 2026 — Chainstack](https://chainstack.com/solana-trading-infrastructure-2026/)
- [RugCheck — Solana token risk scanner](https://rugcheck.xyz/)
- [Solana token analytics tools compared 2026](https://createmycoin.app/articles/solana-token-analytics-tools-compared)
