# 10 — Dashboard & Control Surface

Two jobs: tell you the truth quickly, and let you stop the machine from anywhere.

## 10.1 Design principle

The dashboard is a **monitoring and control** surface, not a trading terminal. You should
not be clicking "buy." If you find yourself wanting to override the bot's individual
trades, either the strategies are wrong (fix the config) or the discipline is slipping
(the thing the mechanical ladder in [doc 06](06-risk-engine.md) exists to prevent).

**The single most important element is the kill switch, and it must work when everything
else is broken** — it lives behind an independent path (a Telegram bot command and a
plain HTTP endpoint), not only inside the React app.

## 10.2 Screens

### Overview — the "am I okay?" screen, readable in 5 seconds
- Bankroll, today's P&L, 7d, 30d, all-time — **all net of fees**
- Current drawdown vs. the 25% breaker, as a filled gauge
- Deployed capital vs. the 20% cap
- System health: ingest lag, RPC status, signer status, last trade time
- Circuit breaker states — green/amber/red per breaker
- **KILL SWITCH** — always visible, top right, two-step confirm

### Positions
Open positions with live price, unrealized P&L, time held, distance to stop and to next
TP rung, the safety score *now* vs. at entry (divergence here is the early warning of a
developing rug), and a manual exit button per position.

### Trades
Full history. Each row expands into the complete audit chain: signal → features at
decision time → risk decision → quote → transaction → fill → exit. If a trade can't be
explained from this view, that's a bug in the logging, and the logging is the product.

### Strategies
Per-strategy: allocation, positions, expectancy (rolling 100), hit rate, payoff ratio,
drawdown vs. its budget, equity curve, enable/disable toggle, current config with
version history.

### Signals & Rejections
Every signal including untaken ones, with the rejection reason code and the
counterfactual P&L of what we passed on. This is the screen that tells you whether the
risk engine is protecting you or strangling you, and it's the one most likely to be
undervalued and most likely to change a decision.

### Agents
Recent agent runs, outputs, cost, and the A/B evaluation scorecard from
[doc 08](08-agent-design.md) §8.6 — is each agent beating its control?

### Wallets (smart-money)
The tracked wallet universe, scores, recent activity, and which wallets our S1 entries
came from. Manual add/remove/blacklist.

### System
Latency histograms, transaction success rate, fee spend, data quality monitors, error
log, LLM spend vs. cap.

## 10.3 Alerting — separate from the dashboard

Push (Telegram bot or ntfy), because you won't be watching a screen:

**Immediate:** any circuit breaker fires · unrecognized transaction from the hot wallet ·
signer unreachable · a held token's safety degrades · drawdown crosses 15% (warning) and
20% (urgent) · transaction failure rate spike.

**Digest (daily):** P&L, trades, best/worst, strategy attribution, post-mortem summary,
cost vs. gross profit.

**Deliberately not alerted:** individual trade entries and exits. At 10–30 trades/day
that's noise, and noise trains you to ignore the channel that carries the urgent alerts.

## 10.4 Controls

| Control | Effect | Confirmation |
|---|---|---|
| **KILL** | Halt all entries, signer refuses new buys, exits continue | Two-step |
| **PANIC EXIT** | Market-sell everything to SOL, then halt | Two-step + typed phrase |
| Pause strategy | Suspend one strategy, existing positions unaffected | One-click |
| Exit position | Market-sell one position | One-click |
| Set regime multiplier | Global sizing scale 0.3–1.0 | One-click |
| Edit strategy config | Opens a PR against the config repo | Review + merge |
| Change a risk limit | **Not available in the UI.** Code change, review, deploy. | — |

The last row is intentional. Risk limits being annoying to change is a feature — it's
what stops a 2am "just this once" from becoming the loss that ends the project.

## 10.5 Build order

Phase 1 gets a read-only overview and system health. Phase 3 adds positions, trades and
the kill switch. Phase 4 adds strategies, signals and controls. Phase 5 adds agents,
wallets and the analytics depth. The dashboard should never be ahead of the machinery it
displays.
