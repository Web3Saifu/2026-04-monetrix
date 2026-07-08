# Sherlock Question Inventory

> Monetrix | generated from `x-ray/` + source spot-checks | branch `main` @ `3d94be1`

This file is a question/hypothesis queue, not a finding list. Nothing below is confirmed until the Reasoning phase proves reachability, impact, and false-positive resistance.

## Priority Queue

| ID | Priority | Status | Target |
|----|----------|--------|--------|
| SQ-001 | Critical | READY_FOR_REASONING | Multisig-vault L1 principal can be counted as `outstandingL1Principal` but bridge-back checks only Vault L1 balance |
| SQ-002 | Critical | READY_FOR_REASONING | Composite backing and surplus can be inflated or stale through HyperCore read assumptions |
| SQ-003 | High | READY_FOR_REASONING | PM supplied registry can become stale relative to actual 0x811 balances after withdraw/repair flows |
| SQ-004 | High | READY_FOR_REASONING | Redemption claim availability under pause/bridge stress |
| SQ-005 | High | READY_FOR_REASONING | sUSDM direct-donation / exchange-rate manipulation against new depositors or cooldown rounding |
| SQ-006 | Medium | READY_FOR_REASONING | Yield distribution mints USDM to sUSDM while splitting USDC to insurance/foundation |
| SQ-007 | Medium | READY_FOR_REASONING | Bridge/yield/principal unit conversions across 6dp EVM, 8dp L1, HLP, and perp account values |
| SQ-008 | Medium | TRUSTED_CONFIG_ROUTE | `maxTVL == 0`, config setters, and tradeable-asset removal while positions exist |
| SQ-009 | Medium | TRUSTED_ROLE_ROUTE | Emergency CoreWriter and emergency bridge paths bypass normal operator/user pause model |
| SQ-010 | Low | DUPLICATE_OR_KNOWN_ROUTE | uint64 bridge truncation and HLP decimal probes already have named regression tests |

---

## SQ-001 - Multisig-Vault L1 Principal Bridge-Back Asymmetry

**Root-cause theory:** `keeperBridge(BridgeTarget.Multisig)` can deposit USDC to `multisigVault` while increasing Vault-local `outstandingL1Principal`, but `bridgePrincipalFromL1()` and `_sendL1Bridge()` check/send only from `address(this)` on HyperCore.

**Evidence anchors:**
- `keeperBridge` chooses `recipient = multisigVault` when `target == BridgeTarget.Multisig && multisigVaultEnabled` and still executes `outstandingL1Principal += amount`: `src/core/MonetrixVault.sol:223-233`.
- `bridgePrincipalFromL1` only requires `amount <= outstandingL1Principal` then calls `_sendL1Bridge(amount)`: `src/core/MonetrixVault.sol:237-243`.
- `_sendL1Bridge` reads L1 USDC from `address(this)` and sends a Vault CoreWriter action: `src/core/MonetrixVault.sol:529-540`.
- Accountant separately includes `_readL1Backing(_multisigVault, multisigSupplied)` when `multisigVault != address(0)`: `src/core/MonetrixAccountant.sol:131-133`.

**Question:** If principal is bridged to `multisigVault`, can user redemptions later require that principal while the Vault contract cannot bridge it back through `bridgePrincipalFromL1`?

**Untrusted path to test:** user deposits -> operator bridges to Multisig target -> user requests redeem -> redemption shortfall exists -> operator attempts `bridgePrincipalFromL1`.

**False-positive filters:**
- Check whether the production design expects the multisig L1 account to manually bridge funds back outside Vault.
- Check whether `multisigVaultEnabled` is a migration-only trusted configuration that makes this out-of-scope.
- Check whether HyperCore `depositFor(multisigVault)` still credits Vault identity rather than recipient identity.

**Proof plan:** add or reuse simulator test with `multisigVaultEnabled = true`, target Multisig, forced L1 balances only on multisig account, then assert `bridgePrincipalFromL1` reverts on Vault-side L1 insufficiency.

**Initial route:** potential code bug if no external/manual bridge-back is part of intended design; otherwise `TRUSTED/CONFIG RISK`.

---

## SQ-002 - Composite Backing / Surplus External-State Assumption

**Root-cause theory:** `settleDailyPnL` allows yield only when `distributableSurplus()` says backing exceeds liabilities, but the backing includes HyperCore balances, supplied assets, account value, and HLP equity with no freshness or cross-source consistency check inside Solidity.

**Evidence anchors:**
- `totalBackingSigned()` sums Vault/RedeemEscrow EVM USDC, Vault L1, and multisig L1 backing: `src/core/MonetrixAccountant.sol:117-135`.
- `_readL1Backing` starts from signed perp account value and then adds spot, supplied, spot hedge, and HLP components: `src/core/MonetrixAccountant.sol:138-175`.
- `settleDailyPnL` gates `proposedYield <= distributableSurplus()` and annualized cap: `src/core/MonetrixAccountant.sol:200-224`.
- `PrecompileReader` checks success/length/positive price but cannot prove freshness or economic correctness: `src/core/PrecompileReader.sol:46-164`.

**Question:** Can a reachable stale, mismatched, or unit-inconsistent HyperCore read make `distributableSurplus()` positive when real backing is not sufficient?

**False-positive filters:**
- Do not classify external HyperCore correctness itself as a code bug.
- Require a concrete mismatched source or stale-read sequence, not “oracle may be wrong” in the abstract.
- Existing HLP/PM simulator tests may already cover specific unit cases.

**Proof plan:** enumerate every component in `_readL1Backing`, then build the smallest mocked-precompile scenario that makes `surplus()` diverge from an independent expected backing calculation.

**Initial route:** `NOT CONFIRMED` until one specific source mismatch is proven.

---

## SQ-003 - PM Supplied Registry Drift

**Root-cause theory:** Accountant iterates only registered supplied slots; Vault registers on `supplyToBlp` and sometimes `executeHedge`, but `withdrawFromBlp` does not remove or update registrations.

**Evidence anchors:**
- Accountant iterates `vaultSupplied` / `multisigSupplied` and calls strict supplied reads: `src/core/MonetrixAccountant.sol:146-157`.
- `supplyToBlp` calls `notifyVaultSupply`: `src/core/MonetrixVault.sol:333-344`.
- `withdrawFromBlp` only sends the withdraw action and emits an event: `src/core/MonetrixVault.sol:348-351`.
- Tests mention supplied-registry behavior in `test/simulator/SuppliedRegistry.t.sol`.

**Question:** Can stale supplied registry entries either revert Accountant reads, double-count returned spot balances, or create a persistent settlement DoS after a BLP withdraw?

**False-positive filters:**
- If 0x811 supplied read returns zero for inactive registered slots, stale entries may be harmless.
- If returned balances move into spot and supplied returns zero, there may be no double count.
- If this is an operator-maintained registry by design, classify as trusted-operator/config risk.

**Proof plan:** run/read `SuppliedRegistry.t.sol`; then add a direct test for `supplyToBlp -> withdrawFromBlp(max/zero amount) -> totalBackingSigned()`.

**Initial route:** proof candidate; likely `REJECTED` if simulator confirms zero-return semantics.

---

## SQ-004 - Redemption Claims During Pause and Bank-Run Stress

**Root-cause theory:** `claimRedeem` is `whenNotPaused`, while bridge-back/funding routes have mixed pause gates; in a stress event, Guardian pause may block user claims while operator/governor recovery paths differ.

**Evidence anchors:**
- `requestRedeem` and `claimRedeem` both use `whenNotPaused`: `src/core/MonetrixVault.sol:183-211`.
- `bridgePrincipalFromL1` is not `whenNotPaused` but is `whenOperatorNotPaused`: `src/core/MonetrixVault.sol:237-245`.
- `fundRedemptions` is operator-gated and not `whenNotPaused`: `src/core/MonetrixVault.sol:419-428`.
- `emergencyBridgePrincipalFromL1` is governor-only and bypasses operator pause: `src/core/MonetrixVault.sol:474-478`.

**Question:** Is there an untrusted griefing path that traps claimants, or is this purely Guardian/Governor trusted pause policy?

**False-positive filters:**
- Pause authority is a trusted role; do not submit “Guardian can pause” as a vulnerability by itself.
- Need an untrusted actor or unavoidable protocol state that causes claims to remain blocked after intended recovery.

**Proof plan:** source-trace all pause modifiers and write a table of claim/fund/bridge behavior under `paused`, `operatorPaused`, both, and neither.

**Initial route:** likely `TRUSTED/CONFIG RISK` unless a non-privileged permanent blockage is found.

---

## SQ-005 - sUSDM Direct Donation / Rate Manipulation

**Root-cause theory:** `sUSDM.totalAssets()` is `USDM.balanceOf(address(this))`, so direct USDM transfers alter the ERC4626 exchange rate outside `injectYield`; cooldown math snapshots that rate and burns/escrows based on current balance-derived assets.

**Evidence anchors:**
- `totalAssets()` returns raw USDM balance: `src/tokens/sUSDM.sol:102-103`.
- deposit/mint use OZ ERC4626 implementations with `_decimalsOffset() == 6`: `src/tokens/sUSDM.sol:106-127`.
- cooldown burns shares, increments `totalPendingClaims`, and escrows assets: `src/tokens/sUSDM.sol:160-228`.
- `injectYield` has an empty-vault guard, but direct transfer to sUSDM has no protocol-level hook: `src/tokens/sUSDM.sol:234-243`.

**Question:** Can a user profitably manipulate sUSDM share price or cooldown rounding through direct USDM donation, especially around first deposit, small deposits, or cooldownAssets rounding?

**False-positive filters:**
- Donations usually benefit existing shareholders and cost the donor; require profit, grief, or invariant break.
- OZ ERC4626 virtual offset may make first-depositor inflation non-profitable.
- Existing `sUSDM*` PoC/rate tests may already reject or document this.

**Proof plan:** run/read `sUSDMRate`, `sUSDMExchangeRateStrict`, and `sUSDMPoc` tests, then test attacker deposit/donate/victim deposit/attacker cooldown.

**Initial route:** `NOT CONFIRMED`; high-value because it is permissionless.

---

## SQ-006 - Yield Distribution Mint/Backing Symmetry

**Root-cause theory:** `distributeYield` pulls USDC from YieldEscrow, mints USDM for the user share into sUSDM, and transfers USDC for insurance/foundation shares; any split or ordering mismatch could create unbacked USDM or wrong yield accounting.

**Evidence anchors:**
- `settle` moves `proposedYield` USDC into YieldEscrow after Accountant gates: `src/core/MonetrixVault.sol:364-373`.
- `distributeYield` pulls all YieldEscrow USDC, computes shares, mints USDM for `userShare`, and sends USDC to InsuranceFund/Foundation: `src/core/MonetrixVault.sol:377-414`.
- Config enforces yield BPS sum to 100%: `src/core/MonetrixConfig.sol:96-99`.

**Question:** Across all split combinations and rounding residues, does USDM supply increase exactly when Vault keeps corresponding USDC backing?

**False-positive filters:**
- `foundationShare = totalYield - userShare - insuranceShare` intentionally absorbs rounding.
- Insurance/Foundation USDC outflows should not mint USDM.

**Proof plan:** property test all BPS splits and yield values; assert post-distribution backing delta equals USDM supply delta for user share.

**Initial route:** proof candidate, likely `REJECTED` if current split accounting is symmetric.

---

## SQ-007 - Decimal / uint64 Conversion Boundary

**Root-cause theory:** HyperCore paths mix EVM 6-decimal USDC, L1 8-decimal spot wei, perp 6-decimal account values, HLP equity, and token-specific decimals; a single wrong conversion can overstate backing or move the wrong L1 amount.

**Evidence anchors:**
- `TokenMath.usdcEvmToL1Wei` multiplies by 100 and SafeCasts to uint64: `src/core/TokenMath.sol:19-20`.
- generic token conversion floors on division: `src/core/TokenMath.sol:31-50`.
- spot notional formulas depend on `weiDecimals` and `szDecimals`: `src/core/TokenMath.sol:59-84`.
- `ActionEncoder.sendBridgeToL1` converts EVM amount before CoreWriter payload: `src/core/ActionEncoder.sol:177`.

**Question:** Is there any conversion path where floor rounding, decimals metadata, or uint64 bounds can be attacker-favorable rather than conservative?

**False-positive filters:**
- Existing tests named `BridgeUint64Truncation`, `HlpEquityDecimalProbe`, `PrecisionBugPoC`, and `TokenMathFuzz` likely cover historical bugs.
- Treat covered historical failures as `DUPLICATE/KNOWN` unless a new path remains untested.

**Proof plan:** map every call into TokenMath and compare against simulator/PoC coverage; only create a new card for uncovered paths.

**Initial route:** likely `DUPLICATE/KNOWN` for uint64/HLP; open for non-USDC spot token decimals.

---

## SQ-008 - Governance-Set Limit Removal and Asset Whitelist Mutation

**Root-cause theory:** `setMaxTVL(0)` disables the TVL cap, and tradeable assets can be removed while off-chain/L1 positions may still exist.

**Evidence anchors:**
- `setMaxTVL` has no lower/upper bound: `src/core/MonetrixConfig.sol:111-112`.
- Vault applies the cap only when `maxTVL > 0`: `src/core/MonetrixVault.sol:174-176`.
- `removeTradeableAsset` deletes whitelist mappings and array entry: `src/core/MonetrixConfig.sol:188-209`.

**Question:** Is there any untrusted path to bypass limits, or are these deliberate Governor policy controls?

**False-positive filters:** Governor is trusted/timelocked per README; absent an untrusted path, classify as `TRUSTED/CONFIG RISK`.

**Proof plan:** do not write PoC first; source-trace role/timelock assumptions and check if active positions can be made unrecoverable by asset removal.

**Initial route:** `TRUSTED/CONFIG RISK`.

---

## SQ-009 - Emergency Paths Bypass Normal Pause Model

**Root-cause theory:** Governor can send arbitrary CoreWriter actions and bridge principal from L1 without the normal operator pause or user pause gates.

**Evidence anchors:**
- `emergencyRawAction(bytes)` sends arbitrary CoreWriter payload: `src/core/MonetrixVault.sol:469-471`.
- `emergencyBridgePrincipalFromL1` decreases `outstandingL1Principal` and calls `_sendL1Bridge`: `src/core/MonetrixVault.sol:474-478`.
- Normal operator bridge-back is `whenOperatorNotPaused`: `src/core/MonetrixVault.sol:237-245`.

**Question:** Is this a necessary recovery authority, or can a lower-trust actor reach it?

**False-positive filters:** Governor is trusted/timelocked per README; do not classify privileged misuse as an untrusted exploit.

**Proof plan:** verify ACL role holder and timelock assumptions from deployment docs/tests; only escalate if Governor role can be bypassed.

**Initial route:** `TRUSTED/CONFIG RISK`.

---

## SQ-010 - Historical Regression Routes

**Root-cause theory:** Several test names indicate known or previously fixed issues around truncation, precision, HLP decimals, bridge async behavior, and sUSDM rate behavior.

**Evidence anchors:**
- `test/simulator/BridgeUint64Truncation.t.sol`
- `test/simulator/HlpEquityDecimalProbe.t.sol`
- `test/PrecisionBugPoC.t.sol`
- `test/VaultUint64TruncationPoC.t.sol`
- `test/sUSDMRate.t.sol`, `test/sUSDMExchangeRateStrict.t.sol`, `test/sUSDMPoc.t.sol`

**Question:** Are any of these still failing or incomplete in current source?

**False-positive filters:** Do not resubmit a known fixed/covered issue. Use these tests as regression evidence or as templates for new variants.

**Proof plan:** after dependencies are restored, run the named tests first and compare failure messages to the question cards above.

**Initial route:** `DUPLICATE/KNOWN` unless a new uncovered variant is isolated.

---

## Recommended Reasoning Order

1. SQ-001 - highest asymmetry: accounting and bridge-back may disagree across Vault vs multisig L1 identities.
2. SQ-002 - most protocol-critical invariant: surplus/yield correctness.
3. SQ-005 - highest untrusted-user surface: sUSDM donation/rate/cooldown behavior.
4. SQ-003 - PM registry drift is narrow and likely testable with existing simulator patterns.
5. SQ-006 - yield distribution accounting is compact and can be proven/rejected with a small property test.

## Environment Blockers

- Current checkout cannot compile because `lib` submodules/dependencies are missing, so the first proof step may require `git submodule update --init --recursive`.
- WSL/bash is unavailable on this machine; use PowerShell and Foundry directly.

