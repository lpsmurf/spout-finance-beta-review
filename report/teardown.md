# Spout Finance: A Beta Teardown — What "Borrow Like a Billionaire" Actually Costs

> **Full structured report** — mechanism analysis, claims-vs-mechanics review, on-chain verification, risk analysis, UX audit, and the complete findings table. Doc analysis verified against Spout's public docs (2026-09-09) and live beta behavior (2026-09-19 → 2026-09-23). Where a question could only be answered from inside Spout's infrastructure, it is marked **[open — needs team clarification]** rather than guessed at.

---

## TL;DR

Spout is a genuinely interesting piece of financial engineering: borrow stablecoins at 0% against tokenized US equities, funded by a systematic covered-call strategy that also pays lenders 9–32%. The mechanism is real and the docs are unusually honest about the risk waterfall. But three things deserve scrutiny before launch:

1. **The headline promise and the mechanics contradict each other on the single most important claim** — the taxable event.
2. **The lender yield is a levered short-volatility position**, and the "Senior never loses" case rests on backtests of a strategy whose whole risk *is* the event backtests miss.
3. **The core loop was unexercisable all weekend — verified with a signed transaction.** Every oracle feed went >52h stale (24h limit), so buys reverted `OraclePriceStale` (0x177d) on-chain and borrow/withdraw were hard-blocked. A KYC'd, funded beta user could not complete a single on-chain action. The market-hours oracle concern (Part 6) isn't theoretical — it's a live availability bug.

Full analysis, UX findings, and bugs below.

---

## Part 1 — The model, stated fairly

Credit where due — the design is coherent:

- Borrowers post tokenized equity collateral (**spAssets**: Token-2022, 1:1 broker-custodied, on-chain Proof of Reserve, KYC transfer hook) and draw stablecoins at up to **50% LTV, 0% interest**.
- Lenders fund the stablecoin pool in two tranches: **Senior** (85%, ~9% APY, 7% priority + 25% of excess) and **Junior** (15%, ~32% APY, first-loss above the insurance fund, 45-day withdrawal notice).
- A continuous **covered-call strategy** on the collateral pool harvests the **Volatility Risk Premium** — the persistent gap between implied and realized vol. That single premium stream pays lenders *and* funds the borrower's 0%.
- Losses flow: **Insurance Fund → Junior → Senior**. Per-asset circuit breakers pause cycles if the fund draws down.

It's a clean two-sided design. The critique isn't that it doesn't work — it's about *where the risk actually sits* and *whether the marketing tells you*.

---

## Part 2 — The headline contradicts the mechanics (the big one)

The homepage promise:

> *"Take a loan against your stocks at 0% interest… you stay long, stay liquid, and **never trigger a taxable sale**… **You keep every share you started with**."*

The `options-assignment` doc:

> *"there is always some chance in a given cycle that the calls finish in the money and **the shares are sold at the strike**… The proceeds first cover any outstanding debt you have. The remainder is yours."*

These cannot both be true. When a covered call written on your collateral finishes in-the-money:

- **Your shares are sold.** For a US taxpayer, selling appreciated stock is a **realized capital gain — a taxable event.** The exact thing the headline says never happens.
- **You do not "keep every share."** Auto-Roll *rebuys* the asset — but that's a **new tax lot with a reset cost basis**, not the shares you started with.
- The upside beyond the strike is **capped** for that cycle.

Spout's docs try to pre-empt this in `what-borrowers-should-know`:

> *"The protocol does not introduce any new failure mode that does not already exist for someone who simply holds the underlying share."*

That claim is **incorrect**. Someone who simply *holds* a share never has it called away, never realizes a forced gain, and never has upside capped at a strike. Writing covered calls on the collateral introduces all three. The "no new failure mode" framing understates the real trade the borrower is making.

**This isn't a reason the product is broken — covered-call borrowing is legitimate.** It's a *disclosure* problem: the single most prominent marketing claim ("never a taxable sale," "keep every share") is contradicted by the core mechanic, and a borrower who believes the headline will be surprised by their first ITM assignment. **Verified in the live beta (2026-09-21):** assignment is disclosed nowhere in the decision path — not on the buy panel, not on the borrow panel, not in transaction confirmation, and not in Settings. The `autoBuyback` toggle (the on-chain Auto-Roll setting) exists per-vault but is unreachable until a position exists. A borrower's first encounter with forced sale will be the sale itself. This is the finding.

*Recommendation:* surface assignment + tax consequence at the borrow screen and before each cycle; soften the homepage tax claim to something defensible ("defer a sale while you hold" rather than "never trigger a taxable sale"); add a "not tax advice" note adjacent to the headline claim, not just buried in the risk page.

---

## Part 2.5 — The headline APYs rest on an optimistic premium assumption (verified from their own example)

Their `lending-tranches` walkthrough is **arithmetically correct** — I recomputed it: $30k gross weekly premium on a $10m pool → 20% fee, Senior 7% priority, 25/75 excess split → Senior ~8.9% / Junior ~32.6% APY. The math holds; credit for that.

But work backwards from the input: **$30k/week is $1.56M/year, or ~15.6% annualized gross premium on the pool** (~7.8%/yr if the collateral base is ~2× the lending pool, as 50% LTV implies). That is a *healthy-vol* assumption. Systematic covered-call programs capture that in normal-to-elevated volatility; in a low-vol grind, gross premium can run materially lower — and the structure is unforgiving when it does: **Senior's 7% priority is paid first off the top**, so if realized premium compresses toward ~7–8% of pool, Senior's floor consumes nearly everything and **Junior's ~32% collapses toward single digits**. The advertised spread between tranches only exists in the good-premium regime.

*Recommendation:* publish the APYs as a function of premium capture (a sensitivity table), not a single headline; show what Junior earns in a low-vol year, not just the $30k/week base case.

## Part 5.6 — The binding document and the docs quote different fees ⚠️ (verified live 2026-09-22)

The `docs/fee-structure` page and the legal [Terms of Service](https://spout.finance/terms) §8.1 publish **different fee schedules — and the Terms are worse on every line**:

| Fee | `docs/fee-structure` | `terms` §8.1 |
|---|---|---|
| Mint/redeem (buy/sell) | 0.20% (20 bps) | **0.25%** (25 bps) |
| Withdrawal | 0.20% (20 bps) | **1%** — 5× higher |
| Liquidation | ~4%–12.5% buffer | **7.5%–20%** buffer spread |
| Idle-routing fee | *not mentioned anywhere in the docs* | **~7% on yield** |

Same class of bug as Part 5's insurance-fund numbers, except this time one of the documents is *the contract*. If the Terms govern, the docs understate every fee a user is quoted; if the docs govern, the Terms misstate the legal fee schedule. And a ~7% yield-routing fee disclosed only in the ToS sits awkwardly next to "no hidden fees" on the front page. *Recommendation:* one canonical fee schedule, docs generated from it, and a stated order of precedence on conflict.

## Part 2.7 — On-chain verification: the compliance model isn't what the docs describe ⭐

I pulled the actual devnet spAsset mint (`B3S7TuCmBPsNJW32SzRLBwbmWmgNzFBuHnBG9rdvt6K5`, Token-2022) and read its extensions directly from chain. Two things the UI-only tester never sees:

**1. It's not a transfer hook.** The `security-and-compliance` doc says spAssets *"use Solana's Token-2022 standard with a transfer hook that enforces wallet-level KYC."* The mint has **no `transferHook` extension.** KYC is actually enforced by:
- `defaultAccountState: frozen` — every new token account is **frozen by default**; the freeze authority (`9D197p…`) must thaw it, which is where the KYC gate really lives.
- a **`permanentDelegate`** (`Bs8qi…`) — an authority that can move or burn **any holder's** spAssets unconditionally.

That's a factual doc error (default-frozen + permanent delegate ≠ transfer hook — different mechanisms, different properties), and it's the kind of thing that matters for anyone integrating spAssets.

**Proven live.** I built a page that connects a wallet holding real devnet spAssets (balance `0.031542255`, mint `B3S7…`) and attempts to send **1 base unit** to a brand-new account. The result, straight from the on-chain program:

```
Simulating transfer of 1 base unit → fresh account CVmXrDaf…
✓ REJECTED: {"InstructionError":[1,{"Custom":17}]}
Program log: Instruction: TransferChecked
Program log: Error: Account is frozen
Program Tokenz…failed: custom program error: 0x11
```

`0x11` (17) is Token-2022's `AccountFrozen`. So a fresh (non-KYC'd) wallet **cannot receive spAssets — because its account is frozen by default**, thawed only by the freeze authority after KYC. That is the entire KYC mechanism, and it is *freeze-gating*, not the transfer hook the docs describe. Reproducible by anyone with a wallet.

**2. "Non-custodial" deserves an asterisk.** The homepage says non-custodial, *"you keep every share."* On-chain, the issuer holds:
- **Permanent delegate** → can seize or burn your spAssets at will (this is *how* liquidation and assignment are enforced — necessary, but total).
- **Freeze authority + default-frozen** → your account works only while the issuer keeps it thawed.

So while the token sits in your wallet, the issuer can freeze it and claw it back unilaterally. That is a defensible, arguably *required* design for a regulated equity token — but "non-custodial / you keep every share" oversells it. The honest framing is "self-hosted, issuer-controlled." **[chain-verified — cite the mint address and extensions in your post; this is the finding no UI tester will have.]**

**3. Devnet ships KYC disabled.** A devnet order logs `DEVNET: KYC verification bypassed (mock)`. Expected for a testnet — but worth one sentence: confirm on mainnet that the bypass is strictly compile-gated to devnet and cannot be reached in production. **[open — needs team clarification]** confirm on mainnet that the mock is compile-gated to devnet builds and unreachable in production.

**4. The program's own instruction names confirm it.** Decoding the on-chain flow, the KYC/compliance path is: `CreateIdentityForUser` → `SetVerified` (KYC program `SKYC…`) → `PlaceBuyOrder` → **`AdminThaw`** → **`FulfillBuyOrderFreezeGated`** / **`MintFreezeGated`** → `MintTo`. The instructions are literally named *FreezeGated* and *AdminThaw* — freeze/thaw gating, not a transfer hook. And Solana's runtime itself emits, on the token account: *"Warning: Mint has a permanent delegate, so tokens in this account may be seized at any time."* That warning is on-chain, not my editorializing.

**5. Orders are admin-fulfilled, not self-executing.** A user `PlaceBuyOrder`, but an **admin/keeper** does `AdminThaw` + `FulfillBuyOrderFreezeGated` + `MintTo` to complete it. Consistent with the regulated-broker model, but worth naming: fills depend on Spout's keeper being live and honest; a user can't self-execute a buy.

**6. The published token IDL spells out the seize power by name.** Spout's public repo ships `idl/spoutsolana.json` (token program `EkU7xRmBhVyHdwtRZ4SJ9D3Nz6SeAvymft7nz3CL2XXB`), and its instruction list includes **`force_transfer`**, **`force_transfer_2022`**, and **`permissioned_transfer`** alongside `mint` / `burn`. So the "issuer can move or claw back a holder's shares" capability isn't an inference from the permanent-delegate extension — it's a first-class, named instruction in Spout's own interface. Legitimate for a regulated/compliant asset (court orders, sanctions, lost-key recovery), but it should be disclosed plainly rather than implied by "non-custodial."

*Recommendation:* fix the docs to describe the real mechanism (default-frozen + freeze-authority KYC, permanent-delegate enforcement); disclose the permanent-delegate/freeze powers and the admin-fulfilled order flow plainly under "non-custodial," since sophisticated users will read the chain anyway.

## Part 2.8 — Who actually controls the protocol (key management)

Reading the on-chain authorities (devnet), the control model is **mixed** — one part well-designed, two parts concentrated:

| Power | Held by | Type |
|---|---|---|
| **Replace the program code** (upgrade authority) | `7N31cE8B…` | **Single system wallet** — no multisig, no timelock |
| **Mint spAssets + permanent delegate (seize/burn any holder)** | `Bs8qi…` | **Single external keypair** (bare hot key) |
| **Freeze / thaw accounts (the KYC gate)** | PDA of program `TACLkU…` | **Program-controlled** ✓ (the good part) |

**Important correction from reading the program binary:** the *intended* model is actually well-designed. The deployed bytecode carries the error strings `"Signer is not the governance admin (Squads multisig)"` and `"Signer is not the operational authority (Turnkey enclave key)"` — so governance is meant to run through a **Squads multisig** and day-to-day operations through a **Turnkey enclave key** (MPC/secure-enclave custody), not raw hot keys. Credit where due: that's a serious key-management design.

The catch is that on **devnet these aren't wired yet** — the binary literally contains `"set GOVERNANCE_ADMIN before init"`, `"Governance admin is not configured (still the placeholder)"`, and `"set ALPACA_ADMIN_WALLET before mainnet"`, and the current on-chain upgrade/mint authorities resolve to single wallets. So the accurate finding is **not** "Spout is centralized" — it's a **mainnet-readiness checklist**: verify before launch that (a) governance admin is the Squads multisig, (b) operational authority is the Turnkey enclave, and critically (c) the **program upgrade authority** (a BPF-loader-level power separate from the in-program roles, still a single wallet on devnet) also moves to the multisig + a timelock. The design intent is right; the wiring is the open question. *Recommendation:* publish the mainnet key-management model and confirm the upgrade authority is under the same multisig.

## Part 2.9 — Reality check: there is no mainnet yet

Spout's on-chain footprint is **four programs**, and mapping them out matters because it reframes what's live:

| Program | ID | devnet | mainnet-beta | Role |
|---|---|---|---|---|
| Orders engine (`spoutOrdersV2`) | `SPoRX…` | ✅ | ✗ | Buy/sell tokenized equities |
| **Vault engine (`spoutVault`)** | `spva…` | ✅ | ✗ | **The headline borrow / lend / covered-call protocol** |
| Freeze router (`freezeRouter`) | `routAR…` | ✅ | ✗ | Thaw/freeze vault-escrow (compliance) |
| Token ACL / compliance (`TACL…`) | `TACLkU6…` | ✅ | **✅ live** | Owns the spAsset freeze authority (KYC gate) |

Neither the orders program nor the spAsset mint (`B3S7…`) exists on **mainnet-beta** — the trading and vault engines are devnet-only. The marketing site speaks in the present tense about *"shares custodied at a regulated US broker,"* Proof of Reserve, and 1:1 backing: on-chain, **that side isn't live yet.**

Two things the program binaries confirm:
- **The broker is Alpaca** (`spoutOrdersV2` references `ALPACA_ADMIN_WALLET` / `alpaca_usdc_account` / an "Alpaca off-ramp wallet") — a real regulated US broker, which *substantiates* the custody claim. But the same strings say it's **"still the placeholder"** and **"set ALPACA_ADMIN_WALLET before mainnet"** — the broker integration isn't wired on devnet.
- **The headline lending product *is* on-chain (devnet), as a separate program.** Correcting an easy first-pass mistake — the borrow/lend engine isn't the orders program, it's `spoutVault` (`spva…`). Its bytecode carries the full protocol: `InitializeVault`, `DepositCollateral`, `BorrowStablecoin`, `RepayDebt`, `SeizeCollateral`, health-factor liquidation (`VaultUnhealthy` / `SettleLiquidationDownside`), a tranched LP pool (`InitializeLpPool` / `DepositLiquidity` / `DistributeYield`), an `InsuranceFund` (`FundInsurance` / `RecapitalizePool`), and — notably — the **covered-call assignment mechanic itself**: `SettleAssignment` ("burned … pooled … shares"), an `auto_buyback` preference, and `ExecuteVaultStrategy` / `UnwindVaultStrategy` with the error *"Collateral type is locked: asset is committed to an off-chain strategy."* That last string is the covered-call in code: when the underlying is written as an option off-chain (via Alpaca), the on-chain collateral is locked. So the borrowing product **is** built and testable on devnet — it just lives in a program the marketing/docs don't name, and it isn't on mainnet.

Two design guards worth crediting, both found in the vault binary:
- The **Turnkey + Squads split is actually enforced in code**, not just aspirational: distinct runtime errors `"Signer is not the operational authority (Turnkey enclave key)"` and `"Signer is not the governance admin (Squads multisig)"`, plus `NotEnclaveSigner` / `NotMultisigSigner`. Recapitalizing a wiped pool *"must go through governance, not a naive deposit."* That's a real separation-of-powers design.
- The vault **rejects collateral mints carrying an unsupported Token-2022 extension (e.g. TransferFee)** — a thoughtful guard against fee-on-transfer accounting attacks.
- Both the mock oracle (`SPOUTVAULT_BUILD=mock-oracle`, *"MOCK price: staleness ignored"*) and the KYC bypass are **build-gated to devnet** — consistent pre-launch discipline.

*Recommendation:* mark pre-launch claims (backing, PoR, live borrowing) as forthcoming or scope them to "at mainnet launch," name the vault program in the docs/IDL, and be explicit about what the current beta does vs. doesn't implement.

## Part 2.95 — The one component that IS on mainnet has a single-key upgrade authority ⚠️

The only Spout-linked program deployed to **mainnet-beta today** is the Token-ACL / compliance program `TACLkU6…` — the program whose PDA (`9D197p…`) is the **freeze authority for the spAsset mint**, i.e. the on-chain KYC gate. Its mainnet **upgrade authority is `DXtFpbPj…`, a plain System-owned keypair** (owner `11111…`, ~2 SOL, no program data) — **not** a Squads multisig or a PDA. Spout uses a *different* key for the mainnet deployment than for devnet (good hygiene), but on mainnet the compliance program that can freeze/thaw holders is replaceable by a single hot key.

*Caveat (fair framing):* I can't fully confirm from the outside whether `TACL` is Spout-owned or a shared/third-party compliance program they build on — the name reads as a generic "Token Access Control List." I'm flagging it as a **mainnet-readiness / please-confirm** item rather than an accusation: if it's Spout's, move that upgrade authority to the same Squads multisig + timelock the vault binary already anticipates. (Raised privately as well, since it concerns a live mainnet key.)

## Part 3 — The lender yield is a levered short-vol trade, not a savings rate

The ~32% Junior APY is presented next to ~9% Senior as if they're points on a risk curve. Worth being explicit about what generates it:

- Harvesting the Volatility Risk Premium means **systematically selling options** — being *short volatility* and *short the right tail*. This premium is real and persistent *in normal regimes*, and evaporates violently in the regimes that matter (Feb 2018 "Volmageddon," Mar 2020).
- The `loss-waterfall` confirms cycles *do* lose money ("assignment cost exceeded premium… or an extraordinary market event") and rests the reassurance on backtests:

  > *"In multi-year backtests… a loss large enough to reach Senior has not occurred in any historical scenario we have tested."*

  Backtests of short-volatility strategies are precisely the ones that look flawless right up until the tail event — that's the nature of selling insurance. "Hasn't happened in the backtest" is the standard famous-last-words of short-vol products.
- **The leverage is structural.** Junior is a thin 15% first-loss slice absorbing the pool's losses in exchange for the residual premium. A ~4–5%/yr pool-level VRP concentrated onto a 15% slice is how you manufacture ~32% — that's roughly the exposure of a leveraged short-vol book, and it should be labeled as such, not as a high-yield deposit.

*Recommendation:* frame Junior explicitly as a leveraged short-volatility / first-loss position; show the historical worst-week drawdown, not just the average APY; state the backtest window and whether it includes 2018/2020.

---

## Part 4 — Correlated assignment vs per-asset circuit breakers

The `circuit-breakers` doc localizes stress **per asset**:

> *"If the fund draws down past a defined threshold, new cycles for affected assets pause… This localizes any stress to the asset that caused it."*

That defends against an *idiosyncratic* blow-up (one name gaps). It does **not** defend against the scenario that actually threatens a covered-call book: a **broad market melt-up**, where calls across *every* name finish ITM *simultaneously*. That loss is correlated and pool-wide — exactly what a per-asset breaker can't localize. The insurance fund (thin — see Part 5) and then Junior absorb a correlated event all at once.

*Recommendation:* add a pool-level / correlation-aware breaker, not only per-asset; model a broad-rally scenario explicitly in the risk docs alongside the crash scenario.

---

## Part 5 — The insurance fund is thin, and the docs disagree with themselves

Two Spout docs give **different numbers** for the first-loss buffer:

- `insurance-fund`: *"seeded at launch… target level of 2% of total pool value. At a $10m pool, that target is $200,000."*
- `loss-waterfall`: *"Seeded at launch (**$50k to $100k**)."*

So the protocol's own first-loss capital at launch is **$50–100k**, targeting **$200k (2%)** — against a pool the same docs size at $10m. That's a 1–2% buffer standing in front of lender capital. It may well cover an *average* bad week; it is not sized for the correlated tail in Part 4. Also worth flagging the doc inconsistency itself as a factual bug. The beta surfaces Proof of Reserves inline ("Reserves 100.2%") but does not surface the insurance fund's current balance or target anywhere in the app — **[open — needs team clarification]** what is the seeded balance today?

---

## Part 6 — The oracle / market-hours mismatch

The price feed is **Stork** (Stork Labs prices the tokenized equities as collateral). Tokenized US equities have a price only when US markets are open (~9:30–4 ET, weekdays). The lending/liquidation engine is 24/7. That creates a structural gap:

- Health Factor and liquidation depend on a feed that is **stale nights, weekends, and holidays**. A Friday-close-to-Monday-open gap-down can't be liquidated in between; a position can be deep underwater before the oracle "reopens."
- Spout does one smart thing here — it **skips cycles through earnings** to avoid the worst single-name gap risk. Good. But that doesn't address weekend/overnight market gaps on the *borrowing/liquidation* side.

**Verified live, 2026-09-21:** all ten instrument feeds reported `priceStale: true` with `priceAgeSeconds ≈ 188,600–188,900` against an 86,400s limit — the keeper hadn't pushed since ~Friday's close. I signed a real $2 GLD buy through Phantom and submitted it; the program invoked `PlaceBuyOrder`, logged `DEVNET: KYC verification bypassed (mock)`, then threw `AnchorError OraclePriceStale (6013)` at `oracle.rs:47`. The vault preflight confirmed `borrow`/`withdraw` hard-blocked with the same code. **Every user path was bricked: buy → stale oracle; deposit → needs tokens that can't be bought; preferences/repay/withdraw → need a vault that needs a deposit.**

**And it's worse than a weekend gap:** after the US market opened (13:31 UTC), the app's own off-chain price API was live (NVDA 222.3, GLD 399.885) and the UI flipped to "Market: Open" — while the on-chain feeds stayed stale past 191,000s. **The keeper is down, not market-hours-gated.** At the time of writing, the app presents as fully functional — live prices, open banner, enabled buy button — while every on-chain transaction reverts. This is exactly the market-hours risk above, observed first as a total product outage and then as a silent false-healthy state.

**One more thing surfaced cross-referencing the deployed programs against the public frontend repo: the components don't agree on which oracle is in use.** The deployed devnet programs price via **Stork** — I read a live `staleness: 53s` from the on-chain feed, and the vault bytecode literally logs `"Stork price: (decimals … staleness …)"` against a `stork_feed` account. But the public app repo (`SpoutSolana/spout-finance`, pushed Apr 2026) fetches a **Chainlink Data Streams** signed report (`lib/solana/fetchChainlinkReport.ts`, an `/api/chainlink/report` route, submitted on-chain as a `signed_report` argument). So one layer says Stork, another says Chainlink. That's not a vulnerability, but it *is* a consistency gap worth resolving publicly: which oracle actually secures collateral valuation at mainnet, and does the market-hours/staleness handling above apply to that one? (The vault binary's own guards — `max_staleness`, `max_clock_skew`, "Oracle timestamp is too far ahead of cluster clock" — are real either way; credit for those stands.)

**Resolution observed 2026-09-22:** the keeper resumed pushing and all ten feeds went fresh (`priceAgeSeconds` 49–274s, `priceStale: false` across the board). A real $4 GOOG buy placed through the production beta UI then confirmed on-chain — `PlaceBuyOrder` logged `Stork price: 353.314 (staleness: 151s)` and succeeded, tx `4LcegboMhAbr1rQphiU8bY8DRywj6CchB35KU7rQTWL4qMq3wot4aw8TwbsiJdJq4JEKvsCqJxCR2n85BbN2LXhY` (devnet slot 502391046). So the ~53-hour window reads as a keeper/ops outage, not a market-hours guard — which strengthens the point: **a single off-chain process failing silently bricked the entire product for a whole weekend with no degraded-state signaling.** And the "Market: Open" banner is computed client-side with no API behind it — it cannot reflect keeper state.

*Recommendation:* document the market-hours oracle behavior explicitly, state which oracle (Stork vs Chainlink Data Streams) is authoritative at mainnet, and state how weekend gap risk is handled on liquidations (wider buffers? paused liquidations? gap insurance?). Operationally: wire the market banner and the buy path to `priceStale`/keeper health, and add keeper alerting — a weekend-long silent outage should page someone.

---

## Part 7 — Liquidation depends on a permissioned buyer set

spAssets enforce wallet-level KYC via a Token-2022 transfer hook — *"Tokens cannot move to non-verified wallets."* That's what makes the regulated model possible, but it has a liquidation cost: **liquidators must themselves be KYC'd**. A permissioned liquidator set is a *thinner* liquidator set, and thin liquidation depth is most dangerous exactly when you need a fast fire-sale in a drawdown. **[open — needs team clarification]** who is in the liquidator set today, and what does depth look like at 2× normal volatility?

---

## Part 8 — Smaller structural notes

- **Liquidity mismatch / run risk:** Junior has a 45-day notice; Senior exits via a FIFO queue dependent on repayments. Long-dated option-cycle carry funding shorter liquidity expectations is a classic mismatch — honest of them to name it, worth stress-testing the messaging. **[open — Earn is marked "coming soon" in the beta, so the lend-side flows are untestable today.]**
- **Counterparty concentration:** a single regulated US broker holds all the shares and runs options execution; SIPC protection caps at $500k against a multi-million pool. Single point of failure worth naming.
- **"Audited" but unnamed.** The `risks-disclaimers` page states *"the protocol has been audited,"* but the `security-and-compliance` page names **no audit firm, no report link, and no date**, and there is **no public bug-bounty program** (security contact is the general `contact@spout.finance`). For a protocol custodying tokenized equities pre-launch, an unspecified audit is a transparency gap — "audited by whom, when, what scope, and can I read it?" is a fair question to put in writing.
- **FinCEN MSB ≠ securities clearance:** Spout is careful to say MSB registration "is not an endorsement." Fair. But tokenized US equities + lending sit in live securities/regulatory uncertainty; that's the largest un-hedgeable risk and belongs higher than the bottom of a risk page.

---

## Part 9 — UX audit (Nielsen's 10 heuristics)

Structured heuristic evaluation of the Spout beta, marketing site, and docs. Severity scale: **4 Catastrophic · 3 Major · 2 Minor · 1 Cosmetic.** UI claims are verified against live app state (captures in [`../screenshots/`](../screenshots/)); API and chain claims are verified against the live endpoints and devnet.

**Top 3 UX issues**
1. Covered-call **assignment isn't disclosed at borrow time** — the product's biggest surprise is its least-visible fact. *(H5/H1, Sev 4)*
2. **"0% interest" and "never a taxable sale" set a false mental model** the mechanics don't honor. *(H2, Sev 3)*
3. **Risk asymmetry (Junior first-loss, 45-day lockup) surfaced after interest, not before commitment.** *(H1/H5, Sev 3)* — verified in the docs and lend-side copy; Earn isn't live yet, so this lands at launch.

---

**H1 · Visibility of system status** — ⭐⭐⚪⚪⚪
- **1.1 (Sev 4) — Assignment/covered-call status is invisible at decision time.** A borrower can't see, on the borrow screen, that their collateral is being continuously covered-called and can be sold at the strike this cycle. The one status that changes the outcome is the one not shown. *Fix: a per-position "cycle status / assignment risk" indicator.* (verified: neither the borrow screen nor settings surfaces assignment state — capture `../screenshots/borrow-page.png`)
- **1.4 (Sev 4) — "You own 0.01 GOOG" is false at the moment it's shown.** The purchase-confirmation modal asserts ownership before fulfillment delivers anything; it also gives a specific fill time ("9:30 AM ET, 22 Sep") that was missed by 34h+. *Fix: "Order confirmed — pending fill" state with a live status, honest ETA wording, and a visible escrow explanation.* (capture: `../screenshots/purchase-confirmed-modal.png`)
- **1.2 (Sev 3) — Health Factor legibility.** HF is surfaced as a labeled element on Borrow (good), but before a vault exists it renders as "—", and the numeric liquidation trigger/buffer isn't shown even then. *Fix: show trigger price and buffer in plain numbers, including a worked preview at the chosen LTV.* (capture: `../screenshots/borrow-page.png`)
- **1.3 (Sev 3) — No degraded-state signaling.** Verified: while all oracles were >2 days stale, the buy preflight returned `blockers: []` — the app happily builds and requests a signature on a transaction that *cannot* succeed. There is no "markets closed / pricing unavailable" state. The borrow side does this correctly (see 5.2) — the trade side doesn't. *Fix: surface oracle staleness as a first-class UI state before any signing request.*

**H2 · Match between system and the real world** — ⭐⭐⚪⚪⚪
- **2.1 (Sev 3) — "Never trigger a taxable sale" / "keep every share."** In real-world terms this is a *tax* promise the assignment mechanic breaks (assignment = a sale = a taxable event; Auto-Roll = new cost basis). The words teach a mental model the system doesn't honor. *Fix: language like "defer a sale while you hold" + an assignment/tax explainer.*
- **2.2 (Sev 3) — "No margin calls" is contradicted by the mechanics.** Verified in-app: the Borrow page sells "No interest, no margin calls, no hidden fees" while the vault enforces a 65–70% liquidation ratio with `liquidating`/`liquidationPrice` machinery. Automated liquidation *is* a margin call — the difference is you don't get warned first. *Fix: "no manual margin calls" if that's the intent, plus the liquidation threshold in the same sentence.*
- **2.3 (Sev 2) — "0% interest" hides a real cost.** True to the letter, but the cost (capped upside + assignment) is moved off the interest line where users look for it. *Fix: a "what you give up" line next to "0%."*

**H3 · User control and freedom**
- **3.1 (Sev 3) — `autoBuyback` is a per-vault on-chain setting.** Verified: the vault API exposes an `autoBuyback` flag ("lets the vault repurchase your collateral automatically after a liquidation") — the Auto-Roll mechanic lives here. Default could not be checked (vault init is blocked behind a token purchase), but it must be presented at borrow time with an opt-out, not buried post-liquidation. **[open — vault init is gated behind a delivered token purchase, so the default could not be observed]**
- **3.2 (Sev 2) — Exit path clarity.** Repay/withdraw preflights return precise plain-language blockers via the API, but the full closure path (and any cycle-timing constraints) can't be seen without first opening a position.

**H4 · Consistency and standards**
- **4.1 (Sev 2) — Docs vs product terminology.** Docs say "transfer hook" for KYC; the chain uses freeze-gating (Part 2.7). If the app repeats "transfer hook," it's inconsistent with its own behavior. Numbers also disagree across docs (insurance fund $50–100k vs $200k, Part 5). *Fix: one source of truth.*
- **4.2 (Sev 2) — Surface disagreements, verified in-app:** the trade UI lists 11 instruments incl. AAPL while `/api/market-data/instruments` returns 10 (no AAPL); the Portfolio page shows "Unrealized P&L +$0.60 (+6.04%)" with zero holdings; every instrument row renders the Pfizer logo regardless of ticker. Small things, but they erode the precision a trading UI lives on. *(screenshots in `screenshots/`)*

**H5 · Error prevention** — ⭐⭐⚪⚪⚪
- **5.1 (Sev 4) — No pre-commit warning of assignment/tax consequence.** The highest-consequence, least-reversible outcome (forced sale of your stock) has no confirmation step framing it — verified across buy, borrow, confirm, and settings. *Fix: a one-time "you understand assignment can sell your shares" acknowledgment.*
- **5.2 (Positive + Sev 3 asymmetry) — Borrow preflight is genuinely good; buy preflight is missing checks.** Verified: the vault endpoint returns `maxBorrow`, `maxWithdraw`, `currentLtv`, `healthUtilization`, `liquidationRatio`, `priceStale`, and plain-language blockers ("Deposit AAPL as collateral first — that opens your vault", "The AAPL price is 188610s old…"). Excellent error prevention. But `/api/orders/buy` returned `blockers: []` for a tx that reverts `OraclePriceStale` on simulation — the same staleness check is simply absent on the trade path. *Fix: share the preflight layer between trade and vault.*
- **5.3 (Sev 3) — Frozen-account failure UX.** A non-KYC'd wallet's transfer fails with `custom program error: 0x11` (verified on-chain). If that raw error reaches users, it's un-actionable. *Fix: translate to "this wallet isn't verified yet."* (→ also H9)

**H6 · Recognition rather than recall**
- **6.2 (Positive) — Declined-signature toast is exactly right.** *"You declined the signature, so nothing was submitted."* — clear, honest, no raw RPC error. The market status pill ("Market: Closed 🌙 2:33:10") with live countdown is also good. (capture: `../screenshots/buy-market-closed-panel.png`) The gap: the pill reflects stock-market hours, not keeper/fulfiller health.
- **6.1 (Positive) — Vault blocker copy is a model for the rest of the app.** Verified plain-language reasons for every blocked action, including severity and remedy ("The GLD price is 188,610s old — borrow and withdraw revert until a keeper pushes a fresh price"). If the trade flow and error surfaces matched this standard, most of this section's issues shrink.

**H7 · Flexibility and efficiency**
- **7.1 (Sev 1) — Power-user affordances** — a "max" affordance exists via the LTV slider cap; keyboard/quick-entry polish gaps remain.

**H8 · Aesthetic and minimalist design**
- **8.1 (Sev 1–2) — Dashboard hierarchy** — the borrow page's primary action is legible in the empty state (capture: `../screenshots/borrow-page.png`); re-check density once positions exist.

**H9 · Help users recognize, diagnose, recover from errors** — ⭐⭐⚪⚪⚪
- **9.1 (Sev 3) — On-chain errors are raw. Verified.** A signed buy reverted with `custom program error: 0x177d`; the API passes the simulation failure text through to the client unmapped. Nothing tells the user "prices are stale, try again when US markets open." *Fix: an error-map layer — 0x177d → "market pricing is stale," 0x11 → "wallet not verified."*

**H10 · Help and documentation** — ⭐⭐⭐⭐⚪
- **10.1 (Positive) — Docs are genuinely strong** — thorough, honest about the risk waterfall, and the reason this teardown is even possible. Credit them.
- **10.2 (Sev 2) — Help isn't contextual.** The docs are excellent but separate; the *app* needs inline "learn more" at the moments of decision (assignment, Health Factor, tranche choice), not a docs link.

**Overall usability: ~5/10.** Updated from provisional after executed-flow testing: the vault preflight layer is a genuine strength (plain-language blockers, full risk params), but the trade path signs transactions that cannot succeed and surfaces raw Anchor errors. Strong docs and a surfaced Health Factor; undermined by the core disclosure gap (assignment/tax) at exactly the decision points. The single highest-leverage fix is making covered-call assignment and its tax consequence visible *before* the user borrows.

> Companion method included: a novice-persona **cognitive walkthrough** of the buy→borrow task ([`cognitive-walkthrough.md`](cognitive-walkthrough.md)).

---

## Part 9.5 — Full chain-history census: what 3,100+ transactions actually did

I pulled every transaction touching all four Spout programs (devnet, genesis → 2026-09-22) and decoded them by instruction. The aggregate tells a story no single UX session can.

**Orders engine** (`SPoRX`, 3,102 txs since 2026-06-08):
- 1,284 buys placed → 1,124 filled (87.5%); 268 sells placed → 247 filled.
- **Cancel is broken end-to-end, with numbers.** 102 `RequestCancelOrder` transactions exist; **zero** refund instructions have ever run — no `CancelOrder`/`RefundSellOrder` has ever executed, and not one of the 102 requests moved a single USDC lamport (verified via token-balance deltas on every tx). Twelve of those cancelled orders were then **filled anyway**.
- Right now: **178 pending orders holding $1,829 in escrowed USDC** — the stranded-funds pile is compounding week over week.
- Fill latency is bimodal: median ~10s when the keeper is live, but **37% of fills took >2h** (max ~28 days). Every long tail is a keeper outage like the one in Part 6 — the product's availability is a single off-chain process.
- **Fill-price drift:** sampling place→fill pairs shows a median **0.40% adverse slippage** vs the order-time quote (range −1.8% to +1.8%, both signs) — fills execute at the fill-time oracle price, not the displayed quote. Combined with `limit_price` existing in the IDL but never being set by the UI (every order in the feed: `limitPrice: 0`), users have zero price protection between signature and fill.

**Vault engine** (`spva`, 333 txs since 2026-08-04) — the headline product, and the part nobody else checked:
- 98 vaults opened, 112 borrows, 35 repays — the borrow loop works and has real usage.
- **4 real liquidations** (`SettleLiquidationDownside` + collateral burn): Sep 2, Sep 4 ×2, Sep 18. The liquidation path is genuinely exercised — credit.
- **The covered-call engine has never executed on-chain. Zero.** No `SettleAssignment`, no `ExecuteVaultStrategy`/`UnwindVaultStrategy`, no `DistributeYield`, no `SettlePoolBuyback`, no `FundInsurance`/`RecapitalizePool` — the only fund events are a single `InitializeInsuranceFund` (Aug 24) and a single `DepositLiquidity`. The mechanism that funds "0% interest" and "9–32% APY" — premium collection, assignment settlement, yield distribution — has never run through the program on devnet. Expected while options settle off-chain at Alpaca, but it means **nothing about the yield claims is on-chain-verified yet**, and that should be said plainly.

**Freeze router** (`routAR`, 317 txs): 254 `AdminThaw` — the KYC thaw path is the most-used compliance operation; consistent with the freeze-gating model in Part 2.7.

**Compliance program** (`TACL`): ~36,800 lifetime txs — far beyond Spout's beta scale, corroborating that it's shared/third-party infrastructure (and why the Part 2.95 ownership caveat matters).

**Causal pin for a bug others hit:** the vault's `MigrateCollateralType` batch ran **2026-09-09 00:28–29 UTC** — 19 collateral-type migrations in one minute. The `CollateralType: unexpected length 213` decode errors that bricked the Borrow tab for ~a week begin exactly there: the schema migrated on-chain while the frontend decoder lagged. A deployment-coordination bug with a timestamp.

*Recommendation:* ship a working `cancel_order`/`RefundSellOrder` path (or remove the button), drain or surface the 178 stranded escrows, wire `limit_price` into the UI with a deviation warning, add keeper alerting, and label the covered-call settlement as "off-chain until mainnet" wherever yield is quoted.

---

## Part 10 — Bugs

Verified during the 2026-09-21 pass (full evidence in `spout-retest-2026-09-21.md`):

| Sev | Bug |
|-----|-----|
| 🔴 P0 | **Oracle staleness bricked every flow for ~53h** (2026-09-19→21). All feeds >52h old vs 24h limit; signed buy reverted `OraclePriceStale` 6013; borrow/withdraw hard-blocked. Keeper resumed 2026-09-22; a real $4 GOOG buy then confirmed on-chain — proving both the outage and its resolution. Root cause: single silent keeper failure + a client-side "Market: Open" banner that can't reflect chain state. |
| 🟠 | **Fulfillment misses its own promised window, and the modal claims ownership that doesn't exist.** The confirmation modal promised *"Your order fills at 9:30 AM ET, 22 Sep"* — the order was still pending 34h later (verified 09-23, capture `screenshots/evidence-order-pending.png`). The same modal asserted *"Your purchase has been confirmed. You own 0.01 GOOG"* while the wallet held zero GOOG. Escrow account `6s9tiW4Rcz…` shows only the placement tx; the feed has no status/ETA fields; canceling the stuck order costs `0.004 USDC`. Backlog grew 145 → 268 pending orders in 24h. Fills are likely market-hours-gated (Alpaca) — defensible, but the stated window wasn't honored and nothing updates the user. |
| 🟠 | **`limitPrice` exists in the protocol but the UI never sets it** — the `place_buy_order` IDL carries a `limit_price` arg, yet every order in the global feed has `limitPrice: "0"` (unprotected market order at whatever price the fulfiller reads). And when set manually, absurd values ($1 on a ~$353 asset) pass silently → `ok:true`, zero warnings, USDC locked into an unfillable order, exit costs a 0.004 USDC cancel fee. Needs a market-deviation warning AND the UI to actually use the field. |
| 🟠 | **No rate limiting anywhere** — 15 rapid tx-build calls all 200; `redeem_beta_code` identical. Each build call does RPC work on their quota. |
| 🟠 | **`/api/kyc/onboard` live + unauthenticated** — returns `already-onboarded` for any arbitrary wallet (KYC-status oracle bypassing the Privy gate on `/api/kyc/status`) and likely burns Persona inquiries for non-onboarded addresses. |
| 🟠 | **Buy button enabled during closed market + stale oracle** — nothing gates the signing path; preflight returns clean on doomed txs. |
| 🟠 | **Buy preflight missing the staleness check** — returns `blockers: []` on a tx that always reverts. Vault preflight checks it; trade doesn't. |
| 🟠 | **`/api/orders` is an unauthenticated global order feed** — `userAddress` param ignored; all 155 pending orders across 30 wallets (addresses, amounts, prices) readable by anyone. |
| 🟠 | **KYC status leaks via vault endpoint** — `/api/kyc/status` needs a Privy token, but `GET /api/vault/deposit` returns `kycStatus: "verified"` for any arbitrary address. |
| 🟠 | **`/api/orders/buy` builds full unsigned txs for arbitrary wallets** unauthenticated — response map exposes the target's identity account, USDC ATA, and wallet-link state. Signature remains the funds boundary. |
| 🟠 | **Beta passcode gate is UI-only** — `redeem_beta_code` is unthrottled (4 rapid probes, all 200; 8-char codes) and no API route enforces the gate. |
| 🟠 | **`/api/kyc/onboard` unauthenticated** — accepts `{walletAddress}` alone; currently 500s (Persona env unwired), but would create Persona inquiries for arbitrary wallets when fixed. |
| 🔵 | `access_count` RPC publicly callable (leaks beta usage count: 225). |
| 🔵 | `cancellationFeeUsdc: 0.004` on order cancel — verify the UI discloses it. |
| 🔵 | AAPL tradable on-chain (in the error-message allowlist) but absent from `/api/market-data/instruments` — chain and catalog disagree. |
| 🔵 | Collateral mints deploy under SPL Token on devnet, not Token-2022 as docs describe — freeze-hook model unverifiable until mainnet. |
| 🔵 | Portfolio shows "+$0.60 (+6.04%) Unrealized P&L" with zero holdings. |
| 🔵 | All instrument logos render as the Pfizer logo (screenshot: `buy-market-closed.png`). |
| 🔵 | No HSTS on `beta.spout.finance` (main site has it); no `security.txt`; Anchor errors leak source paths (`oracle.rs:47`). |
| 🔵 | `DEVNET: KYC verification bypassed (mock)` in program logs — on-chain KYC is mocked on devnet, so identity-gating behavior is unverifiable pre-mainnet. |
| ✅ | Verified good: `/api/orders/submit` simulates before relaying; `orders/cancel` validates ownership server-side before building the tx; `sell`/`repay`/`withdraw` all block correctly with real reasons; $2 min enforced; input validation clean (negatives, bad tickers, malformed wallets, insufficient balance all rejected with useful messages); Supabase fully closed to anon; CSP/XFO/nosniff on beta; `mailer_autoconfirm: false`. |

---

## What Spout gets right

- The risk waterfall is disclosed plainly, with a real first-loss-from-day-one insurance fund.
- Skipping earnings cycles shows genuine options discipline.
- Proof of Reserve + Token-2022 KYC + regulated custody is a credible attempt at a *compliant* on-chain equity model, not a hand-wave.
- The docs are more honest than most DeFi — this teardown is possible *because* they wrote the mechanics down.

## Bottom line

The engine is real; the risk is mostly where a sophisticated user would expect (short-vol tail, oracle/market-hours, thin first-loss). The one thing that needs fixing before a public launch isn't the model — it's the **gap between the front-page promise and the assignment mechanic**. Second place, and now verified live rather than theoretical: **operational resilience** — a single stale keeper took the entire product down for a weekend, and the trade path didn't even notice it was signing doomed transactions. Close the messaging gap, add oracle-freshness to every preflight, and this is a defensible product.

---

*Submitted for the Spout Finance Product Feedback bounty by [@lpsmurf](https://github.com/lpsmurf). Independent testing; not investment advice.*
