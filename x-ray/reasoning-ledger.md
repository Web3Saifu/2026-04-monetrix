# Reasoning Ledger

## SQ-001 - 0x811 Supplied Value Versus Paired Liabilities

**Verdict:** CONFIRMED

**Scope lock:** This verification is only about the economic completeness of the 0x811 amount credited as backing. It is not about stale registries, generic precompile failure, multisig identity, or 0x80F account-value semantics.

**Authoritative external semantics checked:** Hyperliquid's official HyperCore interaction docs state that read precompiles expose latest HyperCore state at EVM block construction time and attach the canonical `L1Read.sol` precompile definitions. The current 0x811 schema is `BorrowLendUserTokenState { borrow: BasisAndValue, supply: BasisAndValue }`, where the ABI-flattened tuple is:

```text
borrow.basis, borrow.value, supply.basis, supply.value
```

Hyperliquid's portfolio-margin docs also state that insufficient balances can be automatically borrowed, borrowed assets accrue interest, and users pay borrow interest while earning on supplied or idle assets. Therefore a supplied amount is not, by itself, proof of net PM equity.

**Source trace:**

- `src/core/PrecompileReader.sol:85` calls 0x811 and checks success plus response length.
- `src/core/PrecompileReader.sol:90` decodes four `uint64` words but returns only the fourth word, `supply.value`.
- `src/core/PrecompileReader.sol:133` converts that returned supplied USDC value to EVM USDC units.
- `src/core/PrecompileReader.sol:152` does the same for non-USDC supplied hedge tokens after oracle valuation.
- `src/core/MonetrixAccountant.sol:117` builds `totalBackingSigned()`.
- `src/core/MonetrixAccountant.sol:146` to `src/core/MonetrixAccountant.sol:156` adds each registered 0x811 supplied slot positively to backing.
- `src/core/MonetrixAccountant.sol:180` computes `surplus()` as reported backing minus USDM supply.
- `src/core/MonetrixAccountant.sol:187` computes distributable surplus from that surplus value.
- `src/core/MonetrixAccountant.sol:200` to `src/core/MonetrixAccountant.sol:224` allows `settleDailyPnL()` when proposed yield is within distributable surplus and the annualized cap.

**Proof test added:** `test/MonetrixAccountant.t.sol:350`

The test creates the smallest paired PM state:

```text
EVM backing       = 1,000,000 USDC
USDM liabilities = 1,000,000 USDM
0x811 borrow     = 100,000 USDC
0x811 supply     = 100,000 USDC
correct PM net   = 0
```

A correct net-equity calculation has zero surplus. Current code reports:

```text
reported backing = 1,100,000 USDC
reported surplus = 100,000 USDC
settlement       = accepts 100 USDC proposed yield
```

The test asserts both the independent net backing and the inflated contract output, then proves `settleDailyPnL()` accepts unearned yield from the phantom surplus window.

**Guards checked:**

- 0x811 success and length checks only validate ABI shape. They do not net borrow against supply.
- Supplied-slot registration only controls whether the read occurs. It does not subtract `borrow.value`.
- `onlyVault` restricts who can call settlement, but the Vault path still relies on the inflated Accountant surplus.
- The interval gate and annualized cap limit timing and rate only; they do not repair the backing calculation.
- EVM USDC availability in the Vault settlement flow proves spendable cash exists, not that the cash is earned surplus.

**Commands:**

```text
forge test --match-test test_SQ001_borrowLiabilityOmittedInflatesBackingAndAllowsSettle -vv
Result: 1 passed, 0 failed

forge test --match-contract MonetrixAccountantTest -vv
Result: 38 passed, 0 failed

forge test --no-match-path test/MonetrixFork.t.sol -vv
Result: 283 passed, 0 failed

forge test -vv
Result: 283 passed, 1 failed
Unrelated caveat: test/MonetrixFork.t.sol reverts in setUp().
```

**Impact:** PM borrow liabilities can be omitted from backing while gross supplied collateral is credited. This can make `distributableSurplus()` exceed true net backing, allowing yield settlement from value that is actually offset by borrow debt or interest. The affected invariant is USDM solvency / yield-bound correctness: only net backing should support distributable yield.

**Report-ready summary:** `PrecompileReader.suppliedBalance()` partially consumes the 0x811 borrow-lend state and returns only `supply.value`. `MonetrixAccountant` then adds that value to backing without subtracting the paired `borrow.value`. In a PM state with equal borrow and supply, true PM net equity is zero, but the protocol reports positive surplus and permits `settleDailyPnL()`. This can overstate backing and authorize unearned yield distribution.

**Residual uncertainty:** None for the accounting mechanism under the canonical 0x811 schema. Contest novelty may be separate from technical validity if the same root cause is already known or submitted elsewhere.
