---
name: guard-consistency
description: Breaks permission models by finding functions that violate the contract's own guard patterns — attacker-style exploitation of inconsistent guards, initialization hijacks, privilege escalation, confused deputies and delegatecall abuse, backed by the consistency-principle detection algorithm (state interaction matrix, guard dependency graph, anomaly detection with confidence thresholds). Use when auditing access control, missing requires, forgotten modifiers, or inconsistent pause/admin checks.
---

# Guard Consistency Analysis

Two lenses on the same hunt ground, merged. **[Q]** quillshield's
Consistency Principle: a contract is its own specification — find where it
breaks its own rules, with a formal detection algorithm. **[P]** pashov's
attacker plan: map the complete permission surface, then exploit every gap.
Record the lens tag in each finding's `rationale` field.

## When to Use

- Auditing for missing access controls, inconsistent pause checks, logic bugs, forgotten modifiers
- Analyzing contracts with emergency/admin functions that might bypass safety mechanisms
- Detecting logic bugs that are syntactically correct but semantically dangerous
- When you suspect "forgotten check" vulnerabilities
- When traditional tools report no issues but logic errors may exist

## When NOT to Use

- Pure state-to-state invariant analysis (use invariant-conservation)
- Full multi-dimensional audit (use behavioral-state-analysis)
- Code quality or gas optimization reviews

## Part A — The attacker plan [P]

You are an attacker that exploits permission models. Map the complete access
control surface, then exploit every gap.

**Map the permission model.** Every role, modifier, and inline access check.
Who grants what to whom. This map is your weapon — every attack below
references it.

**Exploit inconsistent guards.** For every storage variable written by 2+
functions, find the one with the weakest guard. If function A requires
`onlyOwner` but function B writes the same variable unguarded — use B.
Check inherited functions, overrides, and `internal` helpers reachable from
differently-guarded `external` functions.

**Hijack initialization.** Call `initialize()` on the implementation
contract directly. Front-run deployment to initialize with your own roles.
Pass `address(0)` as a role parameter to permanently lock out admins.

**Escalate privileges.** Find routes where role A grants role B to itself.
Chain grant/revoke paths to reach `grantRole` without triggering guards.
Find upgrade paths that bypass timelock. Trigger `renounceRole` to leave the
system unrecoverable.

**Exploit confused deputies.** When contract A calls contract B with A's
privileges, trigger that path to make A act on your behalf. Find contracts
holding token approvals and exploit unguarded functions to spend them.

**Abuse delegatecall/proxy.** Collide storage layouts. Self-destruct
implementation contracts. Collide admin slots with business logic storage.

## Part B — The Consistency Principle [Q]

> **"A smart contract is its own specification."**

Instead of checking against external rules, analyze what the contract
**claims** to enforce, then find where it **breaks its own rules**.

If a critical state variable (like user balances) is protected by a security
check (like a pause mechanism) in 90% of functions, the 10% without that
check are likely vulnerabilities.

### Phase 1: State Interaction Matrix

For each state variable, track every function that touches it:

```
State Variable: balance
├─ deposit()        → [WRITE] + Guards: [paused, initialized]
├─ withdraw()       → [WRITE] + Guards: [paused, initialized]
├─ transfer()       → [WRITE] + Guards: [paused]
└─ emergencyWithdraw() → [WRITE] + Guards: [] ⚠️
```

For each function-variable interaction, record: write access, guard access
(does the function check this variable in `require()` or `if()`?), read
access.

**Extract guard sources:**
- Modifier chains (`onlyOwner`, `nonReentrant`, `whenNotPaused`)
- Explicit `require` statements
- Conditional branches gating state changes
- External calls affecting state
- Event emissions signaling state changes

### Phase 2: Guard Dependency Graph

```
Guard Relationship: A → B (A guards B)

paused ──────┐
             ├──→ balance
initialized ─┘
owner ───→ paused
owner ───→ totalSupply
```

**Frequency weighting**:

```
Confidence(guard → state) = |functions applying guard| / |functions modifying state|
```

`paused` guards `balance` in 9/10 functions → 90% confidence (strong).
`owner` guards `totalSupply` in 3/10 → 30% (weak).

**Composite dependencies**: track multi-variable guards —
`(owner AND timeLock) → criticalFunction`; `(paused OR emergency) → userAccess`.

### Phase 3: Anomaly detection (the solver)

```
For each state variable S that can be modified:
  1. M = all functions that write to S
  2. G = common guards across those functions (above threshold)
  3. V = M \ G (functions that modify without guards)
  4. V is the vulnerability set
```

**Threshold-based inference:**

| Guard Frequency | Classification | Action |
|-----------------|---------------|--------|
| ≥ 80% | Strong invariant | Flag violations as HIGH/CRITICAL |
| 50-79% | Weak invariant | Flag violations as MEDIUM |
| < 50% | No pattern | Ignore (too inconsistent) |

**Severity classification:**

| Bypass Type | Severity |
|-------------|----------|
| Strong invariant on financial state (`balance`, `totalSupply`) | **Critical** |
| Strong invariant on access control (`owner`, admin roles) | **High** |
| Weak invariant on any state | **Medium** |
| Inconsistent pattern with no security implications | **Low/Info** |

**Impact scale for state variables:**

| Variable Type | Impact Score | Examples |
|---------------|-------------|----------|
| Financial balances | 10 | `balance`, `deposits`, `stakes` |
| Supply controls | 9 | `totalSupply`, `mintable` |
| Access control | 8 | `owner`, `admin`, `roles` |
| Protocol parameters | 7 | `feeRate`, `interestRate` |
| Operational state | 6 | `paused`, `initialized` |
| Configuration | 4 | `maxLimit`, `threshold` |
| Metadata | 2 | `name`, `symbol`, `uri` |

`Severity(v) = Confidence(g → s) × Impact(s)`

### Privilege overlay

Not all "bypasses" are vulnerabilities. Apply role-based filtering:

| Role Level | Scrutiny | Rationale |
|------------|----------|-----------|
| Public functions | Highest | Must follow all established patterns |
| Owner/Admin functions | Medium | May bypass operational guards, must be consistent with each other |
| Emergency functions | Lower | Designed for exceptional cases |
| Internal functions | Context-dependent | Analyze based on callers |

**Filtering rule**: for each function in V, identify its privileges, compare
with other functions at the SAME privilege level, flag only if the bypass is
inconsistent WITHIN the privilege tier.

**Context-aware filtering**: constructors and `initialize()` may legitimately
bypass patterns; `view`/`pure` functions cannot modify state — skip; proxy
`delegatecall` requires special handling; emergency functions may
intentionally bypass some guards.

**Cross-contract extension**: if Contract A calls `B.updateState()`, analyze
whether guards in A should propagate to B, and whether B performs unguarded
operations on behalf of A (proxy patterns, upgradeable consistency,
multi-contract protocols, library safety).

## The merged attack loop

For every storage variable written by 2+ functions:

1. Build the row of the State Interaction Matrix (Phase 1) — this is your
   [P] "map the permission model" at variable granularity.
2. Compute guard confidence and classify (Phase 2-3).
3. For each function in the vulnerability set V, run the [P] attacks:
   can the weakest-guard function be reached by an attacker? Does it
   enable initialization hijack, privilege escalation, confused deputy, or
   storage collision chains?
4. Filter through the privilege overlay, then confirm with a concrete call
   sequence achieving unauthorized access.

## Case study: the "forgotten check"

```solidity
contract Vault {
    mapping(address => uint256) public balance;
    bool public paused;

    function deposit() public payable {
        require(!paused, "Contract paused");       // checks paused
        balance[msg.sender] += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(!paused, "Contract paused");       // checks paused
        balance[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }

    function adminWithdraw(address user) public onlyOwner {
        // Missing paused check!
        uint256 amount = balance[user];
        balance[user] = 0;
        payable(owner).transfer(amount);
    }
}
```

Detection: `M_balance = {deposit, withdraw, adminWithdraw}`,
`G_paused = {deposit, withdraw}`, `V = {adminWithdraw}` — modifies balance
without checking paused. Confidence 66.7% (2/3), severity HIGH (financial
state + admin bypass of safety mechanism). The [P] lens adds: a compromised
owner key plus this missing pause check = unstoppable drain during an
incident.

## Workflow

```
Task Progress:
- [ ] Step 1: Map the permission model — every role, modifier, inline check (P)
- [ ] Step 2: Build the State Interaction Matrix for each state variable (Q)
- [ ] Step 3: Build guard dependency graph with frequency weighting (Q)
- [ ] Step 4: Run anomaly detection (V = M \ G) per variable (Q)
- [ ] Step 5: For each V member, run the attacker plan — hijack, escalate, deputy, delegatecall (P)
- [ ] Step 6: Apply privilege overlay (filter legitimate bypasses) (Q)
- [ ] Step 7: Score, confirm with concrete call sequences, report
```

## Output format

```markdown
### Finding: [Title]

**Function:** `functionName()` at `Contract.sol:L145`
**Severity:** [CRITICAL | HIGH | MEDIUM | LOW]
**Confidence:** [Percentage]
**Lens:** [P] / [Q]

**Issue:** Modifies `[state variable]` without checking `[guard]`
**Guard gap:** the guard that's missing — show the parallel function that has it

**Pattern Evidence:**
- `function1()` checks `[guard]` before modifying `[state]` ✓
- `function2()` checks `[guard]` before modifying `[state]` ✓
- `functionName()` does NOT check `[guard]` before modifying `[state]` ✗

**Guard Frequency:** X out of Y functions (Z%)

**Attack Scenario:**
1. [Concrete call sequence achieving unauthorized access]

**Recommendation:**
Add `require([guard], "[message]")` before modifying `[state]`, or document
why this function intentionally bypasses the check.
```

## Quick detection checklist

- [ ] For every variable written by 2+ functions: is the weakest-guard writer attacker-reachable?
- [ ] Any `initialize()` callable on the implementation contract directly?
- [ ] Any role-grant chain reaching `grantRole` without triggering guards?
- [ ] Any contract holding token approvals with an unguarded spend path (confused deputy)?
- [ ] Emergency/admin functions breaking guard patterns the rest of the contract consistently applies?
- [ ] `internal` helpers reachable from differently-guarded `external` functions?

## Rationalizations to reject

- "The admin is trusted, so skipping the check is fine" → Compromised admin + missing pause check = unstoppable drain
- "This function is only called internally" → Verify all callers; internal doesn't mean safe
- "The pattern only appears in 2 functions" → Even 2/3 consistency is a signal worth investigating
- "It's an emergency function" → Emergency functions should be MORE carefully guarded, not less
- "Traditional tools said it's fine" → Traditional tools check syntax, not semantic consistency

## References

- [references/case-studies.md](references/case-studies.md) — worked guard-anomaly cases
- [references/detection-algorithm.md](references/detection-algorithm.md) — formal definition, confidence and severity formulas, multi-guard and cross-contract extensions
