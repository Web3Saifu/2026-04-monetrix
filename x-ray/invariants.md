# Invariant Map

> Monetrix | 20 guards | 9 inferred | 2 not enforced fully on-chain

---

## 1. Enforced Guards (Reference)

Per-call preconditions. Heading IDs below are anchor targets from x-ray.md attack surfaces.

#### G-1
`amount >= config.minDepositAmount() && amount <= config.maxDepositAmount()` · `src/core/MonetrixVault.sol:170` · protects the user mint path from deposits outside governance-set bounds.

#### G-2
`usdm.totalSupply() + amount <= maxTVL` · `src/core/MonetrixVault.sol:176` · caps aggregate USDM liability when a TVL cap is active.

#### G-3
`usdmAmount > 0` · `src/core/MonetrixVault.sol:184` · prevents zero-value redemption obligations.

#### G-4
`req.usdmAmount > 0 && msg.sender == req.owner && block.timestamp >= req.cooldownEnd` · `src/core/MonetrixVault.sol:200` · binds redemption claims to the request owner and cooldown.

#### G-5
`block.timestamp >= lastBridgeTimestamp + config.bridgeInterval()` · `src/core/MonetrixVault.sol:224` · rate-limits operator bridging to HyperCore.

#### G-6
`amount > 0 && amount <= redemptionShortfall() && amount <= outstandingL1Principal` · `src/core/MonetrixVault.sol:238` · limits principal bridge-back to redemption need and recorded L1 principal.

#### G-7
`proposedYield > 0` · `src/core/MonetrixVault.sol:365` · prevents empty yield settlements.

#### G-8
`available >= proposedYield` · `src/core/MonetrixVault.sol:370` · requires EVM USDC before moving yield to escrow.

#### G-9
`lastSettlementTime > 0` · `src/core/MonetrixAccountant.sol:206` · requires explicit settlement initialization before yield declaration.

#### G-10
`block.timestamp >= lastSettlementTime + minSettlementInterval` · `src/core/MonetrixAccountant.sol:208` · enforces the minimum time interval between settlements.

#### G-11
`proposedYield <= ds` · `src/core/MonetrixAccountant.sol:216` · caps declared yield by distributable surplus.

#### G-12
`proposedYield <= cap` · `src/core/MonetrixAccountant.sol:221` · caps declared yield by annualized APR.

#### G-13
`interval > 0` and `interval <= 1 days` · `src/core/MonetrixAccountant.sol:238` · bounds the settlement interval set by governance.

#### G-14
`_userBps + _insuranceBps + _foundationBps == BPS_DENOMINATOR` · `src/core/MonetrixConfig.sol:97` · keeps yield split shares summing to 100%.

#### G-15
`_bps <= MAX_ANNUAL_YIELD_BPS_CAP` · `src/core/MonetrixConfig.sol:153` · hard-caps governance-set APR.

#### G-16
`bal >= amount + totalOwed` · `src/core/RedeemEscrow.sol:62` · prevents reclaiming USDC that backs pending redemptions.

#### G-17
`shares > 0` and `balanceOf(msg.sender) >= shares` · `src/tokens/sUSDM.sol:161` · ensures cooldown share requests burn real user shares.

#### G-18
`assets > 0` and `balanceOf(msg.sender) >= shares` · `src/tokens/sUSDM.sol:192` · ensures asset-denominated cooldown requests resolve to burnable shares.

#### G-19
`block.timestamp < req.cooldownEnd` revert path · `src/tokens/sUSDM.sol:223` · enforces unstake cooldown before escrow release.

#### G-20
`usdmAmount <= config.maxYieldPerInjection()` and `totalSupply() > 0` · `src/tokens/sUSDM.sol:236` · caps yield injection and blocks empty-vault yield capture.

---

## 2. Inferred Invariants (Single-Contract)

#### I-1

`Conservation` · On-chain: **Yes**

> `RedeemEscrow.totalOwed` tracks outstanding redemption obligations.

**Derivation** - delta-pair: `totalOwed += amount` at `src/core/RedeemEscrow.sol:42` and `totalOwed -= amount` at `src/core/RedeemEscrow.sol:50`; Vault creates obligations in `requestRedeem` and pays them in `claimRedeem`.

**If violated** - pending redemption accounting diverges from escrow funding requirements.

---

#### I-2

`Bound` · On-chain: **Yes**

> Redeem escrow reclaim cannot reduce USDC below outstanding obligations.

**Derivation** - guard-lift: `bal >= amount + totalOwed` at `src/core/RedeemEscrow.sol:62`; the only reclaim path is `reclaimTo`.

**If violated** - already-queued redemptions could be underfunded.

---

#### I-3

`Conservation` · On-chain: **Yes**

> sUSDM pending claims move with escrowed USDM during cooldown and claim.

**Derivation** - delta-pair: `totalPendingClaims += assets` with `escrow.deposit(assets)` at `src/tokens/sUSDM.sol:171-172` and `src/tokens/sUSDM.sol:202-203`; `totalPendingClaims -= req.usdmAmount` with `escrow.release(...)` at `src/tokens/sUSDM.sol:227-228`.

**If violated** - unstake claim accounting and escrow balance can drift.

---

#### I-4

`Temporal` · On-chain: **Yes**

> Yield settlement cannot run before initialization and cannot run faster than `minSettlementInterval`.

**Derivation** - temporal: `lastSettlementTime > 0` at `src/core/MonetrixAccountant.sol:206`, `block.timestamp >= lastSettlementTime + minSettlementInterval` at `src/core/MonetrixAccountant.sol:208`, and update at `src/core/MonetrixAccountant.sol:223`.

**If violated** - yield cadence could be compressed.

---

#### I-5

`Bound` · On-chain: **Yes**

> Declared yield is bounded by both distributable surplus and annualized APR cap.

**Derivation** - guard-lift: `proposedYield <= ds` at `src/core/MonetrixAccountant.sol:216`, `proposedYield <= cap` at `src/core/MonetrixAccountant.sol:221`, and `totalSettledYield += proposedYield` at `src/core/MonetrixAccountant.sol:224`.

**If violated** - sUSDM yield can be declared from unearned or over-rate surplus.

---

#### I-6

`Bound` · On-chain: **Yes**

> Configured annual yield BPS is hard-capped by `MAX_ANNUAL_YIELD_BPS_CAP`.

**Derivation** - guard-lift: `require(_bps <= MAX_ANNUAL_YIELD_BPS_CAP, "Config: exceeds hard cap")` at `src/core/MonetrixConfig.sol:153`; only `setMaxAnnualYieldBps` writes `maxAnnualYieldBps`.

**If violated** - governance could set an overlarge rate without upgrade.

---

#### I-7

`StateMachine` · On-chain: **Yes**

> USDM and sUSDM vault addresses are one-time bindings.

**Derivation** - edge: `vault == address(0)` checked at `src/tokens/USDM.sol:41` then written at `src/tokens/USDM.sol:42`; same pattern at `src/tokens/sUSDM.sol:270-271`.

**If violated** - mint/burn or yield injection authority could be redirected.

---

#### I-8

`StateMachine` · On-chain: **Yes**

> sUSDM escrow address is a one-time binding and must point back to the same sUSDM and USDM asset.

**Derivation** - edge: `address(escrow) == address(0)` at `src/tokens/sUSDM.sol:259`, identity checks at `src/tokens/sUSDM.sol:260-261`, write at `src/tokens/sUSDM.sol:262`.

**If violated** - cooldown funds could move through the wrong escrow.

---

#### I-9

`Bound` · On-chain: **No**

> Deposit limits and cooldowns are bounded, but max TVL can be set to zero to disable the TVL cap.

**Derivation** - write-site enumeration: `setDepositLimits` enforces `_min > 0` and `_min < _max` at `src/core/MonetrixConfig.sol:104-105`; `setCooldowns` enforces 1 minute to 30 days at `src/core/MonetrixConfig.sol:124-127`; `setMaxTVL` writes `maxTVL` without lower/upper bound at `src/core/MonetrixConfig.sol:111-112`, and Vault treats `maxTVL == 0` as uncapped at `src/core/MonetrixVault.sol:174-176`.

**If violated** - the TVL limiter becomes governance policy rather than a hard invariant.

---

## 3. Inferred Invariants (Cross-Contract)

#### X-1

On-chain: **No**

> Accountant backing assumes HyperCore precompile responses are current, correctly decoded, and economically truthful.

**Caller side** - `src/core/MonetrixAccountant.sol:117-187` uses PrecompileReader-derived balances and account values in backing, surplus, and distributable surplus.

**Callee side** - `src/core/PrecompileReader.sol:46-164` validates call success, response length, and positive prices but cannot prove freshness or external L1 state correctness.

**If violated** - surplus-gated yield can be calculated from bad external state.

---

#### X-2

On-chain: **Yes**

> Vault redemption request and RedeemEscrow obligation accounting are coupled through the Vault-only escrow interface.

**Caller side** - `src/core/MonetrixVault.sol:183-211` transfers USDM, adds escrow obligation, then burns and pays on claim.

**Callee side** - `src/core/RedeemEscrow.sol:41-62` restricts obligation mutation and payout/reclaim to `onlyVault`.

**If violated** - user redemption state could diverge from escrow liabilities.

---

#### X-3

On-chain: **Yes**

> sUSDM cooldown requests rely on sUSDMEscrow accepting calls only from the bound sUSDM contract.

**Caller side** - `src/tokens/sUSDM.sol:170-228` burns shares, updates pending claims, and calls escrow deposit/release.

**Callee side** - `src/tokens/sUSDMEscrow.sol:21-39` gates both deposit and release with `onlySUSDM`.

**If violated** - external callers could move escrowed USDM outside the cooldown lifecycle.

---

## 4. Economic Invariants

#### E-1

On-chain: **No**

> USDM solvency depends on internal USDC plus external HyperCore backing being at least USDM supply.

**Follows from** - `I-5` + `X-1`

**If violated** - USDM liabilities can exceed recognized backing.

---

#### E-2

On-chain: **Yes**

> Pending redemptions remain fully claim-denominated while funded escrow liquidity exists.

**Follows from** - `I-1` + `I-2` + `X-2`

**If violated** - queued claimants could face accounting or funding mismatch.

---

#### E-3

On-chain: **Yes**

> sUSDM cooldown claims are isolated from live sUSDM vault assets after cooldown creation.

**Follows from** - `I-3` + `X-3`

**If violated** - unstakers and remaining stakers could share the same assets incorrectly.
