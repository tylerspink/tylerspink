# 14 — Open Decisions

Five open, four answered. Everything in this plan was written under the stated
assumptions; where an answer differs, the affected documents change.

The four answered on 2026-09-20 all matched the plan's assumptions, so no document
required revision. They are recorded in [Answered decisions](#answered-decisions) below.

---

### D4 — Chain scope

Solana only, or Solana + Base?

*Plan assumes:* Solana only for v1.
*Recommendation:* Solana only. Adding Base doubles the execution stack for a fraction of
the flow. Revisit at Phase 5.

---

### D6 — Custody setup *(blocks Phase 0)*

Do you have a hardware wallet? Willing to set up a Squads multisig for the treasury tier?

*Plan assumes:* yes to both. The three-tier model in [doc 07](07-custody-and-security.md)
is what bounds a total compromise to ~15% of bankroll. Without the treasury tier, the hot
wallet holds everything and a single compromise is terminal.

---

### D7 — Jurisdiction & tax posture

US? Do you have a CPA who handles crypto?

*Plan assumes:* US, no CPA engaged yet. The cost-basis tracking requirement in
[doc 13](13-legal-tax-compliance.md) is an engineering dependency that has to be built in
Phase 3, not retrofitted — so I need this answer before then either way.

---

### D8 — Social data budget

X/Twitter API access is the expensive input ($100–250+/mo) and it's the primary feed for
S3 (Narrative Rotation).

*Options:* (a) skip S3 for now and defer the cost — my recommendation for Phases 1–4;
(b) budget for it from the start; (c) scrape-based alternatives, which are cheaper,
fragile, and against ToS.

---

### D9 — Your involvement

Realistically, how much time per week for the weekly review, monthly allocation decisions,
and responding when a breaker fires?

*Plan assumes:* 2–4 hours/week in steady state.
*Why it matters:* the scale ladder depends on a human reading the evidence at each rung.
If that review won't happen, the ladder is decorative and we should design something
simpler with tighter automatic limits and no discretionary scaling at all.

---

## Answered decisions

Recorded here as they're settled, and promoted to ADRs in [`adr/`](adr/) when they have
architectural consequences.

| # | Decision | Answer | Date |
|---|---|---|---|
| D1 | Total capital allocation | **$10,000–20,000**, released in rungs up the ladder, plus ~$1,000 separately-funded infrastructure budget. Matches the plan; the ladder in [doc 06](06-risk-engine.md) §6.5 can reach L4. | 2026-09-20 |
| D2 | Drawdown tolerance | **25%** peak-to-trough, automatic halt, manual restart. Confirms the hard limit in [doc 06](06-risk-engine.md) §6.3. Note §6.7: this breaker is expected to fire eventually even if the edge is real. | 2026-09-20 |
| D3 | Autonomy level | **Full autonomy from Phase 4 (L1)**, within the risk engine's caps. No per-trade approval gate. | 2026-09-20 |
| D5 | Repository | **New private repository**, with this planning content moved there too. Implementation does not live on the public profile repo. | 2026-09-20 |
