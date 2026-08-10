---
name: omega-accounting-consistency
description: Find broken bookkeeping by checking that every counter, aggregate and index is updated on every path that changes what it measures. Catches double-counting from add-vs-assign confusion, counters decremented on some state transitions but not others, per-class or per-group totals not updated on the reversal path, irrecoverable drift that underflows later, aggregates derived from a stale or partial view, and double-spend where two parties both hold a claim on the same balance. Use when auditing staking, vesting, reward accrual, share accounting, proposal state machines, fee calculation, or any system with a running total.
---

# Accounting Consistency

Where `omega-asset-exit-paths` asks whether value can leave, this asks whether
the *numbers describing* the value are right. Team Omega finds a lot of these,
and rates them high when the drift is unrecoverable.

> For every state variable that summarises something — a count, a total, an
> index, an accrual — enumerate **every** code path that changes the underlying
> thing, and check that each one updates the summary correctly.

The bug is almost never in the path the tests cover. It is in the reversal, the
timeout, the partial fill, or the branch that was refactored.

---

## Failure modes

### 1. Assign where you meant to accumulate

- `Delphia D1` — `distribute` writes `withdrawBalances[user] = amount`,
  overwriting a previously distributed but unredeemed balance. Omega's fix is
  one character: "change line 89 from setting a value (with `=`) to adding to
  existing value (with `+=`)."
- `Society Protocol SVM1` **[high]** — same shape with worse consequences: the
  expired-lock branch does `userLock.amount = amount`, permanently orphaning
  the previous balance.

**Test:** every write to a balance-like mapping. Is `=` correct there, or should
it be `+=`? Ask specifically what happens on the *second* call.

### 2. Double counting — the callee already did it

- `Altitude HM1` **[high]** — `storeCommitCalculation` calls
  `HarvestHelper.userCalculateCommit` and *adds* the result to
  `commit.userHarvestUncommitted`. But `userCalculateCommit` already returns
  "the original values plus the changes." Omega traces the consequence past the
  wrong number: the later subtraction can underflow, and then "the user will
  also be unable to interact with any functions that call `commitUser`,
  including deposit, borrow, withdraw, and repay. Essentially leaving the user
  completely unable to interact with the system."

  Note the recommendation includes a *test* request: "We also suggest adding
  tests for `storeCommitCalculation`, as it is currently only tested as called
  from `commitUser`, but should also be tested when called independently."
  Helper functions tested only through one caller are where this hides.

- `Giza-Pendle O1` **[high]** — market *names* used as unique keys, but the
  Pendle API returns two distinct markets both named `wstETH` on mainnet
  alone. Same-named markets "will be counted double when calculating
  `weightedAPY_final` or when calculating the `cash_position`."

**Test:** for each `x += f(...)`, read `f`'s contract. Does it return a delta or
a new total? For each aggregation keyed by an identifier, prove the identifier
is actually unique in the domain it is drawn from.

### 3. The counter is decremented on some exits, not all

State machines leak here. Omega rates these high because the drift is one-way
and permanent.

- `dxGovernance V2` **[high]** — `orgBoostedProposalsCnt` is decremented when a
  proposal's execution state is `BoostedBarCrossed`, but *not* when a boosted
  proposal times out (`BoostedTimeOut`). "As there are no other ways to
  diminish the counter, this error is irrecoverable." Severity justification
  names the systemic consequence: the counter gates `MAX_BOOSTED_PROPOSALS`,
  so drift makes boosting progressively harder, "conceivably halting the
  operation of the DAO."
- `dxDAO VM15` — `preBoostedProposalsCounter` is reduced even when
  `MAX_BOOSTED_PROPOSALS` was reached and the state did *not* change, "which
  means the counter will be wrong, and might cause underflow error which will
  not be fixable."
- `dxDAO C2` — "Counter of schemes that can manage schemes is not updated
  correctly."

**Test:** draw the state machine. For every transition edge into and out of the
counted state, check the counter is adjusted. The bug is on the edge nobody
draws — timeout, cancel, revert-and-retry, force-close.

### 4. The reversal path forgets the aggregate

- `PrimeDAO-Seed S4` — `retrieveFundingTokens` refunds a funder but never
  decrements `classFundingCollected`. An attacker whitelisted for a class can
  buy and refund up to `softCap` repeatedly, permanently reducing what everyone
  else can buy. Fix: one line, `classFundingCollected -= fundingAmount`.
- `Spectre B4` — the sale state is not updated after `_escape_`, so the sale
  "remains active despite the NFT already being detached from it."
- `PrimeDAO-Seed S8` — "Counter of total funders could be manipulated upwards."

**Test:** for every function that increments an aggregate, find its inverse —
cancel, refund, withdraw, unstake, revoke — and confirm the aggregate is
decremented there. If there is no inverse function, that is a different finding
(see `omega-asset-exit-paths`).

### 5. Two parties hold a claim on the same balance

- `dxDAO VM13` **[high]** — a `redeem` refactor let stakers who backed the
  *losing* outcome redeem their full stake. "This breaks the staking logic, and
  will cause a double spending of the funds as both winners and losers now have
  a claim on the stake of the losers." The offending line is a default
  initialisation, `reward = staker.amount;`, that the winning branch was
  supposed to overwrite.
- `dxDAO VM1` **[high]** — `redeemDaoBounty` pays the DAO bounty out of the
  contract's staking balance, but nothing ever funds it, so the bounty comes
  from other users' stakes. Omega gives the full exploit: register a
  `paramsHash` with `minimumDaoBounty` equal to the contract balance, create a
  proposal, stake 1 token, pass it, redeem — drains everyone.

**Test:** sum every outstanding claim against the balance backing it. If
`Σ claims > balance` is reachable, the last claimant eats the shortfall
(`Altitude C1` is exactly this at protocol scale).

### 6. The formula is right for one step and wrong for the sequence

- `PrimeDAO-Seed S17` — recalculating `seedAmountRequired` when a class price
  changes. Omega does not argue abstractly; they walk three classes through a
  sale numerically (hardCap 1000; classes at price 10, 5, 4) and show the
  contract holds 146 seed tokens against a possible 150 sold. "There will not
  be enough seedTokens to cover the entire sale, and buyers can lose money."
- `Giza-ARMA-May A9` — "the piecewise approximation systematically
  underestimates the actual APR." Systematic, directional bias, not a rounding
  slip.
- `Giza-Pendle O4` — bridge costs amortised as `cost% × (365/days_to_expiry)`,
  treating a one-time principal charge as if spread over a year, and assuming
  positions are held to expiry. Two independent directional errors, both
  inflating APY.

**Test:** for any recurrence relation or incremental recalculation, run a
concrete multi-step scenario with real numbers. Ask whether the error is
*directional* — a systematic bias is much worse than noise, because it
compounds and always favours the same party.

### 7. The aggregate is computed from a source that does not represent it

- `Giza-ARMA B7` — TVL computed by summing user deposit records. Three separate
  reasons it is wrong: the records are user-supplied and unvalidated (`B6`),
  users can withdraw without the records changing, and yield earned is real
  TVL that never appears. "Needs to be refactored to rely on on-chain data."
- `Giza-Pendle SQLR1` — same finding recurring in the next engagement.
- `Karpatkey K7` — "Performance fee is based on first processed order share
  price" rather than the price that actually applied.

**Test:** for each aggregate, name its source of truth. If the source is a
mutable record written by a party with an interest in the number, it is not a
source of truth — see `omega-external-data-trust`.

---

## Reporting

Show the drift and its endpoint. Omega's `V2` is the model:

> In lines 508ff, the value of `orgBoostedProposalsCnt` is decreased in case the
> execution state of the proposal is `BoostedBarCrossed`. However, the counter
> is not diminished in case a Boosted proposal timed out … **As there are no
> other ways to diminish the counter, this error is irrecoverable.**

Then the severity clause carries the systemic consequence, not the arithmetic.
Nobody funds a fix for "a counter is off by one." They fund a fix for
"conceivably halting the operation of the DAO."

For arithmetic findings, **compute a worked example**. `S17` is persuasive
because you can check it with a calculator.

---

## Checklist

- [ ] Every balance-like write reviewed for `=` vs `+=`
- [ ] Every `x += f()` checked against whether `f` returns a delta or a total
- [ ] Every aggregation key proven unique in its actual domain
- [ ] State machine drawn; counter adjusted on *every* in- and out-edge,
      including timeout, cancel and failure edges
- [ ] Every incrementing function has its inverse, and the inverse decrements
- [ ] `Σ outstanding claims ≤ backing balance` shown to hold in all states
- [ ] Multi-step numeric scenario run for every incremental recalculation
- [ ] Directional bias checked for, separately from magnitude of error
- [ ] Each aggregate's source of truth named and shown to be authoritative
- [ ] Helper functions tested independently, not only through one caller

**Pairs with:** **[Q]** `state-invariant-detection` for automated inference of
`totalSupply = Σ balances`-style invariants · **[P]** `math-precision-agent`
for scale-mixing and rounding-direction analysis, which this skill deliberately
does not cover.
