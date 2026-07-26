# X-Ray Report

> Monetrix | 1726 in-scope nSLOC per README scope | `3d94be1` (`main`) | Foundry | 08/07/26

---


## 1. Protocol Overview

**What it does:** Monetrix is a USDC-backed synthetic dollar system on HyperEVM with USDM mint/redeem, sUSDM staking, HyperCore strategy operations, and accounting-gated yield distribution.

- **Users**: deposit USDC for USDM, request async redemptions, and stake USDM into sUSDM.
- **Core flow**: Vault mints/burns USDM while operators bridge/hedge on HyperCore and settle declared yield through Accountant.
- **Key mechanism**: stablecoin plus ERC4626-style staking wrapper, backed by EVM USDC, escrow balances, HyperCore L1 balances, supplied assets, perps, and HLP equity.
- **Token model**: USDM is 6-decimal stablecoin; sUSDM is yield-bearing staking share token with cooldown-based unstaking.
- **Admin model**: shared ACL roles split DEFAULT_ADMIN, UPGRADER, GOVERNOR, OPERATOR, and GUARDIAN; README states timelock assumptions for admin/governor/upgrader roles.

For a visual overview of the protocol's architecture, see the [architecture diagram](architecture.svg).

### Contracts in Scope

| Subsystem | Key Contracts | nSLOC | Role |
|-----------|--------------|------:|------|
| Core vault/accounting | MonetrixVault, MonetrixAccountant, MonetrixConfig | 820 | Deposit/redeem hub, backing reads, yield gates, parameters |
| Escrows/reserves | RedeemEscrow, YieldEscrow, InsuranceFund | 125 | Redemption obligations, yield custody, insurance reserve |
| Tokens | USDM, sUSDM, sUSDMEscrow | 310 | Stablecoin, staking vault, unstake escrow |
| HyperCore helpers | ActionEncoder, PrecompileReader, TokenMath | 321 | L1 action encoding, precompile reads, decimal conversions |
| Governance | MonetrixAccessController, MonetrixGovernedUpgradeable | 73 | Shared ACL and UUPS authorization |

### Backwards-Compatibility Code

- `MonetrixAccountant.lastSurplusSnapshot` - marked deprecated and retained for storage layout stability at `src/core/MonetrixAccountant.sol:56`.

### How It Fits Together

The core trick: USDM liabilities are minted on EVM while material backing and yield state can live across HyperCore venues, so Accountant is the bridge between external backing and on-chain yield recognition.

### Deposit and Mint

```text
User
└─ MonetrixVault.deposit(amount)
   ├─ checks Config min/max deposit and optional maxTVL
   ├─ pulls USDC from user
   └─ USDM.mint(user, amount)
```

### Redemption Queue

```text
User
├─ MonetrixVault.requestRedeem(usdmAmount)
│  ├─ pulls USDM into Vault
│  ├─ RedeemEscrow.addObligation(usdmAmount)
│  └─ stores owner, amount, cooldownEnd
└─ MonetrixVault.claimRedeem(requestId)
   ├─ checks owner and cooldown
   ├─ USDM.burn(amount)
   └─ RedeemEscrow.payOut(user, amount)
```

### Yield Settlement

```text
Operator
└─ MonetrixVault.settle(proposedYield)
   ├─ Accountant.settleDailyPnL(proposedYield)
   │  ├─ reads HyperCore balances and account values
   │  └─ enforces interval, surplus, and APR gates
   └─ transfers USDC into YieldEscrow
```

### sUSDM Cooldown

```text
User
├─ sUSDM.deposit/mint
└─ sUSDM.cooldownShares/cooldownAssets
   ├─ burns shares
   ├─ increments totalPendingClaims
   └─ sUSDMEscrow.deposit(assets)
      └─ claimUnstake releases after cooldown
```

---

## 2. Threat & Trust Model

### Protocol Threat Profile

> Protocol classified as: **Stablecoin** with **Yield Aggregator / Vault** and **Derivatives/Perps integration** characteristics

The code mints/burns USDM against composite backing and routes yield into an ERC4626-style staking token; HyperCore spot/perp/HLP/BLP reads and actions make external venue accounting central to solvency and yield gates.

### Actors & Adversary Model

| Actor | Trust Level | Capabilities |
|-------|-------------|--------------|
| User | Untrusted | Deposit, request/claim redemption, stake/unstake sUSDM, donate to InsuranceFund |
| OPERATOR | Bounded but trusted hot path | Instant bridge, hedge, HLP/BLP, yield settlement/distribution, redemption funding/reclaim; bounded mostly by destination wiring and pause flags |
| GOVERNOR | Trusted timelocked per README | Sets config, wiring, PM flag, bridge retention, emergency actions, InsuranceFund withdrawals |
| UPGRADER | Trusted timelocked per README | UUPS upgrades for governed contracts through shared base |
| DEFAULT_ADMIN | Trusted timelocked per README | ACL role administration and ACL upgrade |
| GUARDIAN | Bounded instant role | Pauses Vault user flows, operator flows, USDM, and sUSDM; no direct fund transfer authority observed |
| Vault contract | Trusted component | Sole minter/burner for USDM, yield injector for sUSDM, and mover for Redeem/Yield escrow funds |

**Adversary Ranking**

1. **External user / first depositor** - relevant to deposit/mint/redeem and ERC4626-style share accounting.
2. **Operator compromise or bad automation** - relevant because most HyperCore movement and yield distribution is instant operator-gated.
3. **External venue/oracle fault** - relevant because Accountant relies on HyperCore precompile state for backing and surplus.
4. **Governance/upgrader compromise** - relevant because wiring, parameters, emergency paths, and implementation upgrades control the protocol envelope.
5. **Bank-run participant** - relevant because redemption funding depends on EVM liquidity plus bridge-back and escrow coverage.

See [entry-points.md](entry-points.md) for the full permissionless entry point map.

### Trust Boundaries

- **EVM Vault to HyperCore** - `keeperBridge`, hedge, HLP, BLP, and bridge-back paths cross from Solidity checks into CoreWriter/precompile semantics at `src/core/MonetrixVault.sol:223-349`.

- **Accountant to external backing** - `totalBackingSigned` and `distributableSurplus` depend on HyperCore reads that validate decoding but not economic freshness at `src/core/MonetrixAccountant.sol:117-187`.

- **Vault to escrows** - redemptions and yield custody are protected by `onlyVault`, so Vault compromise or upgrade controls escrow movement at `src/core/RedeemEscrow.sol:41-62` and `src/core/YieldEscrow.sol:37-40`.

- **Shared ACL to all governed contracts** - every governed modifier defers to one ACL, and UUPS authorization is centralized at `src/governance/MonetrixGovernedUpgradeable.sol:27-66`.

### Key Attack Surfaces

- **Composite backing and surplus calculation** &nbsp;[[X-1](invariants.md#x-1), [E-1](invariants.md#e-1)] - `MonetrixAccountant.totalBackingSigned` combines EVM balances with HyperCore precompile-derived signed values; worth tracing every unit and freshness assumption.

- **Operator-driven HyperCore movement** - `MonetrixVault.keeperBridge`, hedge, HLP, BLP, and bridge-back functions share instant OPERATOR authority; worth checking pause coverage and destination constraints across all paths.

- **Yield settlement gate composition** &nbsp;[[I-4](invariants.md#i-4), [I-5](invariants.md#i-5)] - `settleDailyPnL` has interval, surplus, and APR gates; worth checking whether every input to those gates has the same unit and timing model.

- **Redemption queue liquidity accounting** &nbsp;[[I-1](invariants.md#i-1), [I-2](invariants.md#i-2), [X-2](invariants.md#x-2)] - request/claim state crosses Vault and RedeemEscrow; worth checking all edge cases where EVM USDC is away on L1.

- **sUSDM rate and cooldown isolation** &nbsp;[[I-3](invariants.md#i-3), [X-3](invariants.md#x-3)] - cooldown burns shares and physically escrows assets; worth checking rounding, donation, and paused-state behavior.

- **Decimal and uint64 conversion boundary** - `TokenMath` and `ActionEncoder` convert between 6-decimal EVM USDC, 8-decimal L1 wei, spot units, and CoreWriter `uint64` payloads.

- **Governance-set limits with deliberate uncapped modes** &nbsp;[[I-9](invariants.md#i-9)] - config setters bound many limits, but `maxTVL == 0` disables the TVL cap and role assumptions become policy-level controls.

### Upgrade Architecture Concerns

- **Shared UUPS gate** - governed contracts inherit `_authorizeUpgrade` from `MonetrixGovernedUpgradeable`, while ACL upgrades use `DEFAULT_ADMIN_ROLE` directly at `src/governance/MonetrixAccessController.sol:67`.
- **Storage layout evolution** - deprecated `lastSurplusSnapshot` and multiple `__gap` comments indicate prior versioning; storage layout should stay a first-class upgrade test target.

### Protocol-Type Concerns

**As a Stablecoin:**
- `MonetrixAccountant.surplus` compares composite signed backing against `USDM.totalSupply`; the external backing terms deserve the most scrutiny.
- `MonetrixVault.deposit` mints 1:1 from received USDC and does not call Accountant, so peg protection is primarily settlement/redemption-side.

**As a Yield Vault:**
- `sUSDM.totalAssets()` reads token balance directly at `src/tokens/sUSDM.sol:102`, so direct USDM transfers affect exchange-rate math.
- `sUSDM.injectYield` blocks empty-vault yield and caps injection at `src/tokens/sUSDM.sol:234-243`.

### Temporal Risk Profile

**Deployment & Initialization:**
- One-time wiring gates exist for USDM/sUSDM vault and sUSDM escrow, but deployment order matters because Vault `requireWired` blocks redemption/yield paths until escrows/accountant are set.

**Market Stress:**
- Redemption stress moves across `redemptionShortfall`, bridge-back, and RedeemEscrow funding; the code has accounting guards, but L1 bridge timing remains an operational dependency.

### Composability & Dependency Risks

> **HyperCore precompiles** - via `PrecompileReader` and `MonetrixAccountant`
> - Assumes: decoded balances, prices, supplied balances, account value, and HLP equity represent current external state.
> - Validates: call success, response length, and positive prices in several read helpers.
> - Mutability: external Hyperliquid/HyperCore state.
> - On failure: reader calls revert.

> **HyperCore CoreWriter / CoreDepositWallet** - via `ActionEncoder` and Vault bridge/hedge functions
> - Assumes: action payloads are encoded with correct assets, units, and destination semantics.
> - Validates: mostly caller-side whitelists and amount bounds.
> - Mutability: external venue semantics and asset configuration.
> - On failure: expected revert or external action failure behavior.

> **USDC / USDM tokens** - via Vault, escrows, InsuranceFund, and sUSDM
> - Assumes: ERC20 transfers are standard; SafeERC20 is used for protocol-handled ERC20 movement.
> - Validates: balance checks before several escrow/yield transfers.
> - Mutability: USDC is external; USDM is protocol-controlled.
> - On failure: SafeERC20 paths revert.

---

## 3. Invariants

> ### Full invariant map: **[invariants.md](invariants.md)**
>
> A dedicated reference file contains the complete invariant analysis.
>
> - **20 Enforced Guards** (`G-1` ... `G-20`) - per-call preconditions with check, location, and purpose
> - **9 Single-Contract Invariants** (`I-1` ... `I-9`) - conservation, bounds, state-machine, temporal
> - **3 Cross-Contract Invariants** (`X-1` ... `X-3`) - caller/callee pairs that cross scope boundaries
> - **3 Economic Invariants** (`E-1` ... `E-3`) - higher-order properties deriving from `I-N` and `X-N`
>
> Every inferred block cites concrete code. The **On-chain=No** blocks are high-signal audit targets, not confirmed findings.

---

## 4. Documentation Quality

| Aspect | Status | Notes |
|--------|--------|-------|
| README | Present | README includes overview, scope, known issues, trusted roles, and invariant descriptions |
| NatSpec | Adequate | File scan found ~302 NatSpec/comment markers across source |
| Spec/Whitepaper | Missing locally | README links external docs, but no local whitepaper/spec beyond README |
| Inline Comments | Thorough in core flows | Vault, Accountant, ActionEncoder, and sUSDM include security-relevant design comments |

---

## 5. Test Analysis

| Metric | Value | Source |
|--------|-------|--------|
| Test files | 23 | File scan |
| Test functions | 40 | File scan for test/invariant/check patterns |
| Line coverage | Unavailable - missing `lib` dependencies prevented compilation | `forge coverage` |
| Branch coverage | Unavailable - missing `lib` dependencies prevented compilation | `forge coverage` |

### Test Depth

| Category | Count | Contracts Covered |
|----------|------:|-------------------|
| Unit | 23 files | Broad: Vault, Accountant, Governance, sUSDM, TokenMath, simulator paths |
| Stateless Fuzz | 14 signals | TokenMath and bounded conversion/action tests present |
| Stateful Fuzz (Foundry) | 8 signals | `test/invariants/SolvencyInvariant.t.sol` and invariant harness patterns |
| Fork | 0 detected by scan | none confirmed from local scan |
| Formal Verification | 0 | no Certora/Halmos/HEVM configs detected |

### Gaps

- Coverage could not run until dependencies under `lib/` are installed or submodules are populated.
- No formal verification configuration was detected for the accounting, conversion, or external-state assumptions.

---

## 6. Developer & Git History

> Repo shape: limited public history - source arrived mainly in one setup commit; current analyzed branch: `main` at `3d94be1361ca01d959f9165a78f0d75c5657fe3e`.

| Signal | Observation |
|--------|-------------|
| Contributors | `git shortlog` shows one dominant contributor plus several single-commit actors |
| Source evolution | `52d5998 Code: Initial Setup` introduced the in-scope source set |
| Later commits | Mostly README/scope updates on current branch |
| Hotspots | No meaningful current-branch source churn after initial setup |
| Security-relevant history | Git history is too shallow to infer fix trends from current branch alone |

**Cross-reference synthesis**

- Because source is effectively imported in one commit, risk ranking should lean more on code structure and tests than git churn.
- README known issues explicitly classify OPERATOR and UPGRADER trust as accepted design assumptions, which matches the code-level role map.
- Several PoC-named tests exist locally, suggesting known issue reproduction work, but coverage could not confirm pass/fail status.

---

## X-Ray Verdict

**Tier: Focused Review Required**

Monetrix has strong local documentation, explicit role boundaries, meaningful tests on disk, and clear accounting invariants, but the highest-risk behavior depends on HyperCore external state and instant operator workflows. Coverage could not run because dependencies are missing, so executable confidence is incomplete in this checkout.

Key observations:
- The most important review target is Accountant backing and yield gating across HyperCore-derived values.
- Redemption and sUSDM cooldown accounting have clear symmetry guards worth preserving in tests.
- OPERATOR, GOVERNOR, and UPGRADER risks are mostly trusted-role/configuration risks unless a non-privileged path is proven.
- No confirmed vulnerability is asserted by this X-Ray; suspicious areas are investigation targets.
