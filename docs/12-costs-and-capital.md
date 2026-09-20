# 12 — Costs & Capital

## 12.1 Monthly operating cost

| Item | Lean (Phase 1–4) | Full (Phase 5+) |
|---|---|---|
| Geyser/RPC (Helius or equivalent) | $49 (dev tier) | $499 (business tier) |
| Backup RPC provider | $0 (free tier) | $50 |
| VPS (16 vCPU / 32GB / NVMe) | $60 | $120 |
| Object storage (tape) | $5 | $20 |
| Data APIs (Birdeye / RugCheck) | $0–50 | $100–200 |
| Social data (X API is the expensive one) | $0 | $100–250 |
| LLM inference | $0–50 | $300–750 |
| Monitoring / errors / secrets | $0–25 | $50 |
| **Total** | **~$120–240/mo** | **~$1,250–1,940/mo** |

## 12.2 The hurdle rate — the number that decides scale

Infrastructure is a fixed cost. It becomes a percentage hurdle against the bankroll, and
at small size that percentage is disqualifying:

| Bankroll | Lean burn | Monthly return needed to break even |
|---|---|---|
| $250 | $120 | **48%** |
| $1,000 | $120 | **12%** |
| $5,000 | $150 | **3.0%** |
| $15,000 | $240 | **1.6%** |
| $40,000 | $1,400 (full stack) | **3.5%** |

Three things follow directly:

1. **At L1 and L2 we are paying tuition, not running a business.** That's fine and it's
   the plan — those phases buy information about fills and slippage, not profit. It just
   needs saying out loud so a negative month at $250 isn't mistaken for failure.
2. **Stay on the lean stack until L3.** Do not buy a $499 Geyser tier to trade $1,000.
   Upgrade only when measured latency is demonstrably costing more than the upgrade.
3. **The full stack only makes sense at $40k+.** Below that, the agent layer and premium
   data are a luxury. This is why [doc 08](08-agent-design.md) §8.5 defers A1/A3/A4
   until L3.

**Total spend to reach the Phase 4 decision point: roughly $700–1,200** (about 4 months
of lean infra plus the $250 test bankroll). That is the real price of finding out
whether this works.

## 12.3 Capital plan

**The rule:** only fund this with money you would be genuinely fine losing entirely.
Not "would be annoyed to lose" — fine. The base rates in
[doc 02](02-market-reality.md) make total loss of the deployed bankroll a live outcome,
not a tail scenario.

Suggested structure:

```
Total allocation to this project:  $X  (you decide; see doc 14)

  Cold storage / not at risk       ──  keep outside the project entirely
  Treasury multisig                ──  ~85% of X  (funds the ladder over time)
  Hot wallet                       ──  ~15% of X  (max loss from compromise)
  Infra budget, 6 months, separate ──  ~$1,000   (do not fund this from trading P&L)
```

Fund the infra budget separately and up front. A system that has to earn its own hosting
in month one will be pushed to take bad trades, which is exactly the failure mode the
risk engine exists to prevent.

**Starting bankroll recommendation:** the ladder starts at $250 regardless of what X is,
so X mainly determines how far the ladder can go. For the ladder to reach L3–L4 — where
position sizes finally clear the fixed-cost drag ([doc 02](02-market-reality.md) §2.3) —
**X in the $10,000–20,000 range** is the natural target, released in rungs. If X is
smaller than about $3,000, the honest read is that fixed costs will dominate and the
project is better run as a research exercise than a profit attempt.

## 12.4 Profit handling

- Profits sweep automatically from hot → treasury above a threshold (one-way, hardcoded
  destination — [doc 07](07-custody-and-security.md) §7.1)
- **Withdraw a fixed fraction of realized profits out of the system**, e.g. 25% quarterly
  to cold storage. Reinvesting 100% means the strategy eventually finds the size at which
  it stops working, and gives it all back. Taking chips off the table is a policy, not a
  mood.
- Reserve for taxes from realized gains as you go — see [doc 13](13-legal-tax-compliance.md).
  This is not optional and the amount is not small.

## 12.5 Time cost

Worth pricing honestly alongside the dollars. Phases 1–4 are roughly 8–12 weeks of
substantive build work, and steady state still requires a weekly review, a monthly
allocation decision, and responding when a circuit breaker fires. This is not a system
that runs unattended for a quarter. If nobody is going to do the weekly review, the
system will decay silently — and the scale ladder, which depends on a human reading the
evidence at each rung, stops functioning.
