# Spout Finance — Beta Testing Log

**Bounty:** Spout Finance Product Feedback ($1,000 USDC, 7 prizes) · deadline **2026-09-28** · [listing](https://superteam.fun/earn/listing/product-feedback-spout-finance)
**Tester:** lpsmurf · **Beta app:** beta.spout.finance · **Started:** ______

> How to use: work each flow below, log what you see. Capture **screenshot + timestamp + txn signature** for every entry. Severity: 🔴 blocking · 🟠 major · 🟡 minor · 🔵 UX/polish · 💡 feature idea. A clean log with reproductions is what separates a top-3 submission from fluff.

---

## 0. Setup & first impressions

| # | Step | Expected | Actual / observation | Sev | Screenshot |
|---|------|----------|----------------------|-----|-----------|
| 0.1 | Enter beta code `B4M779SF`, connect wallet | Clean onboarding, clear next step | | | |
| 0.2 | KYC / wallet verification flow | Explained why (Token-2022 hook), what data, how long | | | |
| 0.3 | First-run dashboard | Obvious "what do I do first" | | | |
| 0.4 | Time-to-first-meaningful-action | (note minutes + friction points) | | | |

---

## 1. Borrow flow (the core product — test hardest)

| # | Step | What to check | Actual | Sev | Shot |
|---|------|---------------|--------|-----|------|
| 1.1 | Deposit spAsset collateral | Which assets available? balances correct? | | | |
| 1.2 | Draw stablecoin at 50% LTV | Max-borrow enforced? which stablecoin? | | | |
| 1.3 | **Health Factor display** | Shown pre-confirm? formula legible? | | | |
| 1.4 | Fee disclosure before signing | Liquidation fee (4–12.5%) surfaced up front? | | | |
| 1.5 | **Assignment disclosure** ⭐ | Does the UI warn that collateral is covered-called and can be **sold at strike** (a taxable event)? | | | |
| 1.6 | Auto-Roll toggle | Default on? consequences explained? | | | |
| 1.7 | Confirm + txn | Signature: __________ · confirmation UX | | | |

⭐ 1.5 is the headline test. The marketing says "never trigger a taxable sale / keep every share," but assignment sells the share. If a borrower can't tell from the UI they're about to be covered-called into a taxable sale, that's a top finding — screenshot exactly what is and isn't disclosed.

---

## 2. Assignment / Auto-Roll

| # | Step | What to check | Actual | Sev | Shot |
|---|------|---------------|--------|-----|------|
| 2.1 | Find assignment history / cycle status in UI | Visible? clear? | | | |
| 2.2 | Auto-Roll ON behavior | Rebuys same asset? new cost basis shown? | | | |
| 2.3 | Auto-Roll OFF behavior | Residual settles to stablecoin cleanly? | | | |
| 2.4 | Tax-lot / cost-basis info | Any basis reset warning after a roll? | | | |

---

## 3. Liquidation (push a position toward HF < 1.00 on testnet)

| # | Step | What to check | Actual | Sev | Shot |
|---|------|---------------|--------|-----|------|
| 3.1 | Drive collateral value down / debt up | HF updates live? | | | |
| 3.2 | Warning lead-time | "notified well in advance" — how far? channel? | | | |
| 3.3 | Partial-liquidation trigger at HF<1.00 | Sells "just enough"? fee = liquidation buffer? | | | |
| 3.4 | **Weekend / market-closed behavior** | Does HF/oracle update when US market is closed? stale? | | | |
| 3.5 | Liquidator set | Who can liquidate given the KYC transfer hook? | | | |

3.4 is the oracle/market-hours-mismatch test — equities price 9:30–4 ET weekdays, DeFi is 24/7. Note what the oracle shows overnight/weekend.

---

## 4. Lending (both tranches)

| # | Step | What to check | Actual | Sev | Shot |
|---|------|---------------|--------|-----|------|
| 4.1 | Deposit to Senior tranche | APY shown? priority-yield explained? | | | |
| 4.2 | Deposit to Junior tranche | First-loss risk clearly disclosed? | | | |
| 4.3 | Junior 45-day withdrawal notice | Surfaced before deposit, not after? | | | |
| 4.4 | Senior withdrawal / FIFO queue | Liquidity messaging honest? | | | |
| 4.5 | Insurance fund balance display | Matches docs? ($50–100k seed vs 2%/$200k target) | | | |

---

## 5. Proof of Reserve & security claims

| # | Step | What to check | Actual | Sev | Shot |
|---|------|---------------|--------|-----|------|
| 5.1 | On-chain Proof of Reserve | Actually resolves? 1:1 verifiable? | | | |
| 5.2 | Move spAsset to non-KYC wallet | Transfer hook blocks it? error clear? | | | |
| 5.3 | Any audit report linked? | Which auditor, what scope, what date? | | | |

---

## 6. Bug hunting — edge cases (log any that misbehave)

| # | Input / action | Result | Sev | Shot |
|---|----------------|--------|-----|------|
| 6.1 | Amount = 0 / negative / dust / max+1 | | | |
| 6.2 | Extreme decimals / precision | | | |
| 6.3 | Double-submit / rapid clicks (idempotency) | | | |
| 6.4 | Wallet disconnect mid-flow | | | |
| 6.5 | Stale/expired session | | | |
| 6.6 | Mobile / responsive layout | | | |
| 6.7 | Error messages (are they human-readable?) | | | |
| 6.8 | Numbers/units consistency (base units vs display) | | | |

> ⚠️ If any bug touches **funds, liquidation logic, the oracle, or the transfer hook** at the *smart-contract* level (not just UI), STOP and treat it as a private security disclosure — do not put it in the public post. Note it here marked `SECURITY-PRIVATE` and we'll route it to Spout's security contact separately. That's worth more than the bounty.

---

## Running findings summary (fill as you go)

**Bugs:**
1.

**UX friction:**
1.

**Feature ideas:**
1.

**Questions for @SpoutHelp:**
1.

---

## Public-surface testing — DONE 2026-09-09 (no beta code used)

Confirmed without logging in (safe to cite in the submission):

- **Gate:** beta.spout.finance is an **email + passcode** gate ("Testnet is live") — no wallet-connect until inside.
- **Tranche math is correct.** Recomputed their $10m-pool example: Senior ~8.9% / Junior ~32.6% APY — matches their docs to rounding. (Credibility point — say so.)
- **…but the input is optimistic:** $30k/wk = ~15.6%/yr gross premium on the pool (~7.8% on collateral at 2× LTV). Headline APYs assume a healthy-vol regime; Senior's 7% priority-off-the-top means Junior compresses hard in low-vol years. → teardown Part 2.5.
- **Insurance-fund figures contradict across their own docs:** `$50–100k` seed (loss-waterfall) vs `2% / $200k` target (insurance-fund). Factual bug — cite both.
- **"Audited" but no auditor named**, no report, no date, **no bug bounty**. Security contact = `contact@spout.finance` (general). → teardown Part 8.
- **Oracle = Stork Labs** (prices the tokenized stocks). → name it in Part 6.
- **Public repo** `SpoutSolana/spout-finance` exists but is a **stale frontend monorepo** (last push 2026-04, has `idl/`) — not the current contracts. Not the audit target.

## ⚠️ Testnet Terms — READ before submitting (Sections quoted from spout.finance/testnet-terms)

- **§8.1 Feedback assignment:** *"you hereby irrevocably assign to Company all right, title, and interest in… the Feedback, including all IP rights."* → Anything you report becomes Spout's IP. Normal for feedback programs, but know it — and it's fine to still publish your teardown publicly (the bounty *requires* public content); just be aware the ideas are assigned.
- **§7.1 No-rewards clause:** the testnet ToS says testnet participation *"do[es] not entitle you to… any… rewards, compensation, payment… of any kind."* The **$1,000 bounty is a separate Superteam listing/contract**, not the testnet ToS — so the payment obligation lives with the Superteam listing, not here. Not a blocker, but don't be thrown by the testnet ToS disclaiming rewards.
- **§4.1 Sanctions:** standard — no sanctioned/embargoed-jurisdiction users.
- **No real funds:** everything is simulated; test assets have no value. Standard.

## Interactive flows (wallet login + KYC + signing)

Sections 1–6 above were exercised interactively by the reviewer with a KYC-verified beta account and a funded devnet wallet: connecting Phantom, placing and signing orders, and observing preflight/submit behavior. Signed-transaction evidence is recorded in `retest-2026-09-21.md` and `retest-2026-09-22.md`.

---

## On-chain testing — DONE 2026-09-09 (read-only, via agent wallet's devnet footprint)

Verified directly on Solana devnet from the wallet's own transactions — no signing, no KYC needed. **Citable, chain-verified, unique.**

**Programs / keys (devnet):**
- Spout program: `SPoRXgsB4gWZmWPwyndoRWQrmZXKUc7o7oPMdkkGRcG`
- KYC program: `SKYCVrkX3mQaHwZcLrUtuvzii43kma7kBUim5MaQm6k`
- spAsset mint (Token-2022): `B3S7TuCmBPsNJW32SzRLBwbmWmgNzFBuHnBG9rdvt6K5` (9 decimals)
- Mint authority / permanent delegate: `Bs8qiyCLC2dQErqFS7sjoMuvfmZvFzDGi5z7n5BX71mf`
- Freeze authority: `9D197pJCjzzmmr1FjYyVkmepRSfoLawH9Zna1mZ6BLyj`
- Oracle: Stork (`stork1JU…` price account)

**Findings:**
1. ⭐ **Docs say "transfer hook"; chain has none.** KYC enforced via `defaultAccountState: frozen` + freeze authority, not a transfer-hook program. Doc error. → teardown Part 2.7.
2. ⭐ **spAssets are issuer-controlled:** `permanentDelegate` (seize/burn any holder) + freeze authority + default-frozen. Tension with "non-custodial / you keep every share." → Part 2.7.
3. **Oracle = Stork confirmed on-chain:** price 18 decimals, live `staleness: 53s` readout.
4. **Pricing math correct on-chain:** 10 USDC → 0.031653 spAsset @ $315.925 = $10.00.
5. **Devnet KYC bypassed (mock)** — verify mainnet enforces + bypass is devnet-gated.
6. USDC (devnet mock) = `4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU` (legacy SPL, 6 dec).

> ⚠️ SECURITY-PRIVATE candidate: if the KYC-bypass path is *not* strictly devnet-compile-gated, or the permanent-delegate/freeze keys are single (non-multisig) hot keys on mainnet, those are private-disclosure items for contact@spout.finance, not the public post. Verify before publishing.

---

## Signed on-chain test — EXECUTED 2026-09-09 (via probe page + Phantom, key never left the extension)

**Freeze-gate KYC test — CONFIRMED LIVE:**
- Wallet spAsset balance: `0.031542255` (raw 31542255) — holds real devnet spAsset (mint B3S7).
- Attempted transfer of 1 base unit → fresh account `CVmXrDafzCBJpyvwpVjF8KPc8BPL3KTNjBYmmuR4j6qh`.
- **Result (simulation, authoritative): `{"InstructionError":[1,{"Custom":17}]}`** = Token-2022 `AccountFrozen (0x11)`.
- Program logs: `Error: Account is frozen`.
- Signed send (sig `fWTCRBs3…`) was dropped (client timeout, not included) — simulation is the proof; a landed failed tx not needed.

**Conclusion:** KYC is enforced by **default-frozen accounts + freeze authority thaw**, NOT a transfer hook. Docs are wrong. Chain-verified and reproducible via `spout-onchain-probe.html`.

Read-only compliance panel (all confirmed on-chain):
- defaultAccountState: **frozen** · permanentDelegate: `Bs8qiyCL…` (can seize) · freezeAuthority: `9D197pJC…` · transferHook: **NONE** (docs claim one).

---

## On-chain authority / key-management analysis — DONE 2026-09-09 (read-only)

- **Program upgrade authority (SPoRX):** `7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp` = **single system wallet** (0.67 SOL). One key can replace the lending program. No multisig/timelock.
- **KYC program (SKYC) upgrade authority:** `BD29wQ5Tj7b1MEFqquRxASsTqoN38oCrjxpU4riTZx7C`.
- **Mint authority + permanent delegate:** `Bs8qi…` = **empty account = single external keypair** (can mint ∞ spAssets + seize/burn any holder).
- **Freeze authority (KYC gate):** `9D197p…` = **PDA of program `TACLkU6CiCdkQN2MjoyDkVg2yAH9zkxiHDsiztQ52TP`** (deployed, executable) → freeze/thaw is program-governed (the well-designed part).
- **Mainnet:** SPoRX program and B3S7 mint **do NOT exist on mainnet-beta** — devnet-only. Regulated/backed/PoR claims are pre-launch aspirational.
- ⚠️ Fair framing: single-key authorities are NORMAL on devnet — this is a **mainnet-readiness** flag (move upgrade + mint/permanent-delegate to multisig+timelock before launch), NOT a live-centralization accusation.

---

## Program structure (read-only, devnet) — DONE 2026-09-09
`getProgramAccounts(SPoRX)` → 58 state accounts, grouped by discriminator:
- **2 accounts** (disc 03db48…) → the **two tranches** (Senior/Junior) — corroborates the tranched model.
- **7 accounts** (disc 04618d…) → likely the **supported markets/assets** (~7 tokenized equities live on devnet).
- **34 accounts** (disc 041031…) → **orders / positions**.
- ~15 singletons → global config, pool, insurance fund, oracle config.
- Exact insurance-fund balance not decoded (no IDL; devnet values are simulated/no-value per their ToS — low signal).

**RPC-limited (not findings):** holder concentration + spAsset ticker/metadata came back empty from public devnet RPC (Token-2022 largest-accounts / metadata not indexed there) — inconclusive, not "zero."

**On-chain testing status: COMPLETE.** Everything reliably obtainable via public RPC + one signed test is captured. Remaining items are blocked: signed control-test needs a 2nd KYC'd/thawed account (Spout admin only); PlaceBuyOrder edge-cases need IDL reconstruction + are gated. Not worth the effort vs findings already secured.

---

## Web recon + program-binary analysis — DONE 2026-09-09 (autonomous)

### Web security headers (passive)
- **beta.spout.finance (the wallet-signing app): MISSING ALL headers** — no HSTS, no CSP, no X-Frame-Options, no X-Content-Type-Options. → **no clickjacking protection on a transaction-signing surface** (attacker could iframe it). Served via CloudFront. Real web-sec finding for the submission (Sev: Major).
- spout.finance (marketing, Cloudflare): has HSTS (weak, 1-day) + nosniff; no CSP / no X-Frame-Options.
- No `/.well-known/security.txt` on either (404) — no disclosure channel.

### Frontend bundle
- Next.js app; bundle contains a **Supabase anon key** (project `axartrdqynqtfclakxru` → `axartrdqynqtfclakxru.supabase.co`).
  - ⚠️ **NOT a vulnerability by itself** — Supabase anon keys are public by design (same class as a Stripe publishable key). Reported honestly as such.
  - **Critical open question:** is Row-Level Security (RLS) enforced on all tables (esp. KYC/PII/orders)? If not, the anon key = data exposure.
  - **NOT TESTED** — querying their Supabase would be active probing of third-party prod infra and could expose real user KYC/PII → SECURITY-PRIVATE, out of scope for the public bounty. Recommend Spout verify RLS; recommend user NOT probe.

### Program binary (SPoRX = spoutOrdersV2, 730KB, strings analysis)
- ✅ **KYC bypass is BUILD-GATED** (`SPOUTORDERS_BUILD=devnet` / "DEVNET: KYC verification bypassed (mock)") → compile-time devnet-only. Resolves the mainnet-KYC concern in Spout's favor.
- ✅ **Intended key management is strong:** governance = **Squads multisig**, operational = **Turnkey enclave key**. → CORRECTED the Part 2.8 "single-key centralization" finding (was devnet-placeholder state, not intent).
- ⚠️ Pre-launch placeholders in binary: `set GOVERNANCE_ADMIN before init`, `set ALPACA_ADMIN_WALLET before mainnet`, off-ramp "still the placeholder".
- ✅ Broker = **Alpaca** (substantiates "regulated US broker").
- ✅ Oracle guards EXIST: `max_staleness`, `max_clock_skew`, `Oracle timestamp too far ahead of cluster clock`, fill-deviates-from-quote → tempers oracle-risk section (credit them).
- 🔍 Deployed program = **orders/trading engine** (PlaceBuyOrder/PlaceSellOrder + KycGated/FreezeGated fulfill). The **borrowing/lending/covered-call protocol is NOT the deployed devnet program** — headline product not yet on-chain. Architecture: `spoutOrdersV2` + a `wrapper` program (`mint_kyc_gated.rs`).
- Full instruction set: InitializeConfig, UpdateAuthority, SetPaused, Create/UpdateOracleConfig, CreateTokenEscrow, FundVault, PlaceBuyOrder, PlaceSellOrder, Fulfill{Buy,Sell}Order{Kyc,Freeze}Gated, RequestCancelOrder, RefundSellOrder. Min order: 2 USDC buy / 1 USDC sell.

---

## Keeper / separation-of-duties — DONE 2026-09-09
- The **operational keeper** that signs `FulfillBuyOrder…` on devnet = `7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp` — the **SAME wallet as the program upgrade authority**. On devnet, one key both operates the protocol and can replace its code → no separation of duties (placeholder state; binary shows intended mainnet split = Turnkey operational + Squads governance). Sharpens Part 2.8.
- Oracle-config exact `max_staleness` value not in the recent 25-tx window (config created earlier); binary confirms the guards exist — exact value is a marginal nice-to-have, not pursued.

## AUTONOMOUS TESTING: COMPLETE
Covered: on-chain compliance model (live-proven freeze-gate), key management + separation of duties, mainnet existence, program structure, full instruction set + build flags (binary), web security headers, frontend bundle (Supabase anon key → private RLS note), broker identity (Alpaca), oracle guards. Remaining gains require the user's KYC'd UI session (screenshots) or would be marginal.

---

## MORE TESTS — DONE 2026-09-09 (round 2: bundle → program discovery)

### Method
Downloaded the beta.spout.finance Next.js bundle (12 static chunks, 808K), extracted every base58 program/mint reference, then checked each on **devnet + mainnet-beta** and resolved upgrade authorities from programdata.

### ⭐ CORRECTION — the headline lending product IS on-chain (I was wrong before)
Earlier I wrote "the borrow/lend/covered-call protocol is NOT the deployed devnet program." **Wrong** — I'd only examined `SPoRX` (the *orders* engine). The bundle surfaced two programs I'd missed, both live on devnet:
- **`spvaDgABYdpFKyatqo4Jr3nfwvVgWF5BbFxKFkzN3Am` = `spoutVault`** — the full borrow/lend/covered-call engine. Binary confirms instructions: InitializeVault, DepositCollateral, BorrowStablecoin, RepayDebt, TopUpCollateral, SeizeCollateral, SettleLiquidationDownside, **SettleAssignment**, SettlePoolBuyback, ExecuteVaultStrategy/UnwindVaultStrategy, InitializeLpPool, DepositLiquidity/WithdrawLiquidity, RecapitalizePool, DistributeYield, InitializeInsuranceFund/FundInsurance, CreateCollateralType. Health-factor liquidation (VaultUnhealthy/VaultStillHealthy), tranched LP pool, insurance fund, covered-call assignment ("Collateral type is locked: asset is committed to an off-chain strategy"; auto_buyback pref) — ALL on-chain.
- **`routARWktuwebKa7YrWrqJrCYsinGRFzsTuzHtPwgfn` = `freezeRouter`** — compliance thaw/freeze router (Initialize, AdminThaw, AdminFreeze, ThawVaultEscrow).
→ Fixed teardown Part 2.9 (now: 4-program map) + credited the code-enforced Turnkey/Squads split and the Token-2022-extension collateral guard. **This is the "verify before publishing" discipline catching a false claim before submission.**

### ⚠️ NEW mainnet finding — single-key upgrade authority on the live compliance program
- Only Spout-linked program on **mainnet-beta**: `TACLkU6CiCdkQN2MjoyDkVg2yAH9zkxiHDsiztQ52TP` (Token-ACL / compliance; its PDA `9D197p…` = spAsset freeze authority = KYC gate).
- mainnet upgrade authority = `DXtFpbPjcn2hxPnw79x1Pfoj35vXh5AsWBkS37YnXMVv` → **owner = System program (`111…`), ~1.997 SOL, 0 data = a plain hot keypair, NOT a multisig/PDA.**
- devnet upgrade authority is a *different* key (`Danxdge6…`) — good per-cluster hygiene.
- Caveat: can't confirm from outside whether TACL is Spout-owned or a shared/3rd-party compliance program (name = "Token Access Control List"). Flagged as please-confirm / mainnet-readiness, raised privately. → teardown Part 2.95.

### Separation-of-duties (devnet) sharpened
`7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp` is the upgrade authority for **BOTH** `SPoRX` (orders) **AND** `spva` (vault) AND is the operational keeper. One devnet key controls all program code + operations. (Placeholder state; binary shows intended mainnet split = Turnkey op + Squads gov.)

### Not findings (checked, clean — credit Spout)
- **No exposed sourcemaps** on beta.spout.finance (`*.js.map` → 404; no `sourceMappingURL` in chunks). No source leak.
- Mock oracle + KYC bypass both build-gated (`SPOUTVAULT_BUILD=mock-oracle`, `SPOUTORDERS_BUILD=devnet`).
- On-chain config/market accounts decode to sane params (staleness/skew windows ~60/300/3600s, USDC wired into escrow config) — corroborates oracle guards; exact field labels need the IDL, not asserted.

**Round-2 status: COMPLETE.** Net effect: one false claim corrected, one new mainnet finding, two design-credit items added.

---

## MORE TESTS — DONE 2026-09-09 (round 3: public GitHub repo `SpoutSolana/spout-finance`)

Public repo (not private), last push 2026-04-05. Read-only review. Ships IDLs + CI + app source.

### 🔒 PRIVATE — Finding A: committed PII (low sev)
`public/mailing-list.json` in the PUBLIC repo contains real email addresses (incl. what look like team members' personal gmail/hotmail + several test entries). The live `/api/mailing-list` endpoint actually writes to a **Google Sheet** (service account) — so this JSON is a stale committed export, but it's PII in public git history (persists even if deleted). → private note, low sev, "purge + git-filter-repo."

### 🔒 PRIVATE — Finding B: CI secret sprawl in `.github/workflows/deploy.yml` (medium sev) — my wheelhouse
- `${{ secrets.GH_PAT }}` (long-lived GitHub PAT with access to private `SpoutFinance/app-interface.git`) is **templated into a `deploy.sh` file** (Actions expands `${{ }}` before the shell, so the `<<'EOS'` quoting does NOT protect it — the raw token becomes literal file content).
- That file is then **`aws s3 cp`'d to S3** (`--sse AES256`, but still an object at rest) **and/or passed through SSM `send-command` parameters** → the PAT persists in **SSM command history + CloudTrail + the S3 object**, all OUTSIDE GitHub's secret-masking.
- `set -x` is enabled in deploy.sh and it runs `git clone "$REPO_HTTPS"` where REPO_HTTPS embeds the PAT → the token-bearing URL is printed to SSM stdout, which the workflow streams back into the Actions log.
- Static AWS access keys (`secrets.AWS_ACCESS_KEY_ID/SECRET`) instead of OIDC role assumption.
→ private disclosure. Fix: use a GitHub App/deploy token or OIDC (no PAT); never write secrets into a file shipped to S3/SSM; drop `set -x` around the clone; rotate the PAT + AWS keys. Not externally exploitable (push-to-main trigger), so infra-hardening, not a live breach.

### Public-safe technical notes (fold into teardown)
- **Oracle inconsistency:** deployed devnet programs use **Stork** (live 53s staleness confirmed; `stork_feed` / "Stork price" in vault binary), but the public frontend repo uses **Chainlink Data Streams** (`lib/solana/fetchChainlinkReport.ts`, `/api/chainlink/report`, on-chain `signed_report` arg). Components disagree — docs/site should state which oracle is authoritative at mainnet. → Part 6.
- **Issuer seize power at the instruction level:** the token IDL `idl/spoutsolana.json` (program `EkU7xRmBhVyHdwtRZ4SJ9D3Nz6SeAvymft7nz3CL2XXB`) exposes `force_transfer`, `force_transfer_2022`, `permissioned_transfer`, `mint`, `burn` — instruction-level confirmation of the permanent-delegate/freeze seize power. → strengthens Part 2.7.
- **EVM heritage (context, not a finding):** `.env.example` has Hardhat/Sepolia/mainnet chain IDs, Alchemy, RainbowKit; partners include Pharos/ERC-3643/Faroswap → Spout migrated from an EVM/Pharos build to Solana. Explains some doc mismatches (e.g. "transfer hook" is ERC-3643/EVM compliance language; Solana impl is freeze-gating).
- IDLs available for legitimate decoding: `idl/spoutorders.json`, `idl/spoutsolana.json`. (Vault program `spva` source is a private git submodule `SolanaVault` — not public.)

**Round-3 status: COMPLETE.** 2 private infra findings (PII, CI secrets) + 2 public-safe corroborations (oracle inconsistency, force_transfer). Public GitHub is the last big autonomous surface.
