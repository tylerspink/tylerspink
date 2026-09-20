# 03 — System Architecture

## 3.1 The governing principle

```
                    SLOW PATH (LLM allowed)          FAST PATH (no LLM, ever)
                    ────────────────────────         ────────────────────────
  Research      ->  narrative detection
  Analysis      ->  token scoring, wallet vetting
  Authoring     ->  proposes strategy CONFIG  ──┐
  Review        ->  nightly post-mortem         │
                                                ▼
                                          [ strategy engine ]  <- deterministic rules
                                                ▼                  p99 < 50ms
                                          [  risk engine    ]  <- can only veto/reduce
                                                ▼                  p99 < 10ms
                                          [   executor      ]  <- builds + submits tx
                                                ▼                  p99 < 400ms
                                          [ signer service  ]  <- isolated, policy-gated
```

The LLM produces **configuration and judgment**, never a signed transaction. The gap
between those two things is the safety property the whole system depends on.

## 3.2 The nine layers

### L0 — Infrastructure
Single VPS (16 vCPU / 32GB / NVMe) co-located in the same region as the chosen
Geyser provider. Docker Compose for v1; Kubernetes is premature. Postgres 16 +
TimescaleDB, Redis 7, object storage for tape archives. Prometheus + Grafana + Loki.

### L1 — Ingestion
Consumes real-time Solana state. Sources in [doc 04](04-data-layer.md). Key property:
**everything ingested is written to an append-only tape before it is processed.** The
tape is what makes honest backtesting possible and is the single highest-leverage
component in the system.

- Yellowstone gRPC / LaserStream subscription (accounts, transactions, slots)
- Program-log decoders per venue (pump.fun, PumpSwap, Raydium, Meteora, LetsBonk)
- Off-chain pollers (price APIs, social feeds) on their own cadence
- Backpressure: bounded queues, drop-and-count on overflow, alert on any drop

### L2 — Enrichment
Turns raw events into features. Stateless where possible, cached in Redis.

- **Safety scoring:** mint/freeze authority, Token-2022 extensions, LP burn/lock state,
  simulated sell (honeypot probe), transfer-hook inspection
- **Holder analytics:** top-10 concentration, holder count velocity, dev holdings
- **Launch forensics:** same-slot buyer clustering (bundle detection), sniper wallet
  overlap, deployer lineage graph
- **Liquidity & flow:** pool depth, realized depth at ±5%, net buy/sell flow, unique
  buyer count, volume/liquidity ratio
- **Wallet reputation:** rolling realized PnL per wallet, hit rate, hold time
  distribution — the input to smart-money mirroring

### L3 — Strategy Engine
Pure functions: `(features, state, config) -> Intent | null`. No I/O, no LLM calls, no
randomness. Every strategy is a versioned config + a deterministic evaluator, which is
what makes tape replay meaningful. Strategies emit *intents*, not orders.

### L4 — Risk Engine
The only component with authority to say no. Sits between every intent and the
executor, has no other job, and is the piece most worth over-engineering. Full spec in
[doc 06](06-risk-engine.md). Property: it can only ever **reduce or reject** — it has no
code path that increases a position's size.

### L5 — Execution
Turns approved intents into landed transactions.

- Quote via Jupiter; compare against direct venue quote; use the better one
- Dynamic priority fee from a percentile oracle over recent slots
- Jito bundle submission with `dontfront` for sandwich protection; tight slippage
  tolerance as the primary defense
- Pre-flight simulation on every transaction, no exceptions
- Confirmation tracking with bounded retry and idempotency keys (never double-buy)
- Records **quoted vs. realized** on every fill — this feeds back into the slippage
  model and is how we detect edge decay

### L6 — Position Manager
Owns open positions and exits. Exits are far more important than entries and get more
engineering attention.

- Laddered take-profit (e.g. 40% at +50%, 30% at +150%, runner with trailing stop)
- Hard stop-loss and trailing stop
- Time stop: no position survives past its thesis horizon without re-validation
- **Safety-degradation exit:** immediate full exit if LP unlocks, mint authority
  reappears, top holder dumps, or liquidity drops below a floor. This runs on a
  separate high-priority loop.

### L7 — Agent Layer
LLM agents. Detailed in [doc 08](08-agent-design.md). Read-mostly, gate-bounded,
structurally unable to sign anything.

### L8 — Interface
Next.js dashboard, API, alerting, and the kill switch. [doc 10](10-dashboard.md).

## 3.3 Repository layout

```
apps/
  dashboard/          Next.js 15, Tailwind, shadcn — read + control surface
  api/                Fastify — REST + WebSocket, auth, serves the dashboard
workers/
  ingest/             Geyser consumer, venue decoders, tape writer
  enrich/             safety scoring, holder analytics, wallet reputation
  strategy/           strategy evaluators (pure), signal bus publisher
  risk/               risk engine + position manager + circuit breakers
  executor/           quoting, tx building, submission, confirmation
  agents/             research, analyst, post-mortem, supervisor
services/
  signer/             ISOLATED. Holds the key. Policy-enforcing. Own container,
                      own network namespace, no outbound internet.
packages/
  core/               domain types + zod schemas (the contract between everything)
  solana/             RPC/Geyser clients, program decoders, tx builders
  sim/                tape replay engine + fill simulator
  db/                 drizzle schema + migrations
  config/             strategy configs, versioned, reviewed like code
infra/
  docker/  grafana/  prometheus/  scripts/
docs/                 this directory
```

## 3.4 Stack recommendation

| Concern | Choice | Why |
|---|---|---|
| Language (all but hot path) | **TypeScript, Node 22** | Best Solana library ecosystem; one language across bot and dashboard; iteration speed matters more than microseconds for our chosen strategies |
| Hot path | **TypeScript v1, Rust later** | Our strategies have 30s–30min decision windows. Rust buys ~50–150ms we don't need yet. Revisit only if measured latency, not intuition, says so. |
| Database | **Postgres 16 + TimescaleDB** | Hypertables for tick/candle data; one database instead of three |
| Cache / bus | **Redis 7 (Streams)** | Consumer groups give us a durable bus without running Kafka |
| Tape archive | **Object storage, zstd-compressed protobuf** | Cheap, append-only, replayable |
| ORM / validation | **Drizzle + Zod** | Typed end-to-end; Zod schemas double as the LLM structured-output contract |
| Frontend | **Next.js + Tailwind + shadcn + TanStack Query** | Fast to build, good realtime story |
| Observability | **Prometheus + Grafana + Loki + Sentry** | Latency histograms are a first-class product feature here |
| Deployment | **Docker Compose on one VPS** | Right-sized. Distributed systems are a Phase 5 problem. |

**Explicit non-goals for v1:** Kubernetes, microservices beyond the list above, a custom
validator, running our own Solana RPC node, multi-chain, a mobile app.

## 3.5 Latency budget

Measured end-to-end, event on chain → our transaction landed:

| Segment | Budget (p99) |
|---|---|
| Geyser event → ingest worker | 80ms |
| Decode + enrich (cache hit) | 30ms |
| Strategy evaluation | 50ms |
| Risk engine | 10ms |
| Quote fetch | 150ms |
| Build + sign | 40ms |
| Submit → landed | 600–1500ms |
| **Total** | **~1.0–1.9s** |

This is roughly 20–40x slower than a co-located sniper, and that is a deliberate,
accepted design choice. Every strategy in [doc 05](05-strategy-portfolio.md) is
selected to be viable at ~2s. If a strategy needs 200ms, we don't run it.

## 3.6 Failure modes the architecture must survive

| Failure | Response |
|---|---|
| Geyser stream drops | Auto-reconnect with replay from last slot; halt new entries while stale; alert |
| RPC degraded / rate limited | Failover to secondary provider; if both degraded, halt entries, keep exits alive |
| Transaction fails to land | Bounded retry with fee escalation; abandon after N; never double-submit (idempotency key) |
| Priority fees spike | Fee oracle caps fee at % of position value; skip trades that fail the cap |
| Risk engine unavailable | **Fail closed.** No risk engine, no trades. Exits still permitted. |
| Signer unavailable | No trades. Alert immediately. |
| Bot process crashes with open positions | Positions are persisted in Postgres, not memory; recovery reconciles on-chain balances against the position table at boot, before doing anything else |
| Dashboard down | Trading unaffected; alerting is a separate path (push notification, not the dashboard) |
