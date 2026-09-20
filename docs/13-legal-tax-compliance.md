# 13 — Legal, Tax & Compliance

**Not legal or tax advice.** This flags what needs professional input and what the
system must record to make that input cheap. Get a crypto-literate CPA before Phase 4,
not after.

## 13.1 Tax — the engineering requirement hiding in a legal section

In the US, crypto is property. **Every swap is a taxable disposal**, including
token→token. A bot doing 10–30 trades/day generates 3,000–10,000+ taxable events per
year. Reconstructing that after the fact from block explorers is a nightmare that costs
real money in accountant hours.

**Therefore, from the very first live trade, the system records per fill:**

- Timestamp (UTC), transaction signature
- Asset in / out, exact amounts
- **USD value of both legs at execution time** (priced at fill, not at end of day)
- Fees paid, in SOL and in USD at the time
- The wallet involved
- A stable lot identifier linking each disposal to its acquisition

And exports in a format Koinly / CoinTracker / TokenTax ingest directly.

Other points to raise with the CPA:
- **Wash sale rules currently do not apply** to crypto as property in the US, though
  legislative proposals to change that recur. Confirm current status before relying on it.
- **Trader tax status (TTS) / §475(f) mark-to-market election** may be advantageous at
  volume, but has strict requirements and an early filing deadline. Ask about this before
  year-end, not in April.
- Estimated quarterly payments may be required once gains are material.
- Entity structure (LLC) — generally not worth it at small scale; worth asking about at $50k+.

**Reserve for taxes as you go.** Owing tax on realized gains you've since lost trading is
a well-documented way people end up in real trouble.

## 13.2 What's fine

Trading your own capital with your own automated system is ordinary activity. Bots are
not special. No licensing implication for personal trading.

## 13.3 What is not fine — hard boundaries for this project

These are out of scope and will not be built:

- **Managing anyone else's money.** Accepting outside capital moves this into investment
  adviser / commodity pool territory with registration requirements. If that's ever a
  goal, it's a lawyer conversation first, not a feature request.
- **Market manipulation.** Wash trading to fake volume, spoofing, coordinated pump
  schemes, insider launches where you or an affiliate deploy the token and trade against
  buyers. This is the category regulators actually prosecute in crypto, and it overlaps
  with where a meaningful share of "smart money" profit comes from — which is exactly why
  [doc 05](05-strategy-portfolio.md) explicitly excludes it and why the S1 wallet filter
  screens insider wallets *out* rather than following them.
- **Sanctions exposure.** Don't interact with OFAC-listed addresses. Impractical to screen
  every AMM counterparty, but we can and should screen deployer wallets against public
  lists and skip flagged tokens.
- **Front-running / sandwiching other users.** Excluded in doc 05 on both practical and
  ethical grounds.

## 13.4 Jurisdiction

This assumes US. If you're elsewhere, several things change materially — tax treatment,
whether derivatives are accessible, reporting obligations. Flag it in
[doc 14](14-open-decisions.md) if so.

## 13.5 Record retention

Keep for at least 7 years: all trade records, the tape archive (or at least the trade
subset), wallet addresses used, and cost-basis exports. Cheap insurance, and if there's
ever a question it's the difference between an afternoon and a very bad month.
