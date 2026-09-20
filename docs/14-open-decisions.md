# 14 — Open Decisions

Nine things I need from you before Phase 1 starts. Everything in this plan was written
under the stated assumptions; where your answer differs, the affected documents change.

---

### D1 — Total capital allocation *(blocks Phase 0)*

How much are you prepared to put at risk in total, understanding that losing all of it
is a live outcome and not a tail scenario?

*Plan assumes:* $10,000–20,000, released in rungs up the ladder, plus ~$1,000 of
separately-funded infrastructure budget.
*Changes if different:* under ~$3,000, fixed costs dominate and the honest framing is a
research project rather than a profit attempt. Over ~$50,000, we need an entity/tax
conversation and a bigger discussion about liquidity capacity.

---

### D2 — Drawdown tolerance *(blocks doc 06 finalization)*

What drawdown makes you shut this down — not intellectually, but actually?

*Plan assumes:* 25% hard stop with automatic halt.
*Why it matters:* this is the single most consequential number in the system. If your
real tolerance is 15%, position sizing has to halve and expected returns halve with it.
Answer honestly rather than aspirationally — a breaker set above your true tolerance
means you'll override it manually at the worst moment, which is strictly worse than
having set it correctly.

---

### D3 — Autonomy level *(blocks Phase 4)*

Fully autonomous from Phase 4, or approval-gated trades initially?

*Plan assumes:* fully autonomous within the risk engine's caps, starting at L1 micro size.
*Alternative:* a push notification with a 60-second approve window for the first 50
trades. Costs some fills, buys confidence. My recommendation is full autonomy at L1 —
$10 positions are precisely the right size to be wrong at, and approval-gating changes
the fill characteristics you're trying to measure.

---

### D4 — Chain scope

Solana only, or Solana + Base?

*Plan assumes:* Solana only for v1.
*Recommendation:* Solana only. Adding Base doubles the execution stack for a fraction of
the flow. Revisit at Phase 5.

---

### D5 — Repository *(blocks Phase 0)*

`tylerspink/tylerspink` is your **public profile repo** — its README renders on your
GitHub profile. This planning doc is fine there. The implementation should not be.

*Need:* confirmation to create a private repo (suggested: `meme-desk` or `trench`), and
whether this planning content moves there too or stays public.

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
| — | — | — | — |
