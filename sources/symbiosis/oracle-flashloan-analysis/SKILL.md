---
name: oracle-flashloan-analysis
description: Detects price oracle manipulation and flash loan attack vectors in smart contracts by combining the quillshield trust-model taxonomy with plamen's systematic per-oracle audit procedure. Classifies oracle trust models (Chainlink, TWAP, spot, custom, pull-based), checks staleness, decimals, deviation references and failure modes per oracle, then models atomic flash loan attacks against every manipulable state. Use when auditing any DeFi protocol that reads external price data, integrates oracles, or is exploitable via flash loans.
---

# Oracle & Flash Loan Analysis

Detect vulnerabilities where **external price data can be manipulated** or
**flash loans can exploit protocol logic** within a single transaction. These
two attack vectors are usually combined and are the most common DeFi attack
pattern.

> **Lens tags.** Sections tagged **[Q]** derive from quillshield's cataloged
> taxonomy, **[L]** from plamen's systematic audit procedure. When emitting
> findings, record the tag in the `rationale` field so the cross-verification
> tier can attribute the vote to the originating methodology.

## When to Use

- Auditing any DeFi protocol that reads external price data (lending, DEX, derivatives, yield aggregators)
- Reviewing Chainlink, Uniswap TWAP, Band, Pyth/Redstone, or custom oracle integrations
- Analyzing protocols that interact with or are accessible via flash loans
- Threat modeling for MEV, sandwich attacks, and price manipulation
- When a protocol uses `balanceOf()`, pool reserves, or spot prices for critical calculations

## When NOT to Use

- Contracts with no price dependencies or external data feeds
- Pure access control analysis (use guard-consistency)
- State-to-state invariant checking (use invariant-conservation)

## Part A — Oracle analysis [L+Q]

**Step priority**: Failure Modes (A6) and Deviation Reference Audit (A5c) are
where HIGH/CRITICAL severity findings most commonly hide. If constrained, skip
conditional sections (A4a-A4d, A5a) before skipping A5c or A6.

### A1. Oracle inventory

Enumerate ALL oracle data sources the protocol reads:

| Oracle | Type | Source Contract | Functions Called | Consumers (protocol functions) | Update Frequency | Heartbeat |
|--------|------|-----------------|-----------------|-------------------------------|-----------------|-----------|
| {name} | Chainlink / TWAP / Spot / Custom / Band / Pyth | {address} | {latestRoundData / observe / etc.} | {list all} | {expected} | {documented or UNKNOWN} |

**For each oracle**: What decision does the protocol make based on this data?
(pricing, liquidation threshold, reward rate, rebase trigger, etc.)

**Search patterns and risk levels [Q]:**

| Pattern | Oracle Type | Risk Level |
|---------|------------|------------|
| `latestRoundData()` | Chainlink | Medium (depends on validation) |
| `latestAnswer()` | Chainlink (deprecated) | HIGH (no round validation) |
| `observe()` / `consult()` | Uniswap TWAP | Medium (depends on window) |
| `getReserves()` | AMM spot price | **CRITICAL** (flash-loan manipulable) |
| `balanceOf(address(this))` | Self-balance | **CRITICAL** (donation attack) |
| `slot0()` / `sqrtPriceX96` | Uniswap V3 spot | **CRITICAL** (single-block manipulable) |
| Custom `getPrice()` | Unknown | Requires investigation |

**Hardcoded stablecoin pricing check [L]**: Does the protocol skip oracle
lookup for any asset and hardcode its price to a constant (e.g., `1e8` for
USDC, `1e18` for DAI)? If yes → FINDING. All assets require dynamic oracle
pricing — stablecoins depeg, and hardcoded pricing fails silently when they
do. Check: `return 1e8`, `return 1e18`, `price = PRECISION`, or an oracle
mapping that excludes specific tokens.

**Build an Oracle Dependency Map [Q]:**

```
Contract: LendingPool
├── borrowLimit() → uses getCollateralPrice()
│   └── getCollateralPrice() → calls chainlinkOracle.latestRoundData()
├── liquidate() → uses getDebtPrice()
│   └── getDebtPrice() → calls uniswapPool.slot0() ← SPOT PRICE!
└── calculateInterest() → uses getUtilizationRate()
    └── getUtilizationRate() → reads internal state (safe)
```

### A2. Staleness analysis [L]

For each oracle from A1:

**2a. Staleness checks present?**

| Oracle | `updatedAt` Checked? | Max Staleness Enforced? | Staleness Threshold | Appropriate? |
|--------|---------------------|------------------------|--------------------:|-------------|
| {name} | YES/NO | YES/NO | {seconds or NONE} | {analysis} |

**If NO staleness check**: What happens when the oracle returns stale data?
- [ ] Protocol uses stale price for liquidations → unfair liquidations
- [ ] Protocol uses stale price for minting → mispriced assets
- [ ] Protocol uses stale price for swaps → arbitrage opportunity
- [ ] Protocol uses stale rate for rewards → incorrect distribution

**2b. Stale data impact trace**: For each consumer function, trace the impact
of receiving data that is {heartbeat × 2} old.

| Consumer Function | Data Used | If Stale By {X}: Impact | Severity |
|-------------------|-----------|------------------------|----------|
| {function} | {price/rate} | {specific impact} | {H/M/L} |

**2c. Chainlink-specific checks [L+Q]**

| Check | Code Reference | Status | Missing = [Q] |
|-------|---------------|--------|---------------|
| `latestRoundData()` return values ALL checked? | {location} | YES/NO | |
| `answeredInRound >= roundId` verified? | {location} | YES/NO | HIGH — stale price from previous round |
| `price > 0` validated? | {location} | YES/NO | HIGH — zero/negative price → infinite borrowing or free liquidations |
| `updatedAt != 0` validated? | {location} | YES/NO | MEDIUM — incomplete round data used |
| Sequencer uptime feed checked? (L2 only) | {location} | YES/NO/N/A | HIGH — stale price during L2 outage → unfair liquidations |
| Heartbeat/freshness bound | {location} | YES/NO | HIGH — hours-old price during volatile markets |
| Price deviation bounds | {location} | YES/NO | MEDIUM — extreme outlier not filtered |

Complete validated integration for reference [Q]:

```solidity
(uint80 roundId, int256 price, , uint256 updatedAt, uint80 answeredInRound) =
    priceFeed.latestRoundData();
require(price > 0, "Invalid price");                    // non-negative
require(updatedAt > 0, "Round not complete");           // round complete
require(answeredInRound >= roundId, "Stale price");     // not stale
require(block.timestamp - updatedAt < HEARTBEAT, "Price too old"); // fresh
// L2: sequencer uptime + grace period
```

**2d. Pull-based oracle checks (Pyth, Redstone, etc.) [L]**

If users supply price data in the transaction, ENUMERATE all update/read
sites and PROCESS each:

| Check | Code Reference | Status |
|-------|---------------|--------|
| Timestamp monotonicity: new update's timestamp >= previously stored timestamp? | {location} | YES/NO |
| Pyth confidence interval: `price.conf` checked relative to `price.price`? | {location} | YES/NO |
| Pyth price sign: `price.price` > 0? (Pyth returns `int64`) | {location} | YES/NO |
| Pyth exponent handling: `price.expo` (negative, e.g., -8) applied correctly? | {location} | YES/NO |

**Timestamp monotonicity attack** (Redstone, Pyth, any pull model): If the
protocol stores a price at timestamp T and accepts a later update at
timestamp T-Δ (within the allowed staleness window), an attacker can roll
back the price. Defense: `require(newTimestamp >= lastStoredTimestamp)`.

**Pyth confidence interval attack**: Pyth returns price ± confidence
bracket. Using the raw price without accounting for confidence may allow
borrowing/liquidation at a price up to `conf` away from the true price.
Defense: for collateral pricing use `price - conf`, for debt pricing use
`price + conf` — always favoring protocol safety.

### A3. Decimal normalization audit [L]

For each oracle data flow:

| Oracle | Oracle Decimals | Consumer Expects | Normalization Applied? | Correct? |
|--------|----------------|-----------------|----------------------|----------|
| {name} | {decimals()} | {expected by math} | YES/NO | {analysis} |

- Does the protocol call `decimals()` dynamically or hardcode it? If
  hardcoded → what if the oracle upgrades and changes decimals?
- **MANDATORY GREP**: search all oracle consumer files for `1e18`, `1e8`,
  `1e6`, `10**18`, `10**8`, `10**6`, `1e10`, `10**10`, `decimals()`,
  `normaliz`. For each hit: (1) Is this a decimal normalization constant?
  (2) Does it match the ACTUAL oracle's `decimals()` return value?
  (3) If the oracle is swapped or upgraded, does this constant break?
  Skipping this sweep is a step-execution violation.

**Common decimal mismatches:**
- Chainlink USD feeds: 8 decimals, but protocol assumes 18
- Chainlink ETH feeds: 18 decimals
- Token decimals: varies (6 for USDC, 18 for DAI)
- Cross-multiplication without normalization: `price * amount` where price and amount have different decimal bases

**Decimal chain trace**: for each arithmetic operation using oracle data,
trace `oracle_output_decimals` → `normalization_step` →
`consumer_expected_decimals`. If any step uses a hardcoded constant rather
than reading `decimals()` dynamically → FINDING.

```
result_decimals = oracle_decimals + token_decimals - normalization_decimals
Expected: result_decimals == output_decimals
```

### A4. TWAP-specific analysis [L] (IF TWAP used)

**4a. Window analysis**

| TWAP Oracle | Window Length | Pool Liquidity | Manipulation Cost (est.) | Sufficient? |
|-------------|-------------|----------------|-------------------------|-------------|
| {oracle} | {seconds} | {USD value} | {estimated} | YES/NO |

Risk by window [Q]: < 5 min CRITICAL · 5-15 min HIGH · 15-30 min MEDIUM ·
30-60 min LOW · > 60 min VERY LOW. Also check: is the window configurable?
Can governance reduce it?

**4b. TWAP arithmetic**

| Check | Status | Impact if Wrong |
|-------|--------|-----------------|
| Overflow protection on `tickCumulatives` difference? | YES/NO | {impact} |
| Geometric vs arithmetic mean — correct for use case? | {which used} | {impact if wrong} |
| Time-weighted vs block-weighted — which is used? | {which} | {manipulation vector} |
| Empty observation slots handled? | YES/NO | {impact} |

**4c. Lagging behavior**: During rapid price movements, TWAP lags spot.
Trace: TWAP significantly lower than spot (discounted minting/borrowing)?
Significantly higher (premium liquidations)? Is the lag exploitable by
attackers who can predict direction?

**4d. Cold-start analysis**: Check oracle behavior when history is
insufficient: (1) zero snapshots, (2) single snapshot, (3) window period not
yet elapsed. For each exploitable state: can an attacker act during the
cold-start window at manipulated price? Tag: `[BOUNDARY:snapshots=0]`,
`[BOUNDARY:snapshots=1]`. If TWAP returns 0 or reverts during cold-start with
no fallback → FINDING (minimum Medium).

### A5. Oracle weight / threshold boundaries [L]

**5a. Multi-oracle systems** (IF multi-oracle)

| Oracle System | Aggregation Method | Oracle Count | Agreement Required | What if Disagreement? |
|---------------|-------------------|-------------|-------------------|----------------------|
| {system} | Median / Mean / Weighted / First-valid | {N} | {M of N} | {fallback behavior} |

What happens at exact threshold boundaries? Median of [100, 100, 101] = 100
— correct? Weighted average rounding — impact? One oracle reverts — does
fallback handle it gracefully?

**5b. Oracle-based thresholds**

| Threshold | Oracle Data Used | Threshold Value | At Exact Boundary | Off-by-One? |
|-----------|-----------------|----------------|-------------------|-------------|
| {name} | {oracle field} | {value} | {behavior at exact value} | YES/NO |

Check `>` vs `>=`: at the exact threshold value, does the protocol behave as
intended?

**5c. Deviation reference point audit** — do not skip

For each deviation check (maxDeviation, priceDeviation, deviationThreshold, …):

| Parameter | Measured Against | Reference Source | Reference Manipulable? | Reference Staleable? |
|-----------|-----------------|-----------------|----------------------|---------------------|

1. What is the deviation MEASURED AGAINST? (previous on-chain price, TWAP, external oracle, hardcoded value)
2. Is the reference point itself manipulable? (deviation checks current vs last-recorded, and last-recorded is admin-settable → admin can set a stale reference that makes all future prices "within deviation")
3. Can the reference become stale? (reference updated only on specific actions, and those actions stop occurring)
4. Is the first recorded price special? (no prior reference → deviation check may be bypassed on first update)
5. **Chained feed deviation stacking**: derived price from multiple feeds
   (wBTC→BTC→ETH→UNI requires wBTC/BTC + BTC/ETH + UNI/ETH feeds) —
   individual thresholds compound. Sum the maximum deviations across the
   chain. If total compounded deviation exceeds the protocol's liquidation
   margin or LTV buffer → FINDING. Example: 0.5% + 2% + 2% = 4.5% total
   deviation; if liquidation threshold is only 5% above LTV, the oracle can
   be 4.5% stale before triggering, leaving <1% real buffer.

Tag: `[TRACE:deviation check: current vs {reference} → reference source: {X} → manipulable: {Y/N}]`

### A6. Oracle failure modes [L] — do not skip

For each oracle, model failure scenarios:

| Failure Mode | Oracle Behavior | Protocol Response | Impact | Mitigation Present? |
|-------------|-----------------|-------------------|--------|-------------------|
| Zero return | Returns 0 | {what happens} | {impact} | YES/NO |
| Revert | Call reverts | {what happens} | {impact} | YES/NO - try/catch? |
| Stale (heartbeat exceeded) | Returns old data | {what happens} | {impact} | YES/NO - staleness check? |
| Extreme value | Returns outlier | {what happens} | {impact} | YES/NO - bounds check? |
| Negative price (Chainlink int256) | Returns < 0 | {what happens} | {impact} | YES/NO - sign check? |
| Sequencer down (L2) | Stale + backlog | {what happens} | {impact} | YES/NO - uptime feed? |

For each unmitigated failure mode: worst-case impact? Can it lead to fund
loss?

**Circuit breaker check**: Does the protocol have a mechanism to pause
oracle-dependent operations if the oracle enters a failure state?

## Part B — Flash loan analysis [L+Q]

### B0. External flash susceptibility check [L]

Before analyzing the protocol's OWN flash loan paths, check whether external
protocols the contract interacts with are susceptible to third-party flash
manipulation.

**0a. External interaction inventory**

| External Protocol | Interaction Type | State Read by Our Protocol | Can 3rd Party Flash-Manipulate That State? |
|-------------------|-----------------|---------------------------|-------------------------------------------|
| {DEX/pool/vault} | {swap/deposit/query} | {reserves, price, balance} | {YES if spot state / NO if TWAP or time-weighted} |

**0b. Third-party flash attack modeling**: For each external state marked
YES: (1) Before — protocol reads external state X; (2) Flash manipulate —
attacker flash-borrows and trades on the external protocol to move state X;
(3) Victim call — attacker calls OUR protocol function that reads
manipulated state X; (4) Restore; (5) Impact — what did the attacker gain
from our protocol acting on manipulated state?

**Key question**: Does our protocol use **spot state** (manipulable) or
**time-weighted state** (resistant)?

**0c. DEX price manipulation cost estimation** (IF DEX interaction)

| Pool | Liquidity (USD) | Target Price Change | Est. Trade Size | Slippage Cost | Protocol Extractable Value | Profitable? |
|------|----------------|--------------------:|----------------|--------------|---------------------------|-------------|
| {pool} | {TVL} | {%} | {USD} | {USD} | {USD} | {YES/NO} |

Cost formula: `manipulation_cost = slippage * trade_size` where
`trade_size = (target_price_change / price_impact_per_unit) * pool_liquidity`.
If `manipulation_cost < extractable_value` → VIABLE. Uniswap V2-style:
`price_impact = trade_size / (reserve + trade_size)`. For V3 concentrated
liquidity: use actual liquidity in the affected tick range, not total TVL.

### B1. Flash-loan-accessible state inventory [L]

Enumerate ALL protocol state manipulable within a single transaction via
flash-borrowed capital:

| State Variable / Query | Location | Read By | Write Path | Flash-Accessible? | Manipulation Cost |
|------------------------|----------|---------|------------|-------------------|-------------------|
| `balanceOf(address(this))` | {contract} | {functions} | Direct transfer | YES | 0 (donation) |
| `totalSupply` | {contract} | {functions} | mint/burn | YES if permissionless | Deposit amount |
| `getReserves()` | {pool} | {functions} | Swap | YES | Slippage cost |
| Oracle spot price | {oracle} | {functions} | Trade on source | YES | Market depth |
| Threshold/quorum state | {contract} | {functions} | Deposit/stake | YES | Threshold amount |

For each YES entry: trace all functions that READ this state and make
decisions based on it.

### B2. Atomic attack sequence modeling [L]

For each flash-loan-accessible state:

```
1. BORROW: Flash-borrow {amount} of {token} from {source}
2. MANIPULATE: {action} to change {state_variable} from {value_before} to {value_after}
3. CALL: Invoke {target_function} which reads manipulated state
4. EXTRACT: {what_is_gained} - quantify: {amount}
5. RESTORE: {action} to return state (if needed for repayment)
6. REPAY: Return {amount + fee} to flash loan source
7. PROFIT: {extract - fee - gas} = {net_profit}
```

**Profitability gate**: If net_profit ≤ 0 for all realistic amounts →
document as NON-PROFITABLE but check B3 for multi-call chains.

For each sequence, verify:
- [ ] Can steps 2-5 execute atomically (same transaction)?
- [ ] Does any step revert under normal conditions?
- [ ] Is the manipulation detectable/preventable by the protocol?
- [ ] What is the minimum flash loan amount needed?

**Detection algorithm [Q]**:

```
For each function F that reads price/value data:
  1. Identify the price source S
  2. Can S be manipulated within a single transaction?
     - Spot price from AMM → YES (swap in same tx)
     - balanceOf(address(this)) → YES (donate tokens)
     - Chainlink feed → NO (off-chain updates)
     - TWAP → DEPENDS (short window = risky)
  3. What does F do with the price?
     - Determines borrowing limit → CRITICAL
     - Triggers liquidation → CRITICAL
     - Sets exchange rate → HIGH
     - Informational only → LOW
  4. Is the manipulation profitable?
     - value_extracted - (flash_loan_fee + slippage + gas) > 0 → EXPLOIT VIABLE
```

### B3. Cross-function flash loan chains [L]

Model multi-call atomic sequences within a single flash loan:

| Step | Function Called | State Before | State After | Enables Next Step? |
|------|---------------|-------------|------------|-------------------|
| 1 | {function_A} | {state} | {state'} | YES - changes {X} |
| 2 | {function_B} | {state'} | {state''} | YES - enables {Y} |
| N | {function_N} | {state^N} | {final} | EXTRACT profit |

**Key question**: Can calling function A then function B in the same
transaction produce a state that neither function alone could create?

**Common multi-call patterns**:
- Deposit → manipulate rate → withdraw (sandwich own deposit)
- Stake → trigger reward calculation → unstake (flash-stake rewards)
- Borrow → manipulate collateral price → liquidate others → repay
- Deposit to inflate shares → withdraw deflated shares

**Common attack patterns [Q]**: oracle manipulation (flash swap → inflate
collateral → over-borrow), governance attack (flash-borrow governance tokens
→ vote → execute → return), liquidation manipulation (crash price →
liquidate at discount), share price inflation (donate to vault → front-run
deposit), arbitrage amplification.

**3b. Flash-loan-enabled debounce DoS**: For each permissionless function
with a cooldown/debounce that affects OTHER users (global cooldown, shared
timestamp): can attacker flash-borrow → call debounced function → trigger
cooldown, blocking legitimate callers?

| Function | Cooldown Scope | Shared Across Users? | Flash-Triggerable? | DoS Duration |
|----------|---------------|---------------------|-------------------|-------------|

If cooldown is global/shared AND function is permissionless AND
flash-triggerable → FINDING (minimum Medium).

**3c. No-op resource consumption**: For each state-modifying function with a
limited-use resource (cooldown, one-time flag, nonce, epoch-bound action):
can it be called with parameters producing zero economic effect (amount=0,
same-token swap, self-transfer) while consuming the resource? If a no-op
call consumes a resource blocking legitimate use → FINDING.

**3d. External flash × debounce cross-reference (MANDATORY)**

For EACH external protocol flagged flash-susceptible in B0:

| External Protocol | Flash-Accessible Action | Debounce/Cooldown Affected (from 3b) | Combined Severity |
|-------------------|------------------------|--------------------------------------|-------------------|

Can the external flash loan trigger ANY debounce/cooldown found in 3b? If
YES: (1) Is the debounce consumption permanent (no admin reset) or temporary?
(2) If permanent: is there ANY on-chain path to reset? (3) Combined finding
inherits the HIGHER severity of the two. (4) Tag:
`[TRACE:flash({external}) → call({debounce_fn}) → cooldown consumed → {duration/permanent}]`.
If no debounce functions exist from 3b: mark N/A and skip.

### B4. Flash loan + donation compound attacks [L] (IF balance-dependent)

| Donation Target | Flash Loan Action | Combined Effect | Profitable? |
|-----------------|-------------------|-----------------|-------------|
| {contract}.balanceOf | Deposit/withdraw | Rate manipulation | {YES/NO} |
| {pool}.reserves | Swap | Price oracle manipulation | {YES/NO} |
| {governance}.balance | Vote/propose | Quorum manipulation | {YES/NO} |

Check: can a flash-borrowed amount be donated (not deposited) to manipulate
`balanceOf(this)` accounting, then extracted via a subsequent protocol call
within the same transaction?

### B5. Circular dependency detection [Q]

Find cases where a protocol's pricing depends on its own state, creating
exploitable feedback loops.

```
Protocol A uses Token X price → from Pool P
Pool P contains Token X + Token Y
Protocol A issues Token X (or affects its supply)

→ CIRCULAR: Protocol A's actions change Token X supply
            → changes Pool P reserves
            → changes Token X price
            → changes Protocol A's valuations
```

Detection:

```
For each price oracle call in the contract:
  1. What token/asset is being priced?
  2. Does THIS contract mint, burn, or distribute that token?
  3. Does THIS contract add/remove liquidity from the pricing pool?
  4. Does any action in THIS contract affect the reserves of the pricing pool?

  If YES to any → CIRCULAR DEPENDENCY
  Severity: CRITICAL if the circular path can be exploited atomically
```

### B6. Flash loan defense audit [L] — do not skip

For each flash-loan-accessible attack path:

| Defense | Present? | Effective? | Bypass? |
|---------|----------|------------|---------|
| Reentrancy guard (`nonReentrant`) | YES/NO | {analysis} | {if YES: how} |
| Same-block prevention (`block.number` check) | YES/NO | {analysis} | Multi-block possible? |
| TWAP instead of spot price | YES/NO | TWAP window length: {N} | Short TWAP vulnerable? |
| Minimum lock period / cooldown | YES/NO | Duration: {N blocks/seconds} | Bypass via partial? |
| Balance snapshot (before/after comparison) | YES/NO | {analysis} | {if YES: how} |
| Flash loan fee exceeds profit | YES/NO | Fee: {X}, max profit: {Y} | Fee < profit? |

**TWAP-specific**: If TWAP window < 30 minutes AND pool liquidity < $10M →
flag as potentially manipulable.

**6b. Defense parity audit (cross-contract)**: For each user-facing action
that exists in multiple contracts (stake, withdraw, claim, exit):

| Action | Contract A | Flash Defense | Contract B | Flash Defense | Parity? |
|--------|-----------|---------------|-----------|---------------|---------|
| {action} | {contract} | {defense list} | {contract} | {defense list} | {GAP if different} |

If ContractA.stake() has a cooldown that prevents flash-stake-claim-withdraw
but ContractB.stake() has NO cooldown for the same economic action — can an
attacker use ContractB as the undefended path to extract the same value? For
each GAP: (1) Can the undefended contract achieve the same economic outcome?
(2) Does the defended contract's protection become meaningless? (3) Is the
difference intentional (documented) or accidental?

## Output format

```markdown
### Finding: [Title]

**ID**: [OR-N] / [FL-N]
**Function:** `functionName()` at `Contract.sol:L42`
**Category:** [Oracle Manipulation | Stale Price | Flash Loan | Circular Dependency]
**Severity:** [CRITICAL | HIGH | MEDIUM]
**Lens:** [L] / [Q]

**Oracle Source:** `[oracle contract/function]`
**Trust Level:** [1-5 from hierarchy below]

**Description / Vulnerability:**
[Data-flow trace; for flash loans the full atomic attack sequence with amounts]

**Attack Scenario:**
1. Attacker obtains flash loan of [X tokens] from [source]
2. ...
N. Net profit: [amount]

**Missing Validations:**
- [ ] Price > 0 check
- [ ] Staleness check (heartbeat)
- [ ] Round completeness check
- [ ] L2 sequencer check
- [ ] Price deviation bounds

**Impact:** [Quantified under worst-case oracle scenario / realistic flash amounts]
**Recommendation:** [Specific fix — add TWAP, add Chainlink validation, circuit breaker, pull pattern]
```

## Oracle trust hierarchy [Q]

Not all price sources are equally secure. Oracle vulnerabilities stem from
the gap between **assumed trust** and **actual manipulation resistance**.

```
Trust Level (highest to lowest):
5: Multi-oracle consensus + circuit breakers + TWAP + staleness checks
4: Chainlink with full validation (staleness, sequencer, min answers)
3: Uniswap V3 TWAP (long window) — multi-block manipulation cost
2: Uniswap V2 TWAP (short window) or Chainlink WITHOUT staleness check
1: Spot price from single pool, or balanceOf() for pricing  ← Manipulable via flash loan
```

## Quick detection checklist

- [ ] Does any function use `getReserves()`, `slot0()`, or `balanceOf()` for pricing? (Flash-loan manipulable)
- [ ] Does Chainlink integration check for `price > 0`, staleness, and round completeness?
- [ ] Is the TWAP window long enough to resist multi-block manipulation (> 30 min)?
- [ ] Does the protocol's own token appear in its pricing oracle's pool? (Circular dependency)
- [ ] Can any critical operation (borrow, liquidate, swap) be called in the same transaction as a flash loan?
- [ ] Are there price deviation circuit breakers for extreme moves?
- [ ] On L2: Is the sequencer uptime checked before using price data?
- [ ] Hardcoded stablecoin pricing present?
- [ ] Deviation checks: reference point manipulable, staleable, or bypassable on first update?

## Rationalizations to reject

- "We use Chainlink, so it's safe" → Only if ALL validation checks are implemented; partial integration is common
- "Flash loans can't affect our protocol" → Any protocol using manipulable price sources is affected
- "The TWAP window is 10 minutes" → Multi-block manipulation is feasible for well-funded attackers
- "Our oracle is a trusted admin feed" → Admin key compromise → arbitrary price → instant drain
- "The pool is too large to manipulate" → Flash loans provide unlimited capital for single-transaction manipulation
- "We check if price is non-zero" → Non-zero is necessary but not sufficient; stale/manipulated non-zero prices are dangerous

## References

For oracle trust models per type and the flash loan attack catalog
(including real-world cases), see:
- [references/oracle-types.md](references/oracle-types.md)
- [references/flash-loan-vectors.md](references/flash-loan-vectors.md)
