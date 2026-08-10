---
name: omega-external-data-trust
description: Audit the boundary where a system accepts data it did not compute — oracles, third-party APIs, bridged messages, user-supplied request fields, and off-chain backends. Checks that returned values are validated against the constraints that were requested, that staleness is measured on the right timestamp, that independently-sourced fields are correlated, that errors are not swallowed into plausible-looking defaults, and that no security decision rests on a number the beneficiary supplied. Use when auditing oracle integrations, DEX/bridge quote consumption, keeper and relayer flows, agent backends, or any API-fed pricing or accounting logic.
---

# External Data Trust

Team Omega audits more than Solidity. Five Giza engagements review Python
agent backends; bridge audits review relayer assumptions; oracle audits review
the publisher's failure modes. This skill covers the boundary those reviews
share: **the moment a value the system did not compute becomes a value it acts
on.**

> For every external input, answer three questions: **what is this trusted
> for**, **what happens if it is wrong or stale**, and **who benefits from it
> being wrong?**

---

## 1. The response is not validated against the request

The single most transferable finding in the archive. You send a constraint with
your request; you must check the response honours it.

`Giza-Pendle BE1` — the bridge estimator sends `dstAmountMin` to the Stargate
API for slippage protection, "however the result from the API is not being
validated to ensure the API respected the `dstAmountMin` limit." Omega did not
assume — they ran the call, with `srcAmount=1e18` and
`dstAmountMin=950e18`, and the API returned `dstAmount: 998499000000000000`,
far under the stated minimum. The report includes the `curl` and the raw JSON.

Their generalisation is the important part: "even if it currently does, it
should not be assumed as it could become a risk in the future." An API's
present behaviour is not part of its contract.

**Test:** for every request carrying a bound — min out, max slippage, deadline,
recipient — find the line that re-checks it on the response. If it does not
exist, the bound is advisory.

### The compounding variant: protection applied to an unprotected number

`Giza-ARMA B2` **[critical]** — `_approve_and_swap` fetches a quote, then takes
0.5% off it as slippage protection. But "the quote received already includes a
slippage with it, which is not checked at all. This renders the whole slippage
protection essentially obsolete, and allows swaps with just about any degree of
slippage."

The same mistake recurs across engagements — `Giza-ARMA B5`, `F1` — and Omega's
recommendation is consistent and worth internalising:

> Use the slippage from the quote received, and **only cap** the total slippage
> at the limit, instead of setting it at the limit *below* the quote.

A hardcoded percentage is a *ceiling*, never a *basis*. Deriving a bound from a
number that is itself unbounded gives you no bound at all.

## 2. Staleness measured on the wrong clock

`Inverter iTRY G1` — the contract reads `latestRoundData()` but ignores
`updatedAt`, validating only the NAV publication timestamp from a separate
feed. Omega lays out the failure precisely:

> 1. At time t0, NAV is published and immediately pushed on-chain by Redstone
> 2. At time t1, a new NAV was published, but Redstone fails to push it on-chain
> 3. As long as the NAV publication date t0 is not expired, the feed returns
>    the price from t0.

Checking the publication timestamp proves the price was *published* recently.
It does not prove a *newer* price does not exist. Two different properties.

**Test:** for each timestamp the code checks, write down which property it
proves. Then write down the property you actually need. `updatedAt` (when it
reached the chain) and the publisher's own timestamp (when it was produced) are
not interchangeable, and neither implies freshness of the *latest available*
value.

Also from the same finding, Omega asks for a value sanity check alongside the
time check: "you are already checking that the price is not equal to 0, but you
can also check that it is not unreasonably high, e.g. require that the price is
less than 20% of the previous price." Related: `Karpatkey K6` ("Oracle stale
price check is too long"), `K12` ("Asset price of zero should revert").

## 3. Independently-sourced fields assumed to correspond

`Inverter iTRY G2` — the feed reads price from one contract and timestamp from
another "with no verification they correspond to the same data point. If the
relayer updates them at different times … you could get Thursday's price with
Friday's timestamp, and erroneously conclude that the price you are reporting
is not stale."

Note the recommendation acknowledges the obvious fix does not work: "Using the
`roundId` from the response will not work, as the Redstone contracts always
return a value of 1." Falling back to a tolerance check on the two `updatedAt`
values is second-best and Omega says so.

**Test:** whenever two values are read from separate calls or separate
contracts and then used together, ask what guarantees atomicity. Usually
nothing does.

## 4. Errors swallowed into plausible defaults

`Giza-Pendle G2` **[high]** — the most systemic finding in the archive. Across
the codebase, failures of external services return values indistinguishable
from success. Omega tabulates them:

| Method | Error handling |
|---|---|
| `_get_active_markets_by_chain` | Logs error, returns `[]` |
| `_get_asset_prices` | Logs error, returns `{}` |
| `is_market_expiring` | Logs warning, returns `False` |
| `get_market_apr` | Logs warning, returns `0.0` |
| `check_current_tvl` | Logs any exception, returns `0` |
| `check_to_execute` | Logs any error, returns `True` |

> This pattern is dangerous, as many of these values are used upstream as the
> good values. For example, if `_get_active_markets_by_chain` returns an empty
> list because the API is slow to respond, the markets from that chain will not
> be considered in the optimization problem.

Two things make this a *high*, not a code-quality note. First, `check_to_execute`
returning `True` on error means "the job is considered safe to execute, even if
the constraints are not met" — the default is fail-*open*. Second, Omega points
out that swallowed `ValidationError`s from the database "can point to a
systematic failure in writing data to the database correctly. If that is the
case … this is something to be resolved, not simply logged."

The recommendation:

> Raise errors where appropriate instead of only logging them. If this is a
> problem for reasons of liveness or otherwise, at least return values that are
> distinguishable from "good" values (e.g. `None` instead of an empty list) —
> but take care to handle these error values properly upstream.

**Test:** for every `except` / `catch` / unchecked call, ask what the caller
does with the returned value, and whether the failure default is fail-open or
fail-closed. Empty collections, zero, `False` and `0.0` are almost always
indistinguishable from legitimate results.

Solidity equivalents: `PrimeDAO-Seed S2` and `Stackly G2` (unchecked
`transfer`/`transferFrom` return values), `Altitude HM3` ("missing error check
on recognise farm rewards"), `Giza-Pendle O5` ("failing to apply a constraint
is silently ignored"), `Backed BR1` (code branches on a `False` return from a
function that reverts instead of returning `False` — so the branch is dead).

## 5. The beneficiary supplies the number

`Giza-ARMA B6` / `Giza-Pendle ABA1` — the user passes their deposit amount to
`activate`, and "this amount is not verified at any point." It then feeds TVL,
APR and fee calculations. Chain of consequences: `B6` → `B7` (TVL flawed) →
`B3` **[high]** (any user can inflate their fake deposit past the TVL cap and
halt agent activation for everyone).

Omega's fix is not "validate the number" — it is *stop asking for it*:

> Instead of asking the user to pass their deposit amount in the request, it
> will be safer and more accurate to use the balance as the deposit amount.

Related: `Giza-ARMA F2` (fee calculated on the backend but *paid* on the
frontend, with no verification — so users simply do not pay).

**Test:** for every value a caller supplies that feeds a security or economic
decision, ask whether the system could read it instead of being told it. If it
can, it should.

## 6. External identifiers assumed unique or stable

`Giza-Pendle O1` **[high]** — market names used as unique keys; the Pendle API
returns two markets named `wstETH` on mainnet. Recommendation: key on
`(address, chainId)`.

`EnterDAO S1` **[high]** is the on-chain analogue — the size of a staked
Decentraland estate is read from the external registry at both stake and
withdraw time. If it changed in between, withdraw reverts and the estate is
stuck; an attacker can *cause* the change by adding land to the staked estate.
Fix: snapshot the value at stake time, do not re-read it.

**Test:** every key drawn from an external system — is uniqueness guaranteed by
that system, or assumed? Every value read twice across time — is it required to
be equal, and what enforces that?

## 7. Off-chain service and backend surface

When the scope includes a backend (Omega's Giza engagements), also check:

- **Authorization binds the session to the subject.** `Giza-ARMA B1`
  **[critical]** — the JWT proves a user is logged in but is never checked
  against the wallet the endpoint acts on: "any user which is logged in can …
  activate, run and deactivate the agent of any other user."
  `Giza-Pendle P1` — two auth fields, `userAddress` and `wallet`, only the
  first verified, no check they are equal.
- **Key scope.** `Giza-Feb2025 M1` — "Jobs API key can access any endpoint of
  any wallet"; `M8` — "Data endpoints don't require authentication";
  `B4` — user tokens and job keys are interchangeable.
- **Timing-safe comparison.** `Giza-Feb2025 M7` — "Jobs API key validation is
  vulnerable to time analysis."
- **Determinism.** `Giza-Feb2025 C2` — "Query in `create_vaults` may miss some
  vaults, is indeterministic."
- **Freshness of the decision inputs.** `Giza-Feb2025 C1` **[high]** —
  "Decision processes may use outdated information on vault performance."
- **Logging.** `Giza-ARMA F3` — "Logs should exclude any sensitive data."
- **Dependency pinning.** `Giza-Pendle PP` — unpinned `pyproject.toml` makes
  builds unpredictable "and makes it easier for compromised new package
  versions to be included in the build."

## 8. Read the integrated protocol's own documentation

Some findings are invisible from the code under review.

`Altitude SPP1` — the strategy calls `getPtToSyRate` for its TWAP. Pendle's
docs say SY tokens are not always 1:1 wrappers of the underlying, and
specifically are not for stablecoin-wrapping SY — "which presumably are the
main target use case for the current system." The correct call is
`getPtToAssetRate`.

`Altitude SPB2` — "TWAP duration is shorter than what Pendle recommends."
`Altitude SM2` — "strategy expects the Morpho market to use supply and borrow
assets without verifying it does."

**Test:** for each integrated protocol, read its integration guide and list the
assumptions it warns about. Check each against the code.

---

## Checklist

- [ ] Every requested bound re-verified on the response
- [ ] No protection derived as a delta from an unbounded quote — caps only
- [ ] Staleness checked on the timestamp that proves the needed property
- [ ] Value sanity bounds (not just non-zero) on oracle reads
- [ ] Fields from separate sources correlated before joint use
- [ ] No error path returns a value indistinguishable from success
- [ ] Failure defaults are fail-closed, especially for go/no-go decisions
- [ ] No security or economic decision rests on a caller-supplied number that
      could be read instead
- [ ] External identifiers proven unique; re-read values proven stable or
      snapshotted
- [ ] Backend: session bound to subject; key scope minimal; comparisons
      timing-safe; queries deterministic; dependencies pinned; logs scrubbed
- [ ] Integrated protocols' own docs read and their stated assumptions checked

**Pairs with:** **[Q]** `oracle-flashloan-analysis` for manipulation mechanics
and oracle trust-model classification — this skill covers the *integration*
failures that occur even with a perfectly honest oracle.
