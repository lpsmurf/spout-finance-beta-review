# I stress-tested Spout Finance's beta for four days. The engine is clever — and I have a receipt for $4 of stock that was never delivered.

*Product-feedback submission for the Spout Finance bounty. Tested 2026-09-19→23 on devnet with a real Phantom wallet, real signatures, and every preflight endpoint. Everything below is verified on-chain or against the live API.*

Spout's pitch is the kind of thing that makes a DeFi person sit up: **deposit tokenized US equities, borrow stablecoins at 0% interest, up to 50% LTV** — and the reason it can be 0% is that your collateral earns a covered-call premium on the traditional options market. Lenders fund the pool in two tranches (Senior ~9%, Junior ~32%) and get paid out of the same volatility risk premium that subsidizes the borrower's zero rate. It's a real, coherent piece of financial engineering, and the docs are honest about where the risk sits.

I spent the weekend inside the beta. Here's what I found — the good, the structural, and the thing that's actually broken right now.

## The mechanic, honestly assessed

The covered-call funding loop is the interesting part. Tokenized shares sit 1:1 at Alpaca Securities (the app shows "Reserves 100.2%" inline — nice touch), a systematic options desk writes calls on the collateral pool, and the premium flows back on-chain to pay lenders and to make borrowing free. Losses cascade insurance fund → Junior → Senior. That ordering is disclosed plainly, which already puts Spout ahead of most DeFi docs.

Three structural notes worth putting in writing:

**1. "Never trigger a taxable sale" vs. the assignment mechanic.** The front page says you keep every share and never realize a sale. The docs say covered calls can finish in-the-money, at which point *your shares are sold at the strike*, proceeds cover your debt, and the remainder is yours. Both can't be true. Assignment is a sale — a taxable event — and Auto-Roll/auto-buyback rebuys the asset as a new tax lot. The fix isn't the mechanic (covered calls need assignment risk to earn premium); it's that the promise needs to move from "never" to "here's exactly when."

**2. The yield is a levered short-vol position wearing a savings-rate costume.** Junior earns ~32% because it absorbs pool losses first, above a thin insurance fund ($50–100k seed vs a $200k target — the docs disagree with themselves). That's a leveraged short-volatility book, and its worst case is precisely the scenario backtests underweight. The 30%/week premium assumption behind the headline APYs is healthy-vol; in a quiet tape, Senior's 7% priority eats nearly everything and Junior's yield collapses toward single digits. Honest framing would show the distribution, not the point estimate.

**3. The market-hours oracle gap.** US equities price 9:30–4 ET weekdays; the liquidation engine runs 24/7. I verified what that means in practice, which brings me to…

## The weekend the whole product went down — and nobody could tell

Sunday into Monday I signed a real $2 buy through Phantom. The server simulated it and refused: `custom program error 0x177d` — `OraclePriceStale`. Every feed for all ten instruments was ~52 hours stale against a 24-hour limit. The borrow and withdraw paths were hard-blocked too. As a KYC-verified, funded user I could not execute a single on-chain action: can't buy (oracle), can't deposit (needs tokens you can't buy), can't open a vault (needs a deposit), can't repay or withdraw (needs a vault).

Then Monday came. The market opened. The app's own price API went live — real-time quotes, the banner flipped to "Market: Open," the buy button sat there enabled. And a second signed transaction still reverted `OraclePriceStale`. The on-chain keeper is simply down; the UI presents a fully healthy product over a chain layer where literally nothing works.

This is the single most important finding of the test — not because it's catastrophic (it's a devnet beta, that's what betas are for) but because of what it reveals:

- **Preflight coverage is asymmetric.** The vault API checks oracle freshness and returns beautiful plain-language blockers ("The GLD price is 188,610s old — borrow and withdraw revert until a keeper pushes a fresh price"). The trade API checks *nothing* — it returned `blockers: []` for a transaction that cannot succeed, let me sign it, and then surfaced a raw Anchor error.
- **There's no degraded state.** "Market: Closed" exists as a pill with a countdown, but it's decorative — it doesn't gate anything, and "Market: Open" means "the stock market is open," not "Spout can execute."
- **The failure mode is silent.** If this keeper stalls on mainnet during a drawdown, you don't just have a UX problem — you have positions that can't be liquidated *or* defended.

## Then the keeper came back — and I finally bought a stock

Tuesday the keeper recovered: all ten feeds fresh (`priceStale: false`, ages under five minutes). I returned to the buy form and did it for real: **$4 USDC → GOOG**. Phantom popped, I signed, and the transaction confirmed on-chain: `PlaceBuyOrder … Stork price: 353.314 (staleness: 151s) … Buy order placed: 4000000 USDC … success`. My USDC moved: balance 10 → 6, escrowed to an on-chain order account owned by the orders program.

And then — nothing. The orders pipeline is asynchronous: the program places the order and an off-chain fulfiller is supposed to deliver the tokenized shares. **Eighteen hours later, through an entire market session, my order is still pending.** The unauthenticated order feed (itself a privacy finding) shows my order sitting there alongside a backlog that grew from 145 to 268 pending orders in a day — nothing appears to be clearing. GOOG's price has since moved 353.31 → 335.69; my fill, if it ever comes, is locked at yesterday's price.

What the user experiences in that window: money gone, no position, no pending-state UI worth the name, no ETA, no status field anywhere in the API — and if you want out, canceling your own stuck order costs a fee (`cancellationFeeUsdc: 0.004`). That's recoverable-poor-UX on a good day; on a bad day it's "user funds held in limbo by a stalled back-end nobody monitors."

## The UX is closer than it looks

Nielsen-style pass plus a cognitive walkthrough of the buy→borrow task (full detail in my report). What stood out:

**Genuinely good:** the vault preflight layer — every blocked action comes with a reason and a remedy in plain English; the LTV slider caps at 50% consistent with on-chain parameters; Health Factor is surfaced on the borrow panel; proof-of-reserves is a first-class line item at the point of purchase; KYC status is legible ("Verified on-chain • identity account found"). This is a team that knows how to communicate state — when it chooses to.

**Where it breaks:**
- **The leverage slider is an invisible loan.** Buy panel offers 1.0x–2.0x leverage without ever saying "borrow," "debt," or "collateral." Sliding to 1.5x opens the vault-and-borrow flow inside a purchase. Meanwhile Health Factor — the number that tells you how close you are to liquidation — appears on the Borrow page but not at the leverage decision point where it's actually needed.
- **"No interest, no margin calls."** The first is true to the letter. The second is marketing over a liquidation engine that *is* a margin call — just one that executes without calling you first. At 65–70% liquidation ratio, "no margin calls" reads as "no risk of forced sale," which is precisely wrong.
- **Assignment is the product's biggest surprise and its least-visible fact.** Nowhere in buy, borrow, confirm, or settings does the app say your collateral is being covered-called and can be sold at strike this cycle. The `autoBuyback` toggle exists per-vault but is unreachable until you have a position.
- **Limit orders have no sanity check.** A `limitPrice` of $1 on a $353 asset sails through preflight with zero blockers — funds lock into a permanently unfillable order, and exit costs the cancel fee. One "your limit is >90% off market" warning would kill this footgun.
- Small polish debts: every instrument renders the Pfizer logo; an empty portfolio shows "+$0.60 (+6.04%) P&L"; the on-chain allowlist accepts AAPL but the instruments API doesn't list it (UI/catalog and chain disagree); no rate limiting anywhere — fifteen rapid transaction-build calls all succeed.

## Security posture (briefly)

I'll keep this short because it's mostly good: tight CSP, `XFO: DENY`, nosniff, Supabase RLS holding on every user table I probed, server-side simulation before relaying signatures. Beta needs HSTS, and a few read endpoints disclose more state than they should (order book, per-wallet KYC status via a side door, wallet-scoped tx building for arbitrary addresses — details sent to the team privately). Nothing funds-reachable; the signature boundary holds.

## Bottom line

Spout is one of the more intellectually honest DeFi products I've tested — the docs admit the risk waterfall, the beta admits Earn isn't live yet, the vault tells you exactly why you can't do things. What's missing is the same honesty in the places users actually look: the buy button doesn't know the market's broken, the order pipeline takes your money without telling you when or if it comes back, the leverage slider doesn't say it's a loan, the headline doesn't mention assignment, and "no margin calls" papers over the mechanism that makes 0% possible.

The engine is real. Ship the transparency that already exists in the API layer into the UI, put monitoring on the keeper *and* the fulfiller before mainnet, and reconcile "never a taxable sale" with the mechanic that occasionally sells your shares — and this is a defensible product.

---

*Testing wallet: GfcpRxboaev5cHDLUR86t1Bh5c5TtGGCP2pQFQKf9KGv (devnet). All findings verified with signed transactions and live API evidence. Sensitive items disclosed privately.*
