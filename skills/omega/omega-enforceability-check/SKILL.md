---
name: omega-enforceability-check
description: Find security checks that exist but constrain nothing. For every guard, limit, penalty, cooldown, threshold or flag, identify who it is meant to bind and prove it actually binds them — catching limits avoidable with a second address, cooldowns controlled by the party they restrict, validation whose result is discarded, reversed conditions, setters that write the wrong key, flags with no code path behind them, and calldata-sniffing allowlists that miss equivalent calls. Use when reviewing access control, rate limits, penalties, slippage protection, permission registries, pause logic, or any require/modifier that looks correct.
---

# Enforceability Check

A distinct Team Omega discipline, and one the bug-class checklists miss
entirely. Static analysers and "missing modifier" detectors look for guards
that are **absent**. Omega repeatedly finds guards that are **present,
correct-looking, and completely ineffective**.

They even have a house word for it. `Blindex P2`: "Redeem penalty of 90% is
*unenforceable*." `dxGovernance W1`: "Limits on ERC20 token transfers are not
*enforceable*." `Toucan T2`: "Cooldown period is *unenforceable*."
`dxGovernance P6`: "TimeDelay in `setAdminPermission` is not *enforceable*."

> For every check in the contract, answer two questions in writing:
> **who is this meant to constrain**, and **what is the cheapest way for that
> party to do the thing anyway?**

If you cannot answer the second question with "there isn't one," you have a
finding — regardless of whether the `require` is syntactically perfect.

---

## The seven failure modes

### 1. The constrained party controls the constraint

The purest form. A limit is only a limit if the limited party cannot move it.

- `Toucan-Celo-Bridge T2` — "Cooldown period is unenforceable as it serves to
  limit the admin but the admin himself controls it."
- `Blindex F1` — an oracle updated by an off-chain bot, with no safeguard
  beyond checking the caller's address. Omega calls it "one of the weakest
  links in the system" and asks for a bounded rate of change (≤1% per 24h).
  Note the *resolution*: an updater role capped at 1%/day was added, but "the
  owner still has the ability to make any changes to the oracle, which is a
  potential security concern" — Omega does not accept a partial fix as a fix.
- `Delphia C6` — "SCO operators control the price of the coordination tokens."

**Test:** for each parameter that bounds behaviour, find its setter and its
access control. If the setter is reachable by the party the parameter bounds —
directly, or through a role they can grant themselves — the bound is decorative.

### 2. A second address defeats it

Per-account state is not per-person state.

- `Blindex P2` — a 90% penalty for redeeming within `minimumMintRedeemDelay`.
  The user transfers the tokens to another account they own and redeems from
  there. Omega's recommendation is to *remove the penalty*, not patch it. The
  partial fix (revert instead of penalise) was still rejected: "the limitation
  is easily circumvented by transferring tokens to a different account."
- `Giza-ARMA-May A3` — a `top_up` endpoint writing a `Deposit` record if the
  amount ≤ wallet balance; call it repeatedly with a small balance to inflate
  the recorded deposit, which drives fee calculation.

**Test:** does the check key off `msg.sender`, an account, or a token balance?
All three are cheap to duplicate. Sybil-resistance requires something scarcer.

### 3. The check runs but its result is thrown away

Mechanically the most embarrassing, and Omega finds them at high severity.

- `Giza-Pendle CPP1` **[high]** — `get_swap_transaction` calls
  `_validate_slippage` to clamp slippage to `MAX_SLIPPAGE`, "however, the call
  does not save the result into the `slippage` variable … meaning the slippage
  check does not take effect."
- `Giza-Pendle M1` — the condition is inverted:
  `if not opt_res or self._is_delta_apr_enough(opt_res)` where
  `_is_delta_apr_enough` returns `True` when the APR is *not* enough.
- `Giza-Pendle W1` — the setter for `selected_chains` calls `_mark_dirty` with
  the `selected_protocols` key, so the new value is never persisted.
- `OpusEdu PP2` — "Allowed slippage is not enforced."
- `Altitude UVS1` — reads `swapPairs[assetTo][assetFrom]` while the mapping is
  written as `swapPairs[assetFrom][assetTo]`; every lookup reverts or returns
  the wrong pair.

**Test:** for each validator/clamp/normalizer function, trace its return value
to a use. For each boolean guard, evaluate the predicate's *name* against its
implementation — `_is_delta_apr_enough` returning `True` on "not enough" is a
naming bug that becomes a logic bug at every call site.

### 4. The flag is set but nothing reads it

- `dxGovernance F2` — `revocable = true` is passed when creating a
  `TokenVesting`. But the vesting contract's owner is its deployer, which is
  the factory, which has no function that calls `revoke`. "So no unclaimed
  token can be removed, whether the revoke flag is set to true or not."
- `dxGovernance P8` — "Unenforced requirement that `timeDelay > 0`."
- `Giza-Pendle JM1` — code references `DEFAULT_THRESHOLD`, which does not exist.
- `Gnosis XDFB4` — "Use of ignored token parameter may lead to unexpected
  behavior."

**Test:** grep every configuration flag and constructor parameter for reads.
Zero reads, or reads only in a branch that is unreachable, is a finding.

### 5. The allowlist enumerates the wrong thing

Trying to bound an *effect* by pattern-matching a *syntax*.

`dxGovernance W1` is the canonical example. A permission registry limits how
many ERC-20 tokens a proposal can move by extracting the function selector from
calldata and, if it is `transfer` or `approve`, decoding the amount. Omega's
demolition:

> There are ways of sending and approving tokens that are different from these
> two methods. Such methods live within the ERC20 standard (such as
> `transferFrom`), but also other methods are commonly used (such as
> `safeTransfer`, `increaseAllowance`). In addition, calls to transfer tokens
> could be triggered by calling other contracts.

And the recommendation states the general principle:

> Instead of trying to determine from the function signature and its arguments
> how many tokens were sent (a task that is impossible as the ERC20 contract
> can define any number of functions that do token transfer), **check the
> difference between the token balance and approvals before and after the
> transaction is executed.**

Measure the effect, not the syntax. Related: `dxGovernance P9` (permissions
specific to token transfer/approval are silently ignored when both an asset and
a function signature are set).

**Test:** if a guard inspects calldata, selectors, or function names, enumerate
the ways to achieve the same effect that it does not match — including
indirection through another contract.

### 6. Coverage is incomplete — the guard protects some paths, not all

- `Altitude C5` — safety mode disables `borrow` and `claimRewards`. Omega
  enumerates what it *should* also disable: `withdraw`, `transfer`,
  `liquidateUsers`, `depositAndBorrow`, `repayAndWithdraw` — every function
  whose correctness depends on the price feed that safety mode exists to
  distrust. (And separately notes `claimRewards` need not be disabled, since it
  does not depend on prices at all.)
- `Backed-Token-ERC4626 WBT1` — `_deposit` mints via
  `_beforeTokenTransfer(address(0), receiver, …)`. Since `from == address(0)`
  on mint, only the *receiver* is sanctions-checked, never the depositor. A
  sanctioned user launders through the vault into a clean address.
- `Backed-Finance G2` — "Blacklist does not extend to spender in
  `transferFrom`."
- `dxGovernance V9` — "Incomplete check in input value `voteDecision`."
- `Giza-Pendle ABA3` — a state check excludes three states when it should
  admit only one.

**Test:** for each guard, list every function that depends on the property the
guard protects, then diff against the functions that actually carry it. This is
the same consistency argument **[Q]** `semantic-guard-analysis` automates —
run both.

### 7. Two identical doors, one lock

- `Gnosis XDFB1` — `executeSignaturesUSDS` and `executeSignatures` have
  *identical* signature checks, so the same bridged message opens either. Both
  are permissionless. An attacker front-runs to choose which token the user
  receives. Compare `executeSignaturesGSN`, which does carry
  `require(isTrustedForwarder(msg.sender))` — the codebase's own inconsistency
  is the tell.
- `Blindex P9` — two redeem functions, same collateral out, one silently omits
  the BDX tokens.
- `Karpatkey K1` **[high]** — `cancelSubscription` and `cancelRedemption` never
  validate the *type* of the request ID passed. Create a redemption request
  with 1 wei of shares at an inflated price, then pass its ID to
  `cancelSubscription` and drain the subscription escrow. A shared ID namespace
  with no type tag.

**Test:** find near-duplicate entry points. Diff their guards. Any asymmetry is
either the bug or the evidence for it.

---

## Reporting

Omega's phrasing is worth copying because it separates the two claims:

> `P2. Redeem penalty of 90% is unenforceable [medium]`
>
> Users that call one of the various redeem functions within
> `minimumMintRedeemDelayInBlocks` after calling one of the mint functions will
> receive a 90% penalty. **The current implementation of the penalty is however
> not enforceable, since a user can redeem within the delay period by
> transferring the tokens to a different account they own, and redeem from that
> account.**

First the intent, then the bypass. Do not write "missing check" — the check is
right there, and the client will push back. Write *what the check was for* and
*the concrete sequence that defeats it*.

**Recommend removal when the guard cannot be made to work.** Omega does this
repeatedly — `Blindex P2` ("we recommend removing the 90% penalty logic"),
`Delphia C5`, `dxGovernance F2` ("set the flag to false for clarity"). A
security control that does not control anything is worse than none, because it
buys false confidence.

---

## Checklist

For every `require`, modifier, limit, penalty, cooldown, threshold and flag:

- [ ] Who is it meant to constrain? Written down.
- [ ] Can that party reach its setter, directly or via a role they can grant?
- [ ] Is it defeated by a second address, or by splitting into N transactions?
- [ ] Is the return value of every validator actually assigned and used?
- [ ] Does each predicate's name match its polarity?
- [ ] Does every configuration flag have a code path that reads it?
- [ ] If it matches on selectors or calldata, what equivalent effects slip past?
- [ ] Does it cover *every* function depending on the property it protects?
- [ ] Do near-duplicate entry points carry identical guards?
- [ ] Is a shared ID namespace type-checked at every consumer?

**Pairs with:** **[Q]** `semantic-guard-analysis` — it finds guards that are
missing by consistency analysis; this one finds guards that are present and
inert. Run the automated pass first, then this one over what it clears.
