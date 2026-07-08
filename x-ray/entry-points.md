# Entry Points

> Monetrix entry-point map, grep-verified from `src/**/*.sol` and filtered for non-view/non-pure public or external functions.

## Protocol Flow Paths

```text
User deposit
  -> MonetrixVault.deposit()
  -> USDC.safeTransferFrom(user, Vault)
  -> USDM.mint(user)

User redeem
  -> MonetrixVault.requestRedeem()
  -> USDM.safeTransferFrom(user, Vault)
  -> RedeemEscrow.addObligation()
  -- time: redeemCooldown
  -> MonetrixVault.claimRedeem()
  -> USDM.burn()
  -> RedeemEscrow.payOut(user)

User stake / unstake
  -> sUSDM.deposit() / sUSDM.mint()
  -> ERC4626 share mint
  -> sUSDM.cooldownShares() / cooldownAssets()
  -> sUSDMEscrow.deposit()
  -- time: unstakeCooldown
  -> sUSDM.claimUnstake()
  -> sUSDMEscrow.release(user)

Operator yield
  -> MonetrixVault.settle()
  -> MonetrixAccountant.settleDailyPnL()
  -> YieldEscrow receives USDC
  -> MonetrixVault.distributeYield()
  -> sUSDM.injectYield() + InsuranceFund/Foundation transfers

Operator L1 strategy
  -> MonetrixVault.keeperBridge()
  -> CoreDepositWallet.depositFor()
  -> MonetrixVault.executeHedge()/closeHedge()/repairHedge()
  -> ActionEncoder -> HyperCore CoreWriter
```

---

## Permissionless

### `MonetrixVault.deposit(uint256 amount)`

| Aspect | Detail |
|--------|--------|
| Visibility | external, nonReentrant, whenNotPaused |
| Caller | User |
| Parameters | `amount` (user-controlled) |
| Call chain | `MonetrixVault.deposit -> USDC.safeTransferFrom -> USDM.mint` |
| State modified | USDM supply/balance via `mint`; Vault USDC balance |
| Value flow | USDC in, USDM out |
| Reentrancy guard | yes |

### `MonetrixVault.requestRedeem(uint256 usdmAmount)`

| Aspect | Detail |
|--------|--------|
| Visibility | external, nonReentrant, whenNotPaused, requireWired |
| Caller | User |
| Parameters | `usdmAmount` (user-controlled) |
| Call chain | `requestRedeem -> USDM.safeTransferFrom -> RedeemEscrow.addObligation` |
| State modified | `nextRedeemId`, `redeemRequests`, `_userRedeemIds`, `RedeemEscrow.totalOwed` |
| Value flow | USDM user -> Vault; redemption obligation created |
| Reentrancy guard | yes |

### `MonetrixVault.claimRedeem(uint256 requestId)`

| Aspect | Detail |
|--------|--------|
| Visibility | external, nonReentrant, whenNotPaused, requireWired |
| Caller | Redeem request owner |
| Parameters | `requestId` (user-controlled id, ownership checked) |
| Call chain | `claimRedeem -> USDM.burn -> RedeemEscrow.payOut` |
| State modified | `redeemRequests`, `_userRedeemIds`, `RedeemEscrow.totalOwed`, USDM supply |
| Value flow | USDC escrow -> user |
| Reentrancy guard | yes |

### `InsuranceFund.deposit(uint256 amount)`

| Aspect | Detail |
|--------|--------|
| Visibility | external |
| Caller | Anyone |
| Parameters | `amount` (user-controlled) |
| Call chain | `InsuranceFund.deposit -> USDC.safeTransferFrom` |
| State modified | `totalDeposited` |
| Value flow | USDC in |
| Reentrancy guard | no |

### `sUSDM.deposit(uint256 assets, address receiver)` / `sUSDM.mint(uint256 shares, address receiver)`

| Aspect | Detail |
|--------|--------|
| Visibility | public, nonReentrant, whenNotPaused |
| Caller | USDM holder |
| Parameters | `assets/shares`, `receiver` (user-controlled) |
| Call chain | `sUSDM.deposit/mint -> ERC4626Upgradeable` |
| State modified | sUSDM shares, USDM balance |
| Value flow | USDM in, sUSDM shares out |
| Reentrancy guard | yes |

### `sUSDM.cooldownShares(uint256 shares)` / `sUSDM.cooldownAssets(uint256 assets)`

| Aspect | Detail |
|--------|--------|
| Visibility | external, nonReentrant, whenNotPaused |
| Caller | sUSDM holder |
| Parameters | `shares/assets` (user-controlled) |
| Call chain | `cooldown -> _burn -> sUSDMEscrow.deposit` |
| State modified | `totalPendingClaims`, `nextRequestId`, `unstakeRequests`, `_userUnstakeIds`, sUSDM supply |
| Value flow | USDM from sUSDM vault -> unstake escrow |
| Reentrancy guard | yes |

### `sUSDM.claimUnstake(uint256 requestId)`

| Aspect | Detail |
|--------|--------|
| Visibility | external, nonReentrant, whenNotPaused |
| Caller | Unstake request owner |
| Parameters | `requestId` (user-controlled id, ownership checked) |
| Call chain | `claimUnstake -> sUSDMEscrow.release` |
| State modified | `unstakeRequests`, `_userUnstakeIds`, `totalPendingClaims` |
| Value flow | USDM escrow -> user |
| Reentrancy guard | yes |

---

## Role-Gated

### `OPERATOR`

| Contract | Function | State / value effect |
|----------|----------|----------------------|
| MonetrixVault | `keeperBridge(BridgeTarget)` | Updates `outstandingL1Principal`, `lastBridgeTimestamp`; deposits USDC to HyperCore wallet |
| MonetrixVault | `bridgePrincipalFromL1(uint256)` | Decreases `outstandingL1Principal`; sends HyperCore bridge action |
| MonetrixVault | `bridgeYieldFromL1(uint256)` | Sends HyperCore bridge action for yield shortfall |
| MonetrixVault | `executeHedge(HedgeParams)` | Sends spot/perp CoreWriter actions; may notify Accountant PM supply |
| MonetrixVault | `closeHedge(CloseParams)` | Sends close actions to HyperCore |
| MonetrixVault | `repairHedge(uint256, RepairParams)` | Sends single-leg repair action |
| MonetrixVault | `depositToHLP(uint64)` / `withdrawFromHLP(uint64)` | Sends HLP vault transfer actions |
| MonetrixVault | `supplyToBlp(uint64,uint64)` / `withdrawFromBlp(uint64,uint64)` | Sends BLP supply/withdraw actions; may notify Accountant |
| MonetrixVault | `settle(uint256)` | Moves USDC to YieldEscrow after Accountant gates |
| MonetrixVault | `distributeYield(uint256)` | Pulls yield, mints USDM to sUSDM, pays Insurance/Foundation shares |
| MonetrixVault | `fundRedemptions(uint256)` / `reclaimFromRedeemEscrow(uint256)` | Moves USDC between Vault and RedeemEscrow |

### `GOVERNOR`

| Contract | Function | State Modified |
|----------|----------|----------------|
| MonetrixConfig | `setYieldBps`, `setDepositLimits`, `setMaxTVL`, `setBridgeInterval`, `setCooldowns`, `setInsuranceFund`, `setFoundation`, `setMaxYieldPerInjection`, `setMaxAnnualYieldBps`, `add/removeTradeableAsset` | Protocol parameters and asset whitelist |
| MonetrixVault | `setAccountant`, `setMultisigVault`, `setMultisigVaultEnabled`, `setRedeemEscrow`, `setYieldEscrow`, `setBridgeRetentionAmount`, `setPmEnabled`, `emergencyRawAction`, `emergencyBridgePrincipalFromL1` | Wiring, emergency paths, PM/bridge settings |
| MonetrixAccountant | `setConfig`, `setMinSettlementInterval`, `addMultisigSupply`, `removeSuppliedEntry`, `initializeSettlement` | Settlement config and supplied registries |
| USDM / sUSDM | `setVault`, `setConfig`, `setEscrow` | One-time authority wiring and config |
| InsuranceFund | `withdraw` | Sends reserve USDC |

### `GUARDIAN`

| Contract | Function | State Modified |
|----------|----------|----------------|
| MonetrixVault | `pause`, `unpause`, `pauseOperator`, `unpauseOperator` | User pause and operator pause |
| USDM / sUSDM | `pause`, `unpause` | Token transfer/user flow pause |

### Contract-only gates

| Contract | Function | Caller |
|----------|----------|--------|
| USDM | `mint`, `burn` | MonetrixVault |
| sUSDM | `injectYield` | MonetrixVault |
| RedeemEscrow | `addObligation`, `payOut`, `reclaimTo` | MonetrixVault |
| YieldEscrow | `pullForDistribution` | MonetrixVault |
| sUSDMEscrow | `deposit`, `release` | sUSDM |

---

## Initialization

All upgradeable contracts expose `initialize(...) external initializer`; constructors call `_disableInitializers()`. UUPS upgrades for governed contracts route through `MonetrixGovernedUpgradeable._authorizeUpgrade`, which checks `UPGRADER`; the ACL itself checks `DEFAULT_ADMIN_ROLE`.
