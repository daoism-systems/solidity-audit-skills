---
name: invariant-conservation
description: Detects broken mathematical relationships between contract state variables by inferring invariants (sum, conservation, ratio, monotonic, synchronization) and finding functions that violate them, then depth-tracing each violation across every reader and writer with exact enforcement-gap line numbers. Catches unauthorized minting, broken tokenomics, accounting desynchronization, conservation law violations and constraint enforcement gaps. Use when auditing for state invariants, broken accounting, supply mismatches, or desynchronized state.
---

# Invariant & Conservation Analysis

Two lenses on the same hunt ground, merged. First **infer** the mathematical
relationships between state variables and find functions that **break** them
[Q]. Then **depth-trace** each candidate across all functions with exact
enforcement-gap line numbers [L].

> **Lens tags.** [Q] sections derive from quillshield's invariant taxonomy;
> [L] from plamen's depth state-trace procedure. Record the tag in each
> finding's `rationale` field for cross-methodology attribution.

## When to Use

- Auditing token contracts for supply/balance mismatches
- Analyzing staking, vault, or pool contracts for accounting errors
- Detecting conservation law violations in treasury/fund management
- Finding AMM/DEX constant product violations
- Verifying that aggregate variables stay synchronized with individual records
- Following up breadth-pass flags on state variables or constraint gaps

## When NOT to Use

- Guard-state consistency analysis (use guard-consistency)
- Full multi-dimensional audit (use behavioral-state-analysis)

## Part A — Invariant inference and violation detection [Q]

### Five types of state relationships

**Type 1: Sum (Aggregation)** — `totalSupply = Σ balance[i]`. Found in ERC20
tokens, staking pools, vaults, share systems.

**Type 2: Difference (Conservation)** — `totalFunds = availableFunds +
lockedFunds`. Found in treasuries, liquidity pools, vesting contracts.

**Type 3: Ratio (Proportional)** — `k = reserveA × reserveB`;
`sharePrice = totalAssets / totalShares`. Found in AMMs, vault share pricing,
collateralization.

**Type 4: Monotonic (Ordering)** — `newValue ≥ oldValue` (only increases).
Found in timestamps, nonces, accumulated rewards, total distributions.

**Type 5: Synchronization (Coupling)** — if stateA changes, stateB must
change correspondingly. Found in deposit/mint pairs, burn/release pairs,
collateral/borrowing power.

### Phase 1: State variable clustering

```
For each pair of state variables (A, B):
  1. Track all functions that modify A
  2. Track all functions that modify B
  3. CoMod(A, B) = |Functions modifying both A and B| / |Functions modifying A or B|
  4. If CoMod(A, B) > 0.6 → A and B are likely related
```

Example: `mint()` modifies BOTH totalSupply and balances; `burn()` both;
`transfer()` only balances → CoMod = 2/3 = 66.7% → cluster
(totalSupply, balances).

### Phase 2: Invariant inference

**Delta pattern matching**: `mint()`: Δtotal = +amount, Δbalance = +amount →
same direction, same magnitude. `transfer()`: Δbalance1 = -x, Δbalance2 =
+x → net zero. Inference: totalSupply = Σ balances.

**Delta correlation**: ΔA = ΔB always → direct proportional (A = B +
constant). ΔA = -ΔB → inverse (A + B = constant). ΔA × constant = ΔB →
ratio. ΔA whenever ΔB → synchronization.

**Expression mining**: parse the actual code — `totalSupply += amount;
balances[user] += amount;` extracts Δtotal = Δbalance, infers total = Σ
balances. `available = total - locked` extracts conservation law.

**Invariant confidence**:

```
Confidence(I) = |functions preserving I| / |functions modifying variables in I|
≥ 90% STRONG · 70-89% MODERATE · < 70% WEAK/NO invariant
```

### Phase 3: Violation detection

```
For each inferred invariant I(stateA, stateB):
  For each function F that modifies stateA or stateB:
    Before: I(stateA, stateB) = True
    After execution: I(stateA', stateB') = False  → F is VULNERABLE
```

## Part B — Depth trace of each candidate [L]

For EACH candidate variable or constraint gap from Part A (or from a breadth
pass), apply the mandatory analysis checks first:

1. **Devil's Advocate**: Answer "What would make this exploitable?" (never "nothing")
2. **Cross-Domain Dependencies**: identify 2-3 assumptions the target makes
   OUTSIDE its domain (oracle freshness, token transfer side effects,
   external call return values). "If this assumption broke, would my target
   become exploitable?" Tag `[CROSS-DOMAIN-DEP: {domain}]` — chain analysis
   uses these to discover compound exploits invisible to single-domain agents.
3. **Chain Check**: search prior findings for ones that CREATE the missing precondition
4. **Evidence Quality**: tag all evidence [PROD-ONCHAIN], [CODE], [MOCK] — [MOCK]/[EXT-UNV] cannot support REFUTED
5. **Confidence Gate**: uncertain? → CONTESTED, not REFUTED. Only REFUTED if defense proven with production evidence
6. **Enabler Search**: before REFUTED, ask "Does ANY other finding enable this?"

### B1. Complete state graph

For the target state variable:
- List EVERY function that READS it
- List EVERY function that WRITES it
- Draw the dependency graph: which functions depend on its value?
- List functions that CHANGE what it SHOULD represent without directly
  writing it (e.g., a function that increases the protocol's balance but
  doesn't update the balance-tracking variable)

### B2. Cross-function consistency

- If X increments in function A, does it decrement in function B?
- Are all increment/decrement operations atomic (no partial updates)?
- Can function A put the variable in a state that function B doesn't handle?

### B3. Constraint enforcement trace

For each constraint variable (min/max/cap/limit) and EACH function that
should enforce it:
- Is the check present? (require/if/assert)
- Is it on ALL code paths? (early returns, branches)
- Is the comparison operator correct? (< vs <=, > vs >=)

Document enforcement gaps with EXACT line numbers.

### B4. Entry point → downstream trace

For each entry point: what state does it modify, what downstream functions
read those variables, and if it forgets to update variable X, what breaks
downstream? Trace the COMPLETE data flow from user input to final state.

### B5. Unenforced variable deep dive

For any variable marked unenforced: confirm there is really NO enforcement;
if enforcement exists, document where; if truly unenforced, what's the
impact — can admin or user abuse it?

### B6. Write-read consistency audit

- How is it READ — what do consumers assume? (stable per period?
  monotonically increasing? reflects total supply?)
- How is it WRITTEN — what does the update logic produce?
- Does the write satisfy what readers assume?
- Should it be constant within a time window (epoch, cycle, day) but gets
  modified mid-window?

### B7. Always-on boundary checklist

For every numeric counter, balance-like field, index, or length in scope,
evaluate `{0, 1, max, boundary-1, boundary, boundary+1, empty-container}`
and state whether downstream readers still behave correctly.

### B8. Cache lifecycle set-cover (node-client / bounded-cache paths)

**Trigger**: target names a cache, pool, index, set, map, or pending/seen
structure (`txCache`, `seen_blocks`, `peerPool`, `pendingBlobs`,
`headerCache`, `msgIDSeen`, `ancestorCache`). Skip if no bounded
memory-backed structure is in the target set.

A bounded cache needs a **complete set** of lifecycle operations — missing
any one leg creates an unbounded-growth DoS (CVE-2023-40591 geth unbounded
p2p cache, reth #20110 post-bad-block OOM) OR a stale-serve bug (geth
#22529/#23195 ancestor cache skew, Erigon #5294/#8193 tx pool retention,
Nethermind #3393 receipts cache staleness). Single-leg presence is NOT
safety.

| Leg | What to look for | Missing-leg consequence |
|-----|------------------|------------------------|
| INSERT bounded by size cap | `if len(cache) >= maxSize { evict }` BEFORE insert | unbounded growth → OOM |
| INSERT bounded by time TTL | `entry.addedAt = now()` with TTL sweep or lazy expiry | long-lived stale entries |
| EVICT on natural lifecycle | `delete(cache, k)` in the handler that retires the object | stale serve after lifecycle end |
| EVICT on error / bad-block | delete in the error path too, not just happy path | reth #20110 class |
| EVICT on reorg / rollback | reorg handler drops entries keyed on orphaned blocks | stale references to dropped chain |
| READ policy consistent (LRU touch vs FIFO) | promote/touch call matches declared policy | unbounded growth under read-heavy workload |
| KEY uniqueness under adversary control | adversary cannot grind N distinct keys mapping to one object | cache amplification DoS |
| SIZE accounting matches reality | counter incremented on insert, decremented on EVERY evict leg | size drift → cap never reached |

For each MISSING leg, state the exact handler that should have the
deletion/bound. For each WRONG_PATH leg, state where the code currently
lives and why that path is not always taken.

**Verdict gate**: CONFIRMED vulnerable if ≥1 leg is MISSING on a path the
adversary can drive. Two+ MISSING legs → HIGH by default on
consensus-reachable paths.

**Near-miss**: a compact numeric index/handle with OTHER structures caching
data keyed by the same index space is asymmetric invalidation across coupled
structures, not single-cache eviction.

## Dual-layer integration [Q]

Combine with **guard-consistency** (the Layer-1 guard lens) for maximum
coverage:

| Layer 1 Violation | Layer 2 Violation | Combined Severity |
|-------------------|-------------------|-------------------|
| Missing Guard | Breaks Invariant | **CRITICAL** |
| Missing Guard | No Invariant Break | **HIGH** |
| No Guard Issue | Breaks Invariant | **HIGH** |
| No Guard Issue | No Invariant Break | **LOW/INFO** |

## Workflow

```
Task Progress:
- [ ] Step 1: Identify all state variables and their modifying functions
- [ ] Step 2: Build co-modification matrix; cluster related variables (CoMod > 0.6)
- [ ] Step 3: Infer invariant type for each cluster (delta patterns)
- [ ] Step 4: Test each function against inferred invariants
- [ ] Step 5: For each violation, run Part B depth trace (B1-B8 as applicable)
- [ ] Step 6: Apply temporal filtering (only flag persistent violations)
- [ ] Step 7: Score severity and generate report
```

## Output format

```markdown
### Finding: [Title]

**ID**: [IC-N]
**Function:** `functionName()` at `Contract.sol:L42`
**Severity:** [CRITICAL | HIGH | MEDIUM]
**Lens:** [Q] inference / [L] depth trace
**Invariant:** `[mathematical expression]`

**Before Execution:**
  stateA = [value], stateB = [value]
  Invariant: [expression] = True

**After Execution:**
  stateA = [value'], stateB = [value']
  Invariant: [expression] = False

**State Graph:**
[variable]
  ├─ READ BY: functionA (line X), functionB (line Y)
  └─ WRITTEN BY: functionC (line Z), functionD (line W)

**Enforcement Points:**
| Function | Line | Check Present? | Correct Operator? | All Paths? |

**Root Cause:** [Which state variable was modified without updating its counterpart]
**Impact:** [Accounting errors, inflated supply, broken pricing, exploitable drift]
**Attack Scenario:**
1. [Step-by-step exploit leveraging the desynchronization]
**Recommendation:** [Specific fix — add the missing state update]
```

Verdicts per candidate: CONFIRMED / REFINED / REFUTED / CONTESTED — with
REFUTED only on proven defense, CONTESTED escalated rather than dropped.

## Quick detection checklist

- [ ] Does every function that modifies `balances` also update `totalSupply` (or have a valid reason not to)?
- [ ] Does every function that moves between `available` and `locked` maintain `total = available + locked`?
- [ ] Does every swap/trade function maintain the constant product `k = reserveA * reserveB`?
- [ ] Do aggregate counters (`totalStaked`, `totalRewards`) stay synchronized with per-user mappings?
- [ ] Are monotonic variables (nonces, timestamps) ever decremented?
- [ ] Are constraint checks (min/max/cap/limit) present on ALL code paths with the right operator?

## Rationalizations to reject

- "The totalSupply is just for display" → Protocols use totalSupply for share pricing, voting power, market cap — drift is exploitable
- "Admin functions can bypass invariants" → Admin functions that break accounting create permanent protocol insolvency
- "The difference is small" → Small accounting errors compound over time and transactions
- "It's an emergency function" → Emergency functions that break state invariants create worse emergencies
- "Transfer doesn't need to update totalSupply" → Correct, but verify the NET change in sum(balances) is zero

## References

- [references/invariant-types.md](references/invariant-types.md) — full definitions and real-world examples per type
- [references/case-studies.md](references/case-studies.md) — worked detection cases
