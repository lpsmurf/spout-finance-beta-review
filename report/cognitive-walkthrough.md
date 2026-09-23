# Cognitive Walkthrough Report — Spout Finance Beta

**Task**: Buy a tokenized stock, deposit it as collateral, and borrow USDC at 0%
**User Persona**: DeFi-native first-time Spout user — has Phantom, testnet USDC, completed beta-gate + Persona KYC. Understands wallets/signatures; does NOT know covered-call collateral mechanics, LTV, or what "assignment" means.
**Interface**: beta.spout.finance (Next.js SPA, Privy auth, devnet)
**Date**: 2026-09-21 — evidence: live DOM inspection + signed-tx attempts + vault/order API probes (spout-retest-2026-09-21.md)

---

## Executive Summary

**Estimated first-attempt success rate: ~0%** — not because of UX, but because the task is currently impossible: the oracle keeper is down and every on-chain step reverts. Evaluated *as if the keeper were healthy*, the flow is well-designed until the point of commitment — then the two things a novice most needs to understand (leverage, assignment risk) are the least explained.

### Critical findings
1. **No state in which the task can complete** — market-open + stale oracle = every tx reverts; the app never communicates this (the one true blocker is invisible).
2. **The 1.0x–2.0x leverage slider on Buy performs a borrow it doesn't explain** — the panel does show an "Amount / Stocks borrowed" line and "Est. borrower cost/yr", but the slider never says "debt"/"loan"/"collateral" and shows no Health Factor. A user buying "just some stock" at 1.5x has opened a 0% loan they may never have intended (or vice versa: a user who wants to borrow may never find the Borrow tab, since buy-with-leverage accomplishes it invisibly).
3. **Nothing anywhere in the flow discloses covered-call assignment** — the mechanic that can sell your collateral is absent from buy, borrow, and confirmation surfaces.

---

## Step-by-step walkthrough (as-if-healthy path)

### Action 1: Reach the app
- Gate: beta code + Privy login + Persona KYC. Verified: `spout-gate` email+passcode gate, then wallet connect. Q1–Q4 all ✅ for the target persona; the passcode gate adds friction but bounty users arrive with a code. **Pass.**

### Action 2: Understand where to start
- Landing = Trade tab with an instrument table. "Trade tokenized stocks — Buy/Sell stocks, then borrow against them at 0% interest." ✅ Q1: the tagline literally narrates the task. Q2: table + buy panel visible above fold ✅. Q4: "$10 USDC" balance chip confirms connection/funding state ✅.
- ⚠️ "Market: Closed" pill + countdown is good ambient status — but it does NOT disable or annotate the Buy button (verified: fully enabled). User's mental model: "market closed" might mean orders queue — actually it means txs revert. **H1/H5 issue.**

### Action 3: Choose an asset
- Table: name, market cap, price, borrow cost %/yr, 30D chart, shares owned, View. ✅ Good density; "Borrow Cost" column primes the second half of the task early — genuinely smart IA.
- ❌ Every row renders the Pfizer logo — a novice can't visually distinguish tickers; a 2-second wrong-asset purchase is one click away.

### Action 4: Enter the buy
- Buy panel: USD input, Max, **Leverage slider (1.0/1.25/1.5/2.0x)**, "Est. borrower cost/yr", "Your cost today", big "Buy X TICKER" button.
- ❌ Q3 failure: what does "Leverage 1.5x" mean to a novice here? The slider implies "buy more stock" — the mechanism is "borrow USDC against the stock you're buying." The panel never says *loan*, *collateral*, or *debt*. A novice can slide to 2.0x and create a loan without ever seeing the word "borrow."
- ⚠️ Q4: "Est. borrower cost/yr $0.00" + "Your cost today" is real-time feedback ✅, but no Health Factor / liquidation preview *at the leverage decision point* — it exists on the Borrow page, not here.

### Action 5: Sign
- Phantom popup via their wallet adapter (custom title/description verified working). ✅ Signing is the only informed-consent moment — and the consent copy describes the purchase, not the loan.

### Action 6: Confirm fill
- Orders are async (pendingOrder account + submit). ⚠️ The global `/api/orders` feed is the only visible "order book" — portfolio shows "No Holdings" until fill. Expect "did it work?" confusion between sign and fill.

### Action 7: Borrow against it
- Borrow tab: "Borrow against X" panel — Amount, LTV slider capped at 50%, Health Factor readout, Active Positions list. ✅ The 50% cap is enforced in the UI (matches on-chain collateralRate). ✅ Blockers render in plain language ("You don't own any XOM yet. Buy some on Trade…").
- ❌ The same Health Factor that was missing at buy-time appears here — asymmetric risk disclosure: leverage during buy hides it, borrow surfaces it.
- ❌ "No interest, no margin calls" — false reassurance at exactly the decision point (liquidation at 65–70% LTV exists).

### Action 8–9: Ongoing position management
- autoBuyback (Auto-Roll) exists per-vault but invisible until a position exists; no account-level setting. ⚠️ Defaults unknown — but a taxable-event-generating auto-mechanic should be a *choice*, not a discovery.

---

## Failure points

| # | Point | Severity | Fix |
|---|-------|----------|-----|
| 1 | Task impossible while oracle stale; app shows "Market: Open" + enabled Buy | **P0** | Global degraded banner + disable signing paths when `priceStale` |
| 2 | Leverage slider performs a loan; "borrowed" appears only as a small line item — no "debt/loan/collateral" wording, no HF at the decision point | Sev 3 | Rename to "Borrow to buy" mode or require explicit opt-in; show HF inline |
| 3 | Assignment/auto-buyback undisclosed anywhere in the flow | Sev 4 | One-time acknowledgment + cycle-status on position card |
| 4 | "Market: Closed" informs but doesn't prevent doomed signatures | Sev 3 | Disable Buy or gate it behind "queue for open" copy |
| 5 | Raw `0x177d` / "Blockhash not found" reach the user unmapped | Sev 3 | Error-map layer (trade path especially) |
| 6 | Pfizer logo on every row; phantom +$0.60 P&L on empty portfolio | Sev 1–2 | Fix logo source; hide P&L when cost basis is 0 |

## What works (credit)
- Plain-language blocker system on the vault side is genuinely best-in-class — reason + remedy for every blocked action.
- LTV cap enforced in UI consistent with on-chain.
- PoR line ("Held 1:1 at Alpaca · Reserves 100.2%") inline at the buy panel — trust signal at the right moment.
- KYC state legible in Settings ("Verified on-chain • identity account found").

## Success likelihood (if keeper healthy)
| User | Est. success | Note |
|------|-------------|------|
| DeFi novice | ~55% | Leverage slider ambiguity + assignment surprise are the risks |
| DeFi intermediate | ~85% | Will figure out leverage; still blindsided by assignment |
| Target bounty reviewer | ~95% | Knows to look |

**Recommendation priority:** (1) oracle-freshness as a global UI state — not a per-endpoint detail; (2) make the buy-time leverage explicit about being a loan; (3) assignment disclosure before first borrow.
