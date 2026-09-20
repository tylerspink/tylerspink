# ADR-0001 — LLM stays out of the execution path

- **Status:** accepted
- **Date:** 2026-09-20

## Context

The brief asks for an "agentic" trading bot. The obvious reading — an LLM agent that
observes the market and issues buy/sell orders — collides with three constraints:

1. **Latency.** LLM inference is 200ms–10s. Co-located Solana snipers land transactions
   in under 50ms using ShredStream and stake-weighted QoS. Any strategy requiring speed
   is lost before it starts.
2. **Non-determinism.** The same input yields different outputs across calls. This makes
   backtesting meaningless — you cannot replay a strategy whose behavior isn't a function
   of its inputs.
3. **Adversarial input.** Token metadata is attacker-controlled text that costs cents to
   publish. An LLM with signing authority reading that text is a standing invitation to
   prompt injection, sprayed across thousands of cheap tokens until one lands.

## Decision

The LLM produces **configuration, scores, nominations and reviews**. Deterministic code
produces **detection, sizing, execution and risk decisions**. No agent holds signer
credentials or can raise a risk limit.

Agents may autonomously **reduce** risk (halt, exit, de-size) and may never **increase**
it without passing a validation gate.

## Alternatives considered

- **LLM issues orders directly, guarded by a risk engine.** Rejected: the risk engine
  bounds the damage per trade but not the strategy's coherence, and non-determinism still
  destroys backtestability — we would never know whether a config change or sampling
  noise caused a result.
- **Small fast model in the hot path.** Rejected: still 10–50x slower than the
  alternative, still non-deterministic, still injectable, and buys nothing a rule can't
  express for the strategies we selected.
- **No LLM at all.** Rejected: narrative detection and post-trade review are genuinely
  better with one, and neither is latency-sensitive. Discarding those gives up the only
  advantage we hold over faster competitors.

## Consequences

**Makes easy:** deterministic replay, honest backtesting, bounded blast radius from
injection, cheap inference (agents run on their own cadence rather than per-event).

**Makes hard:** strategies must be expressible as rules. Genuinely judgment-heavy setups
either get encoded into features or don't get traded. Some edge is left on the table, and
that's accepted.

**Commits us to:** a structured-output contract (Zod schemas in `packages/core`) as the
sole interface between agents and the trading system.

**Revisit if:** inference latency drops below ~20ms at useful quality, *and* a
deterministic replay story for non-deterministic components exists. Neither is close.
