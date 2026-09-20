# 08 — Agent Design

## 8.1 Where the LLM belongs

The word "agentic" in the brief is doing a lot of work, so let's be precise about what
the agents do and don't do.

**What LLMs are genuinely good at, in this domain:**
- Reading unstructured context (news, social, token metadata) and extracting structure
- Recognizing that six seemingly unrelated tokens share a theme
- Writing and revising trading rules given performance feedback
- Explaining *why* a set of trades lost money, in a way that generates a testable fix

**What LLMs are bad at, in this domain:**
- Being fast (200ms–10s per call vs. our 50ms strategy budget)
- Consistency (the same input produces different outputs)
- Calibrated probability estimates (systematically overconfident)
- Arithmetic under pressure, which is what position sizing is
- Resisting persuasion by adversarial text — see [doc 07](07-custody-and-security.md) §7.3

**Therefore:** agents do research, authoring and review. Deterministic code does
detection, sizing, execution and risk. An LLM never sits between a signal and a fill.

## 8.2 The agent roster

### A1 — Research Agent (narrative detection)
- **Cadence:** every 15 minutes
- **Reads:** news feeds, X/Telegram aggregates, on-chain token-naming clusters, trending pairs
- **Emits:** `NarrativeCandidate { theme, evidence[], confidence, expected_duration_h, candidate_mints[] }`
- **Model:** frontier (this is the hard reasoning task)
- **Constraint:** output is a *nomination*. Deterministic on-chain confirmation is
  required before any entry. Feeds S3 only.

### A2 — Token Analyst
- **Cadence:** on demand, when a candidate passes the mechanical safety gate
- **Reads:** structured features + fenced untrusted metadata
- **Emits:** `TokenAssessment { quality_score 0-100, flags[], reasoning, injection_detected }`
- **Model:** fast/cheap (high call volume)
- **Constraint:** score is *one input* to a deterministic scoring function, and it can
  only ever reduce confidence, never raise it above what on-chain data supports. This is
  the agent most exposed to adversarial text, so it is deliberately the least privileged.

### A3 — Strategy Author
- **Cadence:** weekly, or on post-mortem trigger
- **Reads:** trade history, performance attribution, current strategy configs, rejection log
- **Emits:** a proposed strategy config diff + hypothesis + expected effect
- **Model:** frontier
- **Constraint:** proposals go to backtest → paper → **your approval**. Never auto-deployed.
  It writes config, not code that touches execution.

### A4 — Post-Mortem Agent
- **Cadence:** nightly, plus after any circuit breaker fires
- **Reads:** the day's trades, signals (including untaken), risk decisions, fills, slippage
- **Emits:** structured findings — what worked, what didn't, attribution, proposed
  parameter changes, data-quality anomalies
- **Model:** frontier
- **Value:** this is the compounding component. The trade journal plus a disciplined
  nightly review is how strategy quality improves over months.

### A5 — Supervisor
- **Cadence:** continuous, every 60s over system metrics
- **Watches:** slippage drift, fill-rate degradation, latency, correlated drawdown,
  data staleness, anomalous patterns not covered by a specific circuit breaker
- **Can autonomously:** **HALT trading.** That's it.
- **Rationale:** the asymmetric-permission principle in concrete form. A supervisor that
  can only stop things is safe even if it's wrong — a false halt costs opportunity; a
  false start costs money.

## 8.3 Structural constraints on all agents

1. **No agent can sign a transaction.** No agent process has access to signer credentials.
2. **No agent can raise a limit.** Risk parameters come from reviewed config, not agent output.
3. **All output is schema-validated** (Zod) and out-of-range values are clamped, with the
   clamping logged as an anomaly.
4. **All agent runs are logged** — prompt hash, model, tokens, cost, output, downstream effect.
5. **Agents may always reduce risk, never increase it without a gate.**
6. **Untrusted input is fenced and labeled** at every boundary.
7. **Cost-capped.** A daily LLM spend ceiling that halts agent activity, not trading.

## 8.4 Memory and learning

Three layers, deliberately kept separate so that "learning" doesn't quietly become
"prompt drift":

1. **Trade journal** (Postgres) — every trade with full context. The ground truth.
2. **Post-mortem archive** (vector store) — nightly findings, retrievable so the strategy
   author can see "we tried this in March and it failed for reason X."
3. **Strategy configs** (git, versioned, reviewed) — the only thing that actually changes
   behavior.

**What we are not doing:** fine-tuning, RL on trade outcomes, or letting an agent edit
its own prompt. With a few hundred trades of noisy data, every one of those overfits
catastrophically. Learning happens through hypothesis → backtest → paper → gate, which
is slower and is the only version that generalizes.

## 8.5 Model routing and cost

| Agent | Model tier | Est. calls/day | Est. cost/day |
|---|---|---|---|
| A1 Research | Frontier | ~96 | $3–8 |
| A2 Analyst | Fast/cheap | 200–2,000 | $2–10 |
| A3 Author | Frontier | ~1 (weekly) | <$1 |
| A4 Post-mortem | Frontier | 1–3 | $2–5 |
| A5 Supervisor | Fast/cheap, mostly rules | 1,440 | $1–3 |

**~$10–25/day, $300–750/month.** That is a serious line item against a small bankroll —
at L1/L2 it exceeds any plausible trading profit. Controls: aggressive prompt caching,
batch A2 assessments, rules-first with LLM only on ambiguity, and **run A1/A3/A4 only
from L3 upward**. At micro scale the system runs on rules alone, which is also the
cleanest way to measure what the agents actually add.

## 8.6 Measuring whether the agents earn their keep

Every agent gets an explicit A/B evaluation. The null hypothesis is that it adds nothing.

- **A2 Analyst:** compare outcomes on trades where the analyst score was high vs. low,
  holding all mechanical filters constant. If there's no separation, drop it and save
  the money.
- **A1 Research:** paper-trade S3 for 30 days alongside a random-token-basket control.
- **A4 Post-mortem:** track whether its proposed parameter changes, when adopted,
  improved forward performance. This is the honest test of whether "the AI is learning"
  means anything.

If an agent can't beat its control, it gets removed. "It feels smart" is not a metric,
and a system that keeps components on vibes is how the $300–750/month becomes permanent
for nothing.
