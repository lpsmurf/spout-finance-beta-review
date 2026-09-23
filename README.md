# Spout Finance — Beta Product & Security Review

Independent product-feedback submission for the
[Spout Finance bounty](https://superteam.fun/earn/listing/product-feedback-spout-finance)
(Superteam Earn).

Tested 2026-09-19 → 2026-09-23 on Solana devnet with a real Phantom wallet,
real signed transactions, and direct probing of every public API surface.

## Read in this order

1. **[ARTICLE.md](ARTICLE.md)** — the long-form public write-up.
   *"I stress-tested Spout Finance's beta for four days. The engine is clever —
   and I have a receipt for $4 of stock that was never delivered."*
2. **[report/teardown.md](report/teardown.md)** — the full structured
   teardown: mechanism analysis, tax/disclosure contradiction, tranche math,
   on-chain program verification, risk analysis, UX audit, full findings table.
3. **[report/cognitive-walkthrough.md](report/cognitive-walkthrough.md)** —
   novice-user walkthrough of the buy→borrow task (UX evidence).
4. **[report/testing-log.md](report/testing-log.md)**,
   **[report/retest-2026-09-21.md](report/retest-2026-09-21.md)**,
   **[report/retest-2026-09-22.md](report/retest-2026-09-22.md)** — methodology
   and day-by-day executed-transaction evidence, including the ~53-hour oracle
   keeper outage, three signed transactions reverting `OraclePriceStale (6013)`,
   and the subsequent successful `$4 GOOG` buy (`PlaceBuyOrder` confirmed
   on-chain, order still pending 18h later at time of writing).
5. **[screenshots/](screenshots/)** — UI captures (buy panel incl. leverage slider + declined-signature toast, the "You own 0.01 GOOG / fills at 9:30 AM ET, 22 Sep" confirmation modal, borrow page, settings/KYC) and annotated live-API evidence captures (global orders feed, our 34h-pending order, the $1-limit-price acceptance, the unauthenticated KYC endpoint).

## Headline findings

- **Reliability:** a single silent oracle-keeper outage bricked every on-chain
  action for ~53 hours while the UI showed "Market: Open" and live prices.
- **Fulfillment:** a signed, on-chain-confirmed buy did not deliver tokens
  for 18+ hours; the pending-order backlog grew 145 → 268 in a day.
  No status, no ETA, and canceling your own stuck order costs a fee.
- **Disclosure:** "never trigger a taxable sale" contradicts the covered-call
  assignment mechanic documented in their own docs.
- **Security:** signature boundary holds (nothing funds-reachable), but several
  read endpoints disclose more than they should. Sensitive items and a
  mainnet-readiness question were disclosed to the team privately.

Independent testing. Not investment advice.
