# Sherlock Question Inventory

> Monetrix | Sherlock `003f0f2310d6aa81fed6b0bff3800bf860084715` | checkout `1feb7ddd89e8352879b325d2456989a9d3887f2d` | `QUESTION_ONLY`

This portfolio was regenerated with the current GitHub Sherlock question-card engine. It contains investigation questions, not vulnerability claims, proof results, or severity judgments.

## Run Header

- Mode: **Discovery + Novel Attack** (`C+R`).
- X-Ray inputs: `x-ray/x-ray.md`, `x-ray/entry-points.md`, `x-ray/invariants.md`, `x-ray/architecture.json`, and `x-ray/architecture.svg`.
- Additional inputs: README, scope files, all in-scope source, current test names/comments as read-only context, and the current worktree state.
- Missing inputs: `x-ray/enumeration.json`, a local protocol specification beyond README, deployment manifests, and production HyperCore state.
- Scope: the 20 Solidity files in `scope.txt`; tests remain out of contest scope and were not executed.
- Trust boundary: pure OPERATOR compromise/inaction and trusted GOVERNOR/UPGRADER misuse are marked `trusted/config question` unless an untrusted loss path is identified later.
- Search accounting: **196 raw candidates** were tracked across the ten rooms below; **80 protocol-specific questions** survived compression; **16** were compiled into bounded high-risk cards.

## Protocol Story

USDC enters the Vault and creates equal USDM liabilities. USDM can wait in wallets, enter a redemption queue, or become sUSDM shares. Operators move principal and strategy value between HyperEVM and multiple HyperCore venues, then use Accountant's composite backing view to declare yield. That yield waits in YieldEscrow before being split among sUSDM, InsuranceFund, and Foundation. The decisive safety boundary is therefore semantic: each dollar, debt, share, escrow obligation, L1 identity, external tuple field, and time window must retain the same meaning across contracts and lifecycle phases.

## Observation Ledger

| ID | Observed fact | Why it exists | Hidden assumption | Opposite assumption | Question route |
|---|---|---|---|---|---|
| O-01 | `suppliedBalance` decodes four 0x811 fields and returns only the fourth | expose supplied PM balance | returned supply is already net owned value | ignored fields include liabilities that must offset supply | SQ-001 |
| O-02 | `_readL1Backing` adds 0x80F account value, spot, supplied, hedge tokens, and HLP | build composite backing | components are economically disjoint | one external value already includes another | SQ-002 |
| O-03 | one backing call performs many external precompile reads | value all venues | all reads share one state snapshot | a transfer is seen at source and destination inconsistently | SQ-003 |
| O-04 | Vault stores redemption requests; selected RedeemEscrow stores their aggregate obligation | separate queue from custody | escrow wiring stays fixed while requests live | a request later points at a different obligation ledger | SQ-004 |
| O-05 | `distributeYield` consumes the entire YieldEscrow token balance | distribute settled funds | every escrow dollar came from `settle` | direct transfers or stranded funds have another meaning | SQ-005 |
| O-06 | sUSDM rate is raw USDM balance divided by shares | ERC4626 appreciation | donations and rounding cannot shift value unfairly | sequencing transfers value between cohorts | SQ-006 |
| O-07 | APR gate uses current USDM supply after elapsed time accrued | cap declared yield | current supply represents time-weighted earning capital | boundary mint/burn changes the cap | SQ-007 |
| O-08 | `pmEnabled` is a local boolean representing an external account mode | select 0x811 behavior | local and external transitions are synchronized | either side changes first | SQ-008 |
| O-09 | supplied registry stores a `perpIndex`; Config can remove/re-add asset mappings | value supplied non-USDC | pair meaning remains immutable | stored oracle pairing becomes stale | SQ-009 |
| O-10 | HLP equity is decoded as unsigned `uint64` and added at face value | include HLP mark value | negative/exhausted states are encoded conservatively | loss-domain semantics differ | SQ-010 |
| O-11 | Vault, USDM, sUSDM, and operator actions have separate pause planes | isolate emergencies | every partial lifecycle remains recoverable | one plane permits preparation while another blocks completion | SQ-011 |
| O-12 | raw escrow balances are used as accounting/control inputs | simple custody | unsolicited tokens are harmless | donations become obligations or yield | SQ-012 |

## Business-Flow Decomposition

| Flow | Observed facts | Critical states | Value at risk | Hidden assumption | Opposite assumption | Questions | Coverage gap |
|---|---|---|---|---|---|---|---|
| Operation | setup → deposit/stake → strategy/settle → redeem/unstake → recover | wired/unwired, active/paused, funded/shortfall | all user principal and yield | normal order is preserved | delayed/reordered steps change meaning | SQ-004, SQ-011, Q-023 | migration runbook absent |
| Asset lifecycle | USDC → USDM → sUSDM or redemption claim → USDC/USDM exit | EVM, L1, escrow, live vault | conservation and entitlement | every representation has one owner | one dollar is duplicated, omitted, or captured by another cohort | SQ-005, SQ-006, SQ-012 | no independent accounting oracle |
| State machine | PM inactive/active; request pending/claimable/claimed; HLP locked/unlocked | external-mode transitions and request finality | bridge and claim liveness | local flags represent full external state | mismatch states exist | SQ-008, SQ-010, SQ-011 | external transition semantics absent |
| Accounting | EVM balance + L1 components − USDM liabilities − redeem shortfall | surplus positive/negative, funded/unfunded | solvency and yield | components are complete, disjoint, current, and same-unit | liability omitted or value overlaps | SQ-001, SQ-002, SQ-003, SQ-007 | authoritative precompile schemas absent |
| Privilege | Guardian pauses; Operator moves; Governor configures/recovers; Upgrader replaces | live positions during role actions | recoverability and policy | honest actions at bad times remain safe | config timing strands or reallocates value | SQ-004, SQ-014 | deployment role manifest absent |
| External dependency | CoreWriter actions and precompile reads bridge EVM/L1 semantics | partial fill, delayed credit, stale read, outage | strategy value and settlement correctness | accepted action equals completed economic state | success/event precedes completion | SQ-003, SQ-008, SQ-010 | no production state/finality evidence |
| Invariant | supply/backing, owed/escrow, shares/assets, principal/identity | every mutation boundary | peg, claims, fair yield | paired values always move together | cross-contract update breaks the pair | all cards | executable proof deferred |

## Semantic Ledger

| Value | Human/accounting meaning | Must move with | Can be stale/rounded/external? | Failure if meaning changes |
|---|---|---|---|---|
| `outstandingL1Principal` | global principal sent away from EVM | returnable value under the spending L1 identity | external and identity-dependent | redemption bound overstates accessible liquidity |
| 0x811 returned supply | PM supplied collateral credited as backing | all debt/borrow/interest fields defining net ownership | external; tuple partially discarded | phantom distributable surplus |
| 0x80F account value | signed margin-account equity | positions and any collateral included by HyperCore | external and mode-dependent | overlap or omission in backing |
| `vaultSupplied` entries | slots Accountant elects to read | external slot lifecycle and current oracle pairing | stale by config/registry timing | wrong valuation or settlement DoS |
| `RedeemEscrow.totalOwed` | aggregate live USDM redemption obligation | the exact Vault request set assigned to that escrow | wiring-dependent | orphaned or misdirected claims |
| `distributableSurplus` | yield-safe excess after redemption liquidity pressure | backing, supply, current shortfall | external, time-sensitive | unearned USDM yield minted |
| YieldEscrow balance | pending funds treated as distributable yield | settlement attribution and intended split cohort | externally donatable | principal/donation reclassified as yield |
| `sUSDM.totalAssets()` | live USDM owned by current shares | total supply and claim isolation | donatable and rounded | cohort value transfer |
| `totalPendingClaims` | exact assets physically isolated for unstakers | sUSDMEscrow balance and live request sum | donation can alter raw balance | commitment invariant becomes ambiguous |
| `pmEnabled` | local belief about external PM activation | 0x811 availability and bridge semantics | config can lag external state | omitted liquidity or reverting reads |

## Count Summary

| Category | Raw | Final | Cards |
|---|---:|---:|---:|
| Value flow / accounting | 36 | 8 | 4 |
| Ownership / entitlement | 14 | 8 | 1 |
| Lifecycle / timing | 22 | 8 | 2 |
| External dependency behavior | 31 | 8 | 3 |
| Batch / registry / indexing | 16 | 8 | 1 |
| Role / configuration | 15 | 8 | 1 |
| Recovery / escrow | 19 | 8 | 1 |
| Automation / liveness | 14 | 8 | 1 |
| Documentation intent mismatch | 11 | 8 | 1 |
| Unusual cross-feature questions | 18 | 8 | 1 |
| **Total** | **196** | **80** | **16** |

## Priority Queue

| ID | Rank | Route | Target |
|---|---:|---|---|
| SQ-001 | 1 | Reasoning | 0x811 supply credited without proving discarded fields contain no paired liability |
| SQ-002 | 2 | Reasoning | overlap between 0x80F signed account value and later-added components |
| SQ-003 | 3 | Reasoning | non-coherent external snapshot across composite backing reads |
| SQ-004 | 4 | Reasoning | live redemption requests across `setRedeemEscrow` |
| SQ-005 | 5 | Reasoning | unsolicited or accumulated YieldEscrow balance treated as one yield batch |
| SQ-006 | 6 | Reasoning | two-user donation/cooldown/rounding entitlement sequence |
| SQ-007 | 7 | Reasoning | current-supply APR denominator after boundary mint/burn |
| SQ-008 | 8 | Reasoning | `pmEnabled` transition mismatch with real HyperCore mode |
| SQ-009 | 9 | Reasoning | stored supplied oracle pairing after Config remove/re-add |
| SQ-010 | 10 | Reasoning | unsigned HLP equity semantics under loss states |
| SQ-011 | 11 | Reasoning | existing claims across the product of pause states |
| SQ-012 | 12 | Reasoning | direct donations to custody contracts becoming accounting inputs |
| SQ-013 | 13 | Reasoning | unbounded registries or request arrays freezing core operations |
| SQ-014 | 14 | trusted/config | config mutation between settle and distribute |
| SQ-015 | 15 | Reasoning | new depositor becomes exit liquidity during existing insolvency |
| SQ-016 | 16 | Reasoning | multiple settlements plus cohort change exceed injection/entitlement assumptions |

## High-Risk Question Cards

### SQ-001 — 0x811 Supplied Value Versus Paired Liabilities

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** X-1, E-1, composite-backing surface; `PrecompileReader.sol:84-91`; `MonetrixAccountant.sol:146-157`.
- **Observed Facts:** four `uint64` fields are decoded; only the fourth is returned and added positively to backing.
- **Business Flow:** PM supply/borrow → composite backing → surplus gate → yield settlement.
- **Target:** semantic completeness of the credited 0x811 amount.
- **Boundary:** Vault/multisig PM accounts and registered assets; not generic oracle failure.
- **Context:** local success/length checks prove ABI shape, not net economic ownership.
- **Witness / Signal:** discarded tuple positions have no accounting treatment.
- **Hidden Assumption:** the returned supply field is net backing.
- **Opposite Assumption:** ignored fields contain borrow principal, accrued debt, or another paired liability.
- **Entry Sequence:** activate PM slot → create supplied balance plus liability state → `totalBackingSigned()` → `settle()`.
- **Asset / State:** PM collateral and borrow liabilities.
- **Invariant At Risk:** E-1 solvency and I-5 yield bound.
- **Permutation Axis:** external tuple semantics × debt state × settlement timing.
- **Question:** Can gross supplied collateral be credited while an economically paired PM liability remains unaccounted, making distributable surplus exceed net backing?
- **Why Others May Miss It:** the reader is typed and fail-closed, so semantic omission hides behind correct decoding.
- **Novelty Signal:** partial tuple consumption controls protocol solvency.
- **Drift Guard:** This is about ignored 0x811 economics, not registry staleness or precompile reverts. Do not assume field meanings. Later verification must inspect the canonical schema and a real/mocked borrow state. Stop when every tuple field is mapped into net equity.
- **Reasoning-Agent Handoff:** obtain authoritative 0x811 definitions; compare Accountant output with independent net PM equity under the smallest supply-plus-borrow state.
- **Status:** `QUESTION_ONLY`

### SQ-002 — Composite Component Overlap

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** X-1/E-1; `MonetrixAccountant.sol:137-171`; `PrecompileReader.sol:75-82`.
- **Observed Facts:** signed 0x80F account value is added before spot USDC, supplied balances, spot hedge tokens, and HLP equity.
- **Business Flow:** L1 venue allocation → backing aggregation → settlement.
- **Target:** economic disjointness of aggregated components.
- **Boundary:** supported margin modes; not stale-price speculation.
- **Context:** locally correct values can still represent overlapping sets.
- **Witness / Signal:** no component-inclusion matrix exists in code/docs.
- **Hidden Assumption:** 0x80F is perp-only value.
- **Opposite Assumption:** it already internalizes collateral or liabilities added elsewhere.
- **Entry Sequence:** allocate one account across venues → read all components → settle.
- **Asset / State:** composite HyperCore account equity.
- **Invariant At Risk:** each economic dollar counted once; E-1.
- **Permutation Axis:** margin mode × venue allocation × aggregation.
- **Question:** Are 0x80F and every later-added component mutually exclusive under standard, cross-margin, and PM configurations?
- **Why Others May Miss It:** unit tests often validate each reader independently.
- **Novelty Signal:** set-overlap failure rather than arithmetic failure.
- **Drift Guard:** This is not SQ-001's discarded liability question. Later verification must map inclusion/exclusion semantics. Stop when a sourced component matrix proves disjointness or identifies overlap.
- **Reasoning-Agent Handoff:** seed components independently and jointly; compare backing deltas to authoritative semantics.
- **Status:** `QUESTION_ONLY`

### SQ-003 — Cross-Precompile Snapshot Coherence

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** X-1; Accountant and PrecompileReader aggregation paths.
- **Observed Facts:** backing uses multiple reads over assets and two identities during one EVM call.
- **Business Flow:** external venue transfer → backing snapshot → settlement.
- **Target:** whether all terms observe one coherent HyperCore state.
- **Boundary:** legitimate internal venue transfers and documented finality; not arbitrary malicious data.
- **Context:** value can move spot↔supplied↔perp↔HLP without changing net ownership.
- **Witness / Signal:** no snapshot identifier or block/state-root consistency check.
- **Hidden Assumption:** every read is anchored to one finalized L1 state.
- **Opposite Assumption:** source and destination update at different observable times.
- **Entry Sequence:** initiate venue transfer → call `settle` around completion boundary.
- **Asset / State:** value in transit across HyperCore venues.
- **Invariant At Risk:** no double count/omission and E-1.
- **Permutation Axis:** external timing × read order × venue transition.
- **Question:** Can a single backing evaluation see value at both source and destination, or neither, during a real transition?
- **Why Others May Miss It:** every reader can be individually correct.
- **Novelty Signal:** temporal composition across external reads.
- **Drift Guard:** Do not infer asynchronous behavior without evidence. Later verification must establish read-finality guarantees. Stop if one-state-root atomicity is authoritative.
- **Reasoning-Agent Handoff:** document snapshot semantics and simulate the narrowest supported venue transfer around settlement.
- **Status:** `QUESTION_ONLY`

### SQ-004 — Live Redemption Across Escrow Replacement

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** I-1/I-2/X-2; `MonetrixVault.sol:183-211,502-505`; `RedeemEscrow.sol:40-69`.
- **Observed Facts:** request ownership lives in Vault; obligation lives in current escrow; requests do not store escrow identity.
- **Business Flow:** request under escrow A → rewire to B → fund/claim.
- **Target:** lifecycle of existing requests during migration.
- **Boundary:** honest timelocked migration with live positions; not Governor theft.
- **Context:** `claimRedeem` always calls the currently configured escrow.
- **Witness / Signal:** setter has no zero-obligation or migration check.
- **Hidden Assumption:** escrow changes only when no request exists.
- **Opposite Assumption:** old requests survive and later mutate B's ledger.
- **Entry Sequence:** request on A → `setRedeemEscrow(B)` → claim old ID.
- **Asset / State:** queued USDM, A/B USDC, A/B `totalOwed`.
- **Invariant At Risk:** X-2 request/obligation coupling and no burn without correct payout.
- **Permutation Axis:** role timing × live position × wiring.
- **Question:** Can an old request burn USDM while attempting to decrement or pay from B, leaving A's obligation and funds orphaned?
- **Why Others May Miss It:** both escrows correctly enforce `onlyVault` locally.
- **Novelty Signal:** identity pairing is absent from the request object.
- **Drift Guard:** This is not generic privileged misuse. Later verification must track both ledgers through a realistic migration. Stop when an enforced migration invariant is proven.
- **Reasoning-Agent Handoff:** run A→B with one funded and one unfunded request and record every balance/ledger delta.
- **Status:** `QUESTION_ONLY`

### SQ-005 — YieldEscrow Balance Attribution

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** I-5/E-1; `MonetrixVault.sol:364-414`; `YieldEscrow.sol:35-45`.
- **Observed Facts:** distribution pulls the entire raw escrow balance; tokens can arrive without `settle`; multiple settlements can accumulate.
- **Business Flow:** settle/donate/accumulate → cohort changes → distribute.
- **Target:** whether every distributed dollar has the same earned-yield meaning.
- **Boundary:** permissionless transfers, stranded balance, and accumulated settlements; not Operator theft.
- **Context:** no batch ID links escrow balance to settlement time, split, or beneficiary cohort.
- **Witness / Signal:** raw balance is both custody state and distribution input.
- **Hidden Assumption:** all balance came from homogeneous valid settlements.
- **Opposite Assumption:** donations or older batches have different attribution.
- **Entry Sequence:** direct transfer or two settles → user enters/exits sUSDM → distribute.
- **Asset / State:** USDC escrow balance and minted USDM yield.
- **Invariant At Risk:** backing conservation and fair yield entitlement.
- **Permutation Axis:** origin × batch timing × cohort membership.
- **Question:** Can principal/donation or prior-period yield be reclassified and distributed to a cohort that did not earn it?
- **Why Others May Miss It:** unsolicited transfers look value-positive while balance controls minting.
- **Novelty Signal:** custody balance doubles as semantic batch state.
- **Drift Guard:** require measurable holder harm, unfair capture, or backing mismatch; voluntary subsidy alone is not enough.
- **Reasoning-Agent Handoff:** compare post-state supply/backing/beneficiary deltas for settle-origin, donation-origin, and multi-settlement balances.
- **Status:** `QUESTION_ONLY`

### SQ-006 — Donation and Cooldown Path Dependence

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** I-3/X-3/E-3; `sUSDM.sol:102-127,160-245`.
- **Observed Facts:** direct USDM transfer changes rate; cooldown burns shares and moves rounded assets immediately.
- **Business Flow:** stake → donate/yield → victim action → fragmented cooldown → claims.
- **Target:** profitable or griefing two-user sequence, not donation alone.
- **Boundary:** public calls and realistic amounts.
- **Context:** virtual-share offset can block one classic path without proving every ordering.
- **Witness / Signal:** conversion rounding is repeated per request and rate is externally influenceable.
- **Hidden Assumption:** donor always bears the donation and fragmentation is weakly worse.
- **Opposite Assumption:** timing externalizes rounding/value loss to another cohort.
- **Entry Sequence:** attacker stake → donate → victim deposit/cooldown → attacker fragmented cooldown → both claim.
- **Asset / State:** shares, live assets, escrowed claims, residual dust.
- **Invariant At Risk:** fair pro-rata entitlement and conservation.
- **Permutation Axis:** actor order × donation × rounding × fragmentation.
- **Question:** Is there any bounded sequence where attacker recovery exceeds its stake plus donation while another holder receives less than ideal pro-rata value?
- **Why Others May Miss It:** single-path donation tests do not cover sequence space.
- **Novelty Signal:** cross-user path dependence rather than first-depositor inflation alone.
- **Drift Guard:** account for every final balance; do not call donor self-loss a protocol issue. Stop if differential fuzzing proves bounded non-profitability.
- **Reasoning-Agent Handoff:** compare short action sequences against an ideal rational-share model after all claims settle.
- **Status:** `QUESTION_ONLY`

### SQ-007 — APR Denominator Timing

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** I-4/I-5; `MonetrixAccountant.sol:196-224`; deposit/redeem paths.
- **Observed Facts:** elapsed time accrues between settlements but cap uses supply at the final transaction.
- **Business Flow:** yield accrues → supply changes → settle/distribute.
- **Target:** mismatch between current and time-weighted earning supply.
- **Boundary:** permissionless deposits/claims near settlement; not false Operator reporting.
- **Context:** supply can change without equivalent earning time.
- **Witness / Signal:** no accumulator or snapshot for period supply.
- **Hidden Assumption:** boundary supply approximates period capital.
- **Opposite Assumption:** last-moment mint loosens cap or burn tightens it.
- **Entry Sequence:** wait interval at supply S → deposit or claim → settle → distribute.
- **Asset / State:** USDM supply and allowable yield.
- **Invariant At Risk:** annualized yield limit and fair allocation.
- **Permutation Axis:** supply timing × settlement boundary × cohort.
- **Question:** Can a last-moment supply change authorize yield above the intended time-weighted APR or redirect period yield unfairly?
- **Why Others May Miss It:** formula is arithmetically correct at one instant.
- **Novelty Signal:** denominator semantics span time.
- **Drift Guard:** require a safety bypass or measurable unfair allocation, not cap variability alone.
- **Reasoning-Agent Handoff:** compare current-supply and time-weighted caps under boundary deposits and claims.
- **Status:** `QUESTION_ONLY`

### SQ-008 — PM Flag Transition Product State

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** X-1; `MonetrixVault.sol:78-80,267-273,519-540`.
- **Observed Facts:** local flag controls registry notification and 0x811 bridge-availability reads; external PM activation is separate.
- **Business Flow:** external mode migration → local flag → hedge/supply/bridge/settle.
- **Target:** four mismatch states during activation/deactivation.
- **Boundary:** honest configuration timing; not malicious Governor action.
- **Context:** boolean compresses a multi-step external transition.
- **Witness / Signal:** no on-chain attestation that PM mode equals flag.
- **Hidden Assumption:** transitions are simultaneous.
- **Opposite Assumption:** external-first or local-first windows are reachable.
- **Entry Sequence:** toggle either side first → execute each PM-sensitive path.
- **Asset / State:** supplied USDC, registry state, bridge availability.
- **Invariant At Risk:** complete backing and bridge liveness.
- **Permutation Axis:** configuration order × external state × entry point.
- **Question:** Can a mismatch omit accessible value, force strict-read reverts, or approve a bridge against liquidity the action cannot spend?
- **Why Others May Miss It:** boolean state is reviewed as static configuration.
- **Novelty Signal:** product-state migration analysis.
- **Drift Guard:** mark pure coordination failure as trusted/config unless normal recovery leaves persistent user harm.
- **Reasoning-Agent Handoff:** enumerate and test all local/external mode combinations.
- **Status:** `QUESTION_ONLY`

### SQ-009 — Stored Oracle Pairing Drift

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** backing and registry surfaces; `MonetrixAccountant.sol:244-295`; `MonetrixConfig.sol:158-209`.
- **Observed Facts:** supplied entry stores `spotToken, perpIndex`; idempotence is keyed by spot token; Config can remove and later re-add mappings.
- **Business Flow:** register supply → mutate asset config → value supply.
- **Target:** stale stored `perpIndex` after reconfiguration.
- **Boundary:** supported timelocked migration with live/historical slot.
- **Context:** Config and Accountant registries evolve independently.
- **Witness / Signal:** existing entry is not refreshed when a spot token's mapping changes.
- **Hidden Assumption:** spot-to-perp valuation pairing is immutable.
- **Opposite Assumption:** re-add selects a different perp/oracle.
- **Entry Sequence:** register token/perp A → remove → add token/perp B → read backing.
- **Asset / State:** supplied hedge token valuation.
- **Invariant At Risk:** correct units/price and E-1.
- **Permutation Axis:** registry lifecycle × config lifecycle × oracle identity.
- **Question:** Can Accountant keep valuing supply with stale perp A after current Config assigns perp B?
- **Why Others May Miss It:** each registry remains internally valid.
- **Novelty Signal:** cross-registry semantic drift.
- **Drift Guard:** not generic Governor misuse; verify whether live reconfiguration is supported. Stop if pairing immutability is enforced operationally and in scope.
- **Reasoning-Agent Handoff:** execute remove/re-add and compare stored/current pairing and notional.
- **Status:** `QUESTION_ONLY`

### SQ-010 — HLP Loss-Domain ABI Semantics

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** X-1/E-1; `PrecompileReader.sol:19-22,55-63`; `MonetrixAccountant.sol:169-171`.
- **Observed Facts:** HLP equity is `uint64` and added positively.
- **Business Flow:** HLP deposit → mark gain/loss → backing → settlement.
- **Target:** sign, floor, and scale semantics at zero/negative equity.
- **Boundary:** authoritative 0x802 behavior; positive-value decimal tests are context only.
- **Context:** loss-domain encoding may differ from normal positive states.
- **Witness / Signal:** no signed type or explicit saturation handling.
- **Hidden Assumption:** HLP equity cannot be negative and zero is conservative.
- **Opposite Assumption:** response wraps, saturates unexpectedly, or uses a signed representation.
- **Entry Sequence:** create HLP loss state → read backing → attempt settle.
- **Asset / State:** HLP equity under loss/liquidation.
- **Invariant At Risk:** losses reduce backing; E-1.
- **Permutation Axis:** external loss state × ABI decoding.
- **Question:** How does 0x802 encode exhausted or negative equity, and is unsigned decoding conservative for all reachable states?
- **Why Others May Miss It:** existing probes establish scale only for positive equity.
- **Novelty Signal:** ABI domain rather than unit alone.
- **Drift Guard:** do not infer wraparound; require authoritative docs or simulator behavior.
- **Reasoning-Agent Handoff:** probe zero/loss/boundary states and reconcile exact ABI semantics.
- **Status:** `QUESTION_ONLY`

### SQ-011 — Cross-Contract Pause Product

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** redemption/cooldown and privilege surfaces; Vault/USDM/sUSDM pause modifiers.
- **Observed Facts:** user, token, and operator actions stop under different flags; existing requests span contracts.
- **Business Flow:** create claim → pause subset → fund/recover/claim → unpause.
- **Target:** persistent damage to existing positions, not temporary trusted pause.
- **Boundary:** all documented pause combinations and recovery order.
- **Context:** claim can require a token burn/transfer controlled by another pause plane.
- **Witness / Signal:** no unified state machine enforces compatible pause transitions.
- **Hidden Assumption:** healthy unpause restores every request.
- **Opposite Assumption:** partial progress leaves an irrecoverable intermediate state.
- **Entry Sequence:** live redeem and unstake requests → activate combinations → attempt each recovery order.
- **Asset / State:** locked USDM, escrow USDC/USDM, request ledgers.
- **Invariant At Risk:** claim liveness and claim-once conservation.
- **Permutation Axis:** pause subset × request stage × recovery order.
- **Question:** Is any request permanently unclaimable after every role returns to the documented healthy state?
- **Why Others May Miss It:** modifiers appear correct contract-by-contract.
- **Novelty Signal:** product-state rather than single pause bypass.
- **Drift Guard:** temporary downtime is insufficient; require persistent state damage or unsafe mandatory recovery.
- **Reasoning-Agent Handoff:** build a pause truth table and state-machine test for pre-existing requests.
- **Status:** `QUESTION_ONLY`

### SQ-012 — Unsolicited Escrow Assets

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** I-1/I-3/X-2/X-3; both escrow contracts.
- **Observed Facts:** anyone can transfer tokens directly to escrow addresses; raw balances feed `shortfall`, reclaim, commitment comparisons, or distribution.
- **Business Flow:** donate → accounting view → fund/reclaim/claim/distribute.
- **Target:** whether direct assets remain a subsidy or change control/accounting meaning.
- **Boundary:** correct USDC/USDM token donations; not malicious-token behavior.
- **Context:** custody contracts cannot reject ERC20 transfers.
- **Witness / Signal:** balance and ledger can diverge by construction.
- **Hidden Assumption:** extra balance is harmless excess.
- **Opposite Assumption:** downstream code reclassifies it as obligation coverage or yield.
- **Entry Sequence:** direct transfer to each escrow → invoke every balance-dependent path.
- **Asset / State:** excess USDC/USDM and owed/pending ledgers.
- **Invariant At Risk:** semantic equality between custody and entitlement.
- **Permutation Axis:** asset origin × escrow type × downstream action.
- **Question:** Can unsolicited assets change who receives value, which funds are reclaimable, or whether a safety gate passes?
- **Why Others May Miss It:** donation is normally assumed benign.
- **Novelty Signal:** raw balance is a protocol input.
- **Drift Guard:** require adverse reallocation, lock, or false gate; trapped voluntary donation alone is low value.
- **Reasoning-Agent Handoff:** trace every donated dollar through shortfall, reclaim, claim, and distribution.
- **Status:** `QUESTION_ONLY`

### SQ-013 — Unbounded Collection Liveness

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** entry-point and registry maps; Accountant loops and per-user request removal loops.
- **Observed Facts:** tradeable/supplied lists are iterated during backing; per-user request IDs are linearly searched on claim.
- **Business Flow:** normal growth/fragmentation → critical settle or claim.
- **Target:** practical gas ceiling and who controls growth.
- **Boundary:** reachable production limits; not theoretical infinite arrays.
- **Context:** a safety-critical view is executed inside `settle`.
- **Witness / Signal:** no explicit on-chain collection cap.
- **Hidden Assumption:** governance/user list sizes stay operationally small.
- **Opposite Assumption:** permitted growth makes critical paths unexecutable.
- **Entry Sequence:** grow registries or fragment user requests → settle/claim middle ID.
- **Asset / State:** yield liveness and user claim liveness.
- **Invariant At Risk:** progress and recoverability.
- **Permutation Axis:** list length × index position × gas bound.
- **Question:** At what reachable size do core paths exceed practical gas, and can an untrusted user force that size for anyone but itself?
- **Why Others May Miss It:** every loop is straightforward and bounded by storage length.
- **Novelty Signal:** protocol liveness depends on off-chain cardinality assumptions.
- **Drift Guard:** require realistic configured bounds and third-party/protocol impact.
- **Reasoning-Agent Handoff:** benchmark gas by length and identify controlling actor and maximum reachable cardinality.
- **Status:** `QUESTION_ONLY`

### SQ-014 — Configuration Between Settle and Distribution

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** privilege and yield surfaces; Config setters; Vault settle/distribute.
- **Observed Facts:** settle binds neither split nor recipients; distribute reads current Config.
- **Business Flow:** settle → config mutation → distribute.
- **Target:** policy and entitlement of already-settled funds.
- **Boundary:** honest timelocked configuration; `trusted/config question` unless documented user right is violated.
- **Context:** arithmetic still sums to 100%, hiding temporal policy drift.
- **Witness / Signal:** no batch snapshot.
- **Hidden Assumption:** configuration is stable between phases.
- **Opposite Assumption:** pending yield uses a different split or destination.
- **Entry Sequence:** settle → set BPS/foundation/insurance → distribute.
- **Asset / State:** pending YieldEscrow balance and beneficiary shares.
- **Invariant At Risk:** intended yield entitlement.
- **Permutation Axis:** role timing × pending batch × cohort.
- **Question:** Which policy should govern pending yield, and can mutation violate the protocol's stated user share?
- **Why Others May Miss It:** both calls are individually authorized.
- **Novelty Signal:** two-phase policy binding.
- **Drift Guard:** do not classify policy discretion as public loss without documented entitlement.
- **Reasoning-Agent Handoff:** identify intended binding time and test all Config mutations with pending yield.
- **Status:** `QUESTION_ONLY`

### SQ-015 — New Depositor as Insolvent Exit Liquidity

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** E-1 and README soft-solvency statement; Vault deposit/redeem paths.
- **Observed Facts:** deposits remain open at par during negative surplus and add equal asset/liability without repairing deficit.
- **Business Flow:** strategy loss → new deposit → earlier holder redemption.
- **Target:** allocation of pre-existing deficit between cohorts.
- **Boundary:** reachable negative backing and permissionless users; not generic market loss.
- **Context:** new EVM USDC can become immediately available for redemption funding.
- **Witness / Signal:** entry price does not reflect deficit.
- **Hidden Assumption:** par mint is fair during transient insolvency.
- **Opposite Assumption:** earlier holders use new capital as exit liquidity, leaving deficit to newcomer.
- **Entry Sequence:** create deficit → newcomer deposit → old holder request/fund/claim.
- **Asset / State:** USDC recovery pool and USDM liabilities.
- **Invariant At Risk:** fair loss allocation and E-1.
- **Permutation Axis:** insolvency state × cohort order × redemption.
- **Question:** Can an informed old holder externalize a pre-existing deficit onto a new par depositor?
- **Why Others May Miss It:** aggregate deficit is unchanged by deposit.
- **Novelty Signal:** loss-transfer rather than deficit creation.
- **Drift Guard:** distinguish documented soft-solvency policy from an actionable sequence; quantify cohort recovery.
- **Reasoning-Agent Handoff:** compare pro-rata recoveries before and after newcomer deposit and old-holder exit.
- **Status:** `QUESTION_ONLY`

### SQ-016 — Accumulated Yield and Cohort/Injection Boundaries

- **Mode:** Discovery + Novel Attack.
- **X-Ray Sources:** I-5/E-1; settle/distribute and `sUSDM.injectYield` cap.
- **Observed Facts:** multiple valid settlements may accumulate; distribution uses entire balance; user share is injected once and capped per injection.
- **Business Flow:** settle N times → cohort changes → one distribution.
- **Target:** liveness and entitlement when aggregated user share exceeds cap or spans cohorts.
- **Boundary:** normal operator cadence and public stake/unstake timing.
- **Context:** per-settlement validity does not imply aggregate distributability.
- **Witness / Signal:** no partial-distribution cursor.
- **Hidden Assumption:** distribution occurs before aggregation becomes problematic.
- **Opposite Assumption:** one valid accumulated balance freezes distribution or rewards the wrong cohort.
- **Entry Sequence:** settle repeatedly without distribute → change total supply → distribute.
- **Asset / State:** YieldEscrow balance, current sUSDM supply, injection cap.
- **Invariant At Risk:** yield liveness and fair entitlement.
- **Permutation Axis:** batch count × cohort timing × cap.
- **Question:** Can individually valid settlements compose into an undistributable balance or a cross-period value transfer?
- **Why Others May Miss It:** each settle and one-shot distribution is locally valid.
- **Novelty Signal:** safe batches become unsafe when merged.
- **Drift Guard:** require reachable cadence and no ordinary bounded workaround.
- **Reasoning-Agent Handoff:** find the minimum N/cap/supply sequence that blocks or reallocates distribution.
- **Status:** `QUESTION_ONLY`

## Full Question Inventory

### Value Flow / Accounting

1. **Q-001 / SQ-001:** Does 0x811 supplied collateral require subtracting any ignored borrow/debt fields before it becomes backing?
2. **Q-002 / SQ-002:** Is 0x80F account value disjoint from spot, supplied, hedge-token, and HLP components?
3. **Q-003 / SQ-003:** Can a venue transfer be observed at both source and destination during one backing evaluation?
4. **Q-004:** Can 0x801 spot `total` include held/order-reserved value that is not economically available backing?
5. **Q-005:** Does funded versus unfunded redemption state preserve identical surplus for equivalent assets and liabilities?
6. **Q-006 / SQ-007:** Can boundary supply changes distort the intended annualized cap?
7. **Q-007:** Can cumulative settlements exceed cumulative realized surplus despite every point-in-time check passing?
8. **Q-008:** Are all rounding directions conservative after multiple asset valuations are summed and later unwound?

### Ownership / Entitlement

9. **Q-009 / SQ-004:** Which escrow owns a request created before `setRedeemEscrow`?
10. **Q-010 / SQ-006:** Can donation plus fragmented cooldown transfer value from another user?
11. **Q-011:** Who owns residual sUSDM dust after the last live share is cooled down?
12. **Q-012:** Can a later depositor capture residual assets after total supply reaches zero?
13. **Q-013:** Does payer/receiver separation in ERC4626 deposit/mint enable harmful yield timing?
14. **Q-014:** Can transferring shares immediately before cooldown change rounding allocation without equivalent exposure?
15. **Q-015:** Can per-user request arrays disagree with mapping ownership after external failure and retry?
16. **Q-016 / SQ-016:** Which staker cohort owns yield settled before it entered but distributed after it entered?

### Lifecycle / Timing

17. **Q-017 / SQ-007:** What happens when deposit occurs immediately before settlement?
18. **Q-018 / SQ-007:** What happens when redemption claim burns supply immediately before settlement?
19. **Q-019 / SQ-008:** Are PM activation/deactivation safe in both ordering directions?
20. **Q-020 / SQ-014:** Which Config governs yield already waiting in escrow?
21. **Q-021 / SQ-016:** Can several settlements accumulate beyond one injection cap?
22. **Q-022:** Can users enter after settle and exit after distribute to capture past-period yield?
23. **Q-023:** Can an asset be removed while its hedge/supply/repair lifecycle remains active?
24. **Q-024:** Does delayed settlement create a one-shot cap inconsistent with per-injection limits?

### External Dependency Behavior

25. **Q-025:** Is configured USDC guaranteed to transfer exactly the requested amount for protocol lifetime?
26. **Q-026:** Can any supported USDC behavior report success without the expected recipient balance delta?
27. **Q-027 / SQ-010:** How does 0x802 encode zero, negative, or liquidated HLP equity?
28. **Q-028:** Can a CoreWriter call succeed on EVM while the economic action is silently dropped on L1?
29. **Q-029:** Does `SEND_ASSET` spend supplied USDC under PM exactly as bridge preflight assumes?
30. **Q-030:** Can `depositFor` remove EVM USDC before target L1 credit appears in backing?
31. **Q-031:** Are dynamic 0x80A/0x80C responses safely validated beyond a fixed minimum byte length?
32. **Q-032:** Can oracle price and decimals metadata come from inconsistent market/config versions?

### Batch / Registry / Indexing

33. **Q-033 / SQ-009:** Can supplied valuation retain stale perp mapping after remove/re-add?
34. **Q-034:** Can the same economic slot enter both Vault and multisig registries during custody migration?
35. **Q-035:** Can stale off-chain index remove a different supplied entry after swap-and-pop?
36. **Q-036 / SQ-013:** Can registry growth make `totalBackingSigned` or `settle` exceed practical gas?
37. **Q-037 / SQ-013:** Can request fragmentation make middle-ID claim removal unexecutable?
38. **Q-038:** Are token index, perp index, and spot-pair asset ID separated consistently on every path?
39. **Q-039:** Can perp index zero be confused with the USDC special case outside `_resolvePerp`?
40. **Q-040:** Do non-unique `batchId`/`positionId` event identifiers break any off-chain safety automation?

### Role / Configuration

41. **Q-041 / SQ-004:** Can honest escrow migration with live requests strand obligations? (`trusted/config question`)
42. **Q-042 / SQ-014:** Can pending yield change recipients or split between phases? (`trusted/config question`)
43. **Q-043:** Can replacing Accountant disconnect pending YieldEscrow funds from the gate state that admitted them? (`trusted/config question`)
44. **Q-044:** Can asset removal make an existing unmatched hedge irreparable through normal paths? (`trusted/config question`)
45. **Q-045:** Does `maxTVL == 0` intentionally remove all on-chain liability growth limits? (`trusted/config question`)
46. **Q-046:** Are ACL bootstrap renunciation and actual timelocks enforceable deployment invariants? (`trusted/config question`)
47. **Q-047:** Can one-time Vault/escrow bindings make safe migration impossible after component failure? (`trusted/config question`)
48. **Q-048:** Can extreme but valid yield splits create a stranded approval or zero-recipient branch? (`trusted/config question`)

### Recovery / Escrow

49. **Q-049 / SQ-011:** Are live redemption requests recoverable after every pause combination returns healthy?
50. **Q-050 / SQ-011:** Are live unstake requests recoverable after every pause combination returns healthy?
51. **Q-051 / SQ-012:** Can donations to RedeemEscrow change shortfall/reclaim semantics adversely?
52. **Q-052 / SQ-012:** Can donations to sUSDMEscrow change commitment/recovery semantics adversely?
53. **Q-053:** Can YieldEscrow funds migrate if its bound Vault becomes obsolete?
54. **Q-054:** Can InsuranceFund recovery assets be distinguished from deposit principal and yield afterward?
55. **Q-055:** Is every partial hedge/HLP/BLP state recoverable while operator actions are paused?
56. **Q-056:** Can a strict precompile outage block the very unwind required to restore solvency?

### Automation / Liveness

57. **Q-057:** Does `canKeeperBridge` account for both pause planes before signaling automation?
58. **Q-058:** Can `yieldShortfall` revert or mislead under partial wiring?
59. **Q-059 / SQ-016:** Can accumulated yield become permanently undistributable without a partial cursor?
60. **Q-060:** Are two-leg partial hedge fills observable before the next backing read?
61. **Q-061:** Can HLP withdraw pass local checks but be silently dropped externally?
62. **Q-062:** Can strict stale registry entries freeze settlement until an unavailable role acts?
63. **Q-063:** Is multisig-to-Vault SPOT_SEND completion observable before bridge-back automation proceeds? (`trusted/config question`)
64. **Q-064:** Can redemption growth race bridge-out automation so that useful liquidity repeatedly leaves too early?

### Documentation Intent Mismatch

65. **Q-065 / SQ-001:** Does the README's supplied-backing definition explicitly mean net of PM borrowing?
66. **Q-066:** Does “code-bounded Operator” account for economic value transferred through arbitrary limit-order counterparties?
67. **Q-067:** Is the documented cumulative-yield invariant actually enforced by point-in-time surplus checks?
68. **Q-068 / SQ-015:** What entry policy is intended while the documented soft solvency invariant is negative?
69. **Q-069:** Do stated Governor/admin/upgrader delays match actual deployment role holders?
70. **Q-070:** Is `bridgeRetentionAmount` consistently treated only as working capital rather than solvency reserve?
71. **Q-071:** Does “strict inactive 0x811 read” match real inactive-slot behavior?
72. **Q-072 / SQ-012:** How should escrow-balance-equals-commitment invariants treat unsolicited transfers?

### Unusual Cross-Feature Questions

73. **Q-073 / SQ-015:** Can a new depositor become exit liquidity for an older holder during existing insolvency?
74. **Q-074 / SQ-005:** Can donated YieldEscrow USDC be converted into USDM yield owned by current stakers?
75. **Q-075:** Can cooling all shares before distribution redirect user yield to Foundation, and can any actor benefit from forcing that state?
76. **Q-076:** Can asset removal both omit backing and block hedge repair, compounding accounting and recovery pressure?
77. **Q-077 / SQ-003:** Can PM auto-supply move value from 0x801 to 0x811 between reads and be counted twice?
78. **Q-078:** Can custody migration leave value counted under old and new identities while only one can bridge it?
79. **Q-079:** Can pause ordering allow recovery funding but block cleanup, then let settlement misread the intermediate state?
80. **Q-080:** Can redemption timing lower surplus while the same actor changes sUSDM cohort membership to capture the next distribution?

## Unusual Questions Others May Miss

The strongest non-checklist routes are SQ-001 (partial external tuple semantics), SQ-003/Q-077 (snapshot coherence during venue movement), SQ-004 (request has no escrow identity), SQ-005/SQ-016 (raw balance merges yield batches and cohorts), SQ-009 (Config and Accountant registries drift independently), SQ-015 (new capital changes deficit ownership), Q-075 (empty-vault distribution redirection), and Q-078 (custody identity migration).

## Gap Cycle

| Gap | Missing flow | Why it matters | Existing cards | Supplemental route | Stop condition |
|---|---|---|---|---|---|
| G-01 | Canonical 0x811/0x80F/0x802 semantics | local code cannot prove net-value meaning | SQ-001, SQ-002, SQ-010 | authoritative field/inclusion matrix | every returned and ignored field is economically mapped |
| G-02 | HyperCore snapshot/finality model | backing spans moving venues | SQ-003, SQ-008 | transition-boundary reasoning packet | atomicity or bounded conservative behavior is established |
| G-03 | Live-position migration | mutable wiring meets persistent requests/custody | SQ-004, SQ-009, SQ-014 | escrow/accountant/config migration state machine | every obligation and asset remains paired |
| G-04 | Yield cohort specification | settle and distribute are separated | SQ-005, SQ-007, SQ-016 | time/batch entitlement model | intended cohort rule is explicit and testable |
| G-05 | Operational cardinality bounds | critical loops have no code cap | SQ-013 | gas growth benchmark | realistic maximum sizes remain executable |

## Reasoning-Agent Handoff Packets

1. **External net-backing packet:** SQ-001, SQ-002, SQ-003, SQ-010. Required evidence: canonical precompile schemas, component inclusion matrix, snapshot semantics, and independent net-equity calculations. Do not drift into generic “oracle may fail.”
2. **Escrow lifecycle packet:** SQ-004, SQ-005, SQ-012. Required evidence: per-contract balances, ledgers, request ownership, and batch attribution through rewire/donation/claim/distribute sequences.
3. **sUSDM cohort packet:** SQ-006, SQ-007, SQ-016. Required evidence: ideal pro-rata oracle, time-weighted supply comparison, attacker/victim final balances, and all residual dust.
4. **External-mode/config packet:** SQ-008, SQ-009, SQ-014. Required evidence: every transition ordering and explicit trusted/config classification unless untrusted user harm survives.
5. **Recovery/liveness packet:** SQ-011, SQ-013, SQ-015. Required evidence: pause truth table, gas-by-cardinality benchmarks, and cohort recovery during deficit.

## Stop Boundary

Sherlock stops here. No question has a verdict. Reasoning should begin with SQ-001, then SQ-002, SQ-003, SQ-004, and SQ-005. Existing regression tests are false-positive filters for their exact historical routes; they do not prove or reject adjacent questions automatically.
