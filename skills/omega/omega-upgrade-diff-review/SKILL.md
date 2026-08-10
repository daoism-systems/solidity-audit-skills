---
name: omega-upgrade-diff-review
description: Audit an upgrade, a follow-up engagement, or a PR diff rather than a fresh codebase. Covers scoping to a commit range, diffing storage layout across versions, initializer and migration hazards, checking that a v2 still honours promises v1 made to existing holders and in-flight messages, verifying that fixes did not introduce new bugs, and re-checking findings left open by earlier reports. Use when the client has been audited before, when scope is a PR or commit range, or when reviewing a fix commit.
---

# Upgrade & Diff Review

Team Omega's book is mostly repeat business — Backed Finance across nine
engagements, Altitude five, Gnosis Bridge four, Giza five, PrimeDAO four. Most
of their reports audit a *change*, not a codebase. That is a different job and
it has its own failure modes.

> Two questions govern the whole review: **what did this diff change**, and
> **what did the change silently break for state and counterparties that
> already exist?**

---

## 1. Scope the diff precisely

Omega's scope sections for these engagements name both endpoints:

- `202501-Altitude-parallel-farming` — a PR URL, plus "the diff between commit
  `5b3026b…` and `4ce09aa…`".
- `202309-Gnosis-Bridge` — a fork: commit `9eb8f1d…` of `Luigy-Lemon/tokenbridge-contracts`,
  which "is a fork of `gnosischain/tokenbridge-contracts` at commit
  `57dd76e…`. The scope of the audit is limited to the changes between these
  two commits."
- `202307-Backed` — "any changes made since our last report from January … i.e.
  the difference between `78c355a…` and `25333f3…`".

Write both hashes. For a fork, write the upstream hash too — the fork point *is*
the baseline, and the delta from upstream is the attack surface.

## 2. Diff the storage layout

The mechanical check that most needs doing and most gets skipped.

`Backed-Token-Bridge-Update BCR1` — in `BackedCCIPReceiver.sol` a variable was
removed from the second slot and a mapping inserted *between two existing
mappings*:

```solidity
- uint256 private _defaultGasLimitOnDestinationChain;   // removed from slot 2
+ mapping(uint64 => ChainInfo) public chainInfos;       // inserted mid-layout
```

> These contracts are upgradeable. If this new version is deployed as an
> upgrade to a deployment of the previous version, it will use the original
> storage layout — this means that the upgraded contract will read data from
> the wrong storage slot, with unpredictable consequences.

The **Resolution** is instructive: the team said the new code would be a fresh
deployment, not an upgrade, so the issue does not apply. Omega still filed it at
medium. File the finding; let the deployment plan be the mitigation, and record
that the mitigation is a *plan*, not code.

Also: `Backed-token-bridge C1` — "Add a `__gap` variable," so the next upgrade
has room.

**Test:** produce the storage layout for both versions (`forge inspect
<Contract> storageLayout`, or `hardhat-storage-layout`) and diff them. Any
change to slot, offset or type of an existing variable is a finding unless the
deployment is fresh.

## 3. Initializers

Upgradeable contracts concentrate risk in initialization, and Omega finds
something here in almost every upgrade engagement.

- `Backed-Multiplier-Updates BAFT1` — "Unprotected v4 Migration Initializer can
  be front-run and permanently lock legitimate migration." The migration
  initializer is a one-shot; whoever calls it first wins, and if that is an
  attacker the real migration can never run.
- `Backed-rebasing-tokens B2` — "`initialize_v3` can be called also after
  initialization."
- `Backed-RebasingTokens BI3` — "Contract initialize function might be re-called
  by anyone."
- `Backed-wrapped-tokens G6` / `BI10` — "Call `_disableInitializers` in the
  constructor of proxy contracts" / "Disable initializers on the implementation
  contract."
- `lsdai L7` — "Remove default values in proxy implementation" (values set at
  declaration land in the implementation's storage, not the proxy's).

**Test:** every `initialize*` function. Is it `initializer`/`reinitializer(n)`
guarded? Is `n` monotonically increasing across versions? Is it access-controlled
in addition to being one-shot? Does the constructor call `_disableInitializers`?
Are there state variables with inline initializers that will never take effect
behind a proxy?

## 4. Does v2 still honour v1's promises?

The subtlest class. Existing holders, in-flight messages and outstanding
approvals were made promises by v1. An upgrade can break them without touching
any line that looks security-relevant.

- `Gnosis XDFB3` — "`executeSignaturesGSN` sends different tokens before and
  after the upgrade." Messages signed under v1 semantics get executed under v2
  semantics.
- `Backed-Token-ERC4626 WBT1` — "The contract does not follow ERC4626
  standard." A wrapper that gains an ERC4626 face must actually satisfy it, or
  integrators built against the standard will break.
- `Altitude C11` — "Transfer and transferFrom functions do not respect ERC20
  standard."
- `Blindex U3` — a modified `swap` that keeps 90% as a penalty "does not respect
  its 'promise' to return at least `amount0Out`". Omega's severity reasoning
  turns on the integration surface: "an end-user will typically not call the
  `swap()` function directly, but rather as part of a chain of swap calls
  triggered by the UniswapV2Router, and these functions expect the `amountOut`
  values to be respected, and will fail if they do not."

**Test:** list what the old version guaranteed — standard conformance, event
shapes, return-value semantics, message formats, the meaning of each stored
value — and check each still holds. Pay attention to state that is *in flight*
across the upgrade: pending requests, unclaimed rewards, signed-but-unexecuted
messages, outstanding approvals.

## 5. Re-check the previous reports' open findings

Omega treats this as an in-scope deliverable, not a courtesy.

`Giza-Pendle G1` **[medium]** — "Unaddressed issues from older reports." It
lists two findings still open from `202505-Giza-ARMA-II`, and then this:

> Also issue A2 from [the May report] (Top-up logic is confusing and can be
> abused) which **was resolved, re-appears in the current code base.**

A regression of a fixed finding. Only a review that carries the old reports
forward catches it. `202505-Altitude` has a dedicated "Fixes from older
reports" section tracking `HV1b`, `HM3` and `HM5` across three prior reports,
with the original report and finding ID cited for each.

**Test:** open every prior report for this client. For each `[not resolved]` and
`[acknowledged]` finding, check the current code. For each `[resolved]` finding,
check it is *still* fixed.

## 6. Audit the fix commit as new code

Fixes introduce bugs. Omega's own archive proves it: `dxDAO VM1` was resolved,
and the resolution note reads —

> This issue was resolved. However, there are issues with the new `redeem`
> function which we added at the end of the report, like `VM13` …

`VM13` **[high]** — "Stakers who lost can redeem their stake" — exists only
because of the fix for `VM1`. Similarly `Altitude VC1` was fixed by routing
`setBufferConfig` through the registry, which required checking that the
registry's own call passes `msg.sender` correctly.

And `202605-Backed-Token-ERC4626` states plainly: "We also audited the other
changes that were made in this commit." Clients bundle unrelated work into fix
commits.

**Test:** diff the fix commit in full, not just the touched lines you asked
about. Re-run every lens over the changed code.

## 7. Watch for divergence from upstream

Several Omega clients fork well-known code. The delta is where the bugs are.

- `Blindex U4` — "UniswapV2Pair.sol code is heavily and unnecessarily
  modified," which Omega flags as the *root cause* of `U1` (broken
  `DOMAIN_SEPARATOR`), `U2` and `U3`. The resolution to all three was deleting
  the fork.
- `dxGovernance V2` — the fix is literally "also decrease the counter in case of
  `BoostedTimeOut`, as it is done in the original code," with a line-anchored
  link to daostack's `GenesisProtocolLogic.sol#L597`. The upstream was right;
  the fork dropped a branch.
- `dxGovernance` on `Avatar.sol` — Omega identifies it as a near-verbatim copy
  of `@daostack/arc` v0.0.1-rc.57, enumerates the two differences, notes the
  file was covered by ChainSecurity's 2019 audit, and concludes the differences
  "are almost trivial and can safely be ignored." **Scoping out is a finding
  too**, when you show your work.

**Test:** for each forked file, diff against the exact upstream version and
review only the delta — but review all of it. And check whether upstream has
since fixed something the fork carries (`Blindex G2` is upstream advisories
against a pinned OpenZeppelin).

---

## Checklist

Scope
- [ ] Both commit hashes recorded; for forks, the upstream fork point too
- [ ] Prior reports for this client enumerated

Mechanics
- [ ] Storage layouts generated for both versions and diffed
- [ ] `__gap` present and correctly sized for future upgrades
- [ ] Every `initialize*` one-shot, access-controlled, correctly versioned
- [ ] `_disableInitializers()` in the implementation constructor
- [ ] No inline state-variable initializers relied on behind a proxy
- [ ] Migration/initialization entry points cannot be front-run into a lock

Semantics
- [ ] Standards the old version satisfied still satisfied
- [ ] Event shapes, return-value semantics and message formats unchanged, or the
      change is deliberate and documented
- [ ] In-flight state — pending requests, unexecuted signed messages, unclaimed
      rewards, outstanding approvals — still valid under v2

Continuity
- [ ] Every open finding from every prior report re-checked
- [ ] Every previously-resolved finding checked for regression
- [ ] Fix commit audited in full, including unrelated changes
- [ ] Forked files diffed against exact upstream; upstream fixes checked for

**Pairs with:** **[Q]** `proxy-upgrade-safety` for the mechanics of storage
collision, selector clashing and delegatecall context across Transparent/UUPS/
Beacon/Diamond patterns — this skill covers the *engagement* around an upgrade
rather than the proxy pattern itself.
