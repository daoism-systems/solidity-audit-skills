---
name: omega-asset-exit-paths
description: Find permanently trapped funds by tracing every asset's exit path instead of hunting bug classes. For each asset that can enter a contract, enumerate the ways in and the ways out, then ask which reachable states have no way out and who can force the system into them. Catches stuck NFTs, unclaimable remainders, single-token failures that block whole batches, overwritten balances, push-payment blockades, and residue that can never be swept. Use when auditing vaults, escrows, auctions, staking, reward distribution, bridges, or any contract that custodies assets.
---

# Asset Exit Paths

Team Omega's highest-yield lens. Roughly a third of their high-severity
findings are some form of *value that went in and cannot come out* — and they
find them by asking a structural question, not by matching a vulnerability
signature.

> The question is not "is there a bug in `withdraw()`". It is: **for every kind
> of asset this contract can hold, and every reachable state, is there an
> account that can get it out — and can anyone else prevent that?**

## Why this beats a bug-class checklist

These findings do not share a bug class. They share a shape.

- `Fragmint A1` — an auction that only ever sends the NFT to a *winner*. No
  bids, no winner, NFT stuck forever. Nothing is "wrong" with any line.
- `Fragmint A2` — shares capped at ≤10,000; each share pays `1/10000` of the
  winning bid. Assign 9,000 shares and 10% of every winning bid is
  unreachable. The `require` is correct; the *bound* is the bug.
- `Society Protocol SVM1` — `lock()` on an expired lock takes the `else`
  branch and does `userLock.amount = amount`, overwriting the expired balance.
  Tokens gone. A `=` where the live-lock branch used `+=`.
- `dxDAO VM14` — the DAO's own share of staking rewards is allocated but the
  claim path was deleted in a refactor.
- `Gnosis PUP1` — `relayTokens` asserts `balanceOf(this) == amount`. Send 1 wei
  of DAI to the contract directly and the equality never holds again. The
  contract is bricked *and* the dust cannot be swept.

A checklist for "unbounded loop" or "unchecked return" catches none of these.
The exit-path question catches all five.

---

## The procedure

### 1. Inventory the assets

Every distinct thing of value the contract can hold:

- ETH — via `receive`, `fallback`, `payable` functions, `selfdestruct`
  force-feed, and as a coinbase recipient
- Each ERC-20 in a configured list, plus **any ERC-20 anyone can transfer in
  directly**
- NFTs (ERC-721/1155), via `safeTransferFrom` *and* plain `transferFrom`
- Internal claims: shares, credits, accrued rewards, pending-request escrow,
  vesting balances, buffer credit
- Assets held on the contract's behalf elsewhere — LP positions, aTokens,
  vault shares in an integrated protocol

> **Do not restrict yourself to assets the contract intends to hold.** Half of
> these findings involve an asset that arrived unexpectedly. `Altitude SG1`:
> anyone can DoS deposits by sending Curve LP tokens directly to the strategy,
> because the code approves only the newly-minted amount but calls
> `depositAll()`.

### 2. Map ways in and ways out

Build a two-column table per asset. Then look for asymmetry.

| Asset | Ways in | Ways out | Gated by |
|---|---|---|---|
| NFT | `deposit()` | `selectWinner()` → winner only | a bid existing |
| ETH | `bid()` | `transferRevenue()` loop | every shareholder accepting ETH |
| Reward token | `addRewards()` | `claim()` over the whole array | every token transferring successfully |

The Fragmint auction's table makes `A1` and `A3` obvious on sight.

### 3. Enumerate reachable states, then check exits in each

Not just the happy path. States Omega finds bugs in:

- **Empty / zero state** — no bids, no stakers, zero supply, first depositor
- **Expired / timed-out** — `Society SVM1` (expired lock), `dxGov V2`
  (proposal boosted then timed out)
- **Cancelled / paused / emergency** — `DXdao-staking D6` ("disable all
  functionality after contract is canceled"), `Altitude C5` (which functions
  are disabled in safety mode — and `C4`, whether you can even *enter* it)
- **Liquidated / insolvent** — `Altitude C1`: if the vault cannot repay, the
  remaining debt is socialised onto other borrowers and the last withdrawers
- **Partially filled** — `Fragmint A2` (shares < 100%), `Altitude FD1` (all
  strategies at max deposit, funds sit idle while interest accrues)
- **Sanctioned / blacklisted** — `Altitude AC2`: "debt of sanctioned users will
  remain stuck in the system"; `Backed-wrapped-tokens WBT3`: tokens of
  de-whitelisted addresses stuck in the wrapper
- **Post-upgrade** — a v2 that no longer honours a v1 balance

For each state × asset: **name the account that can get it out.** If you cannot
name one, you have a finding.

### 4. Ask who can block the exit

An exit that exists but can be blocked by a third party is not an exit.

**The single-element-fails-the-batch pattern.** Omega finds this constantly:

- `DXdao-staking D2` — `claim()` iterates all reward tokens and transfers each.
  One paused, malicious, or non-standard token and *nothing* is claimable.
- `DXdao-Carrot E1` — same shape, collateral tokens: "a single collateral token
  could block redeeming of all collateral of a token owner."
- `Blindex R1` — `getReward` fails after being called 128 times.

Recommendation Omega gives, every time: **make the per-item exit callable
individually.** Not "add a try/catch" — restructure so each token, each
shareholder, each collateral can be withdrawn on its own. Keep the batch
convenience path if you like, but the individual path must exist.

**The push-payment blockade.** Any loop that *sends* value to an
attacker-chosen address is a hostage situation:

- `Fragmint A3` — `transferRevenue()` loops `shares[i].account.transfer(fee)`.
  One shareholder deploys a contract that reverts on receive and both the NFT
  and the winning bid are frozen. Omega notes the attacker can register the
  address *before deploying the contract*, so it is undetectable at
  registration time — and prices the extortion at up to twice the NFT's value.
- `Fragmint A6` — same function, gas-exhaustion variant.

**Unbounded iteration** over a list that grows or that users can extend:
`DXdao-staking D1`, `Blindex R2` (`withdrawLocked` runs out of gas),
`DXdao-Carrot KM5`, `Everbloom EN5`, `dxDAO C6`.

**Griefed entry, not just exit.** `Altitude SG1` (dust blocks deposits),
`Altitude I1` ("an attacker can block withdrawals and borrows at any time"),
`Everbloom EN3` (front-run a bulk mint with 1 NFT so the victim's tx exceeds
supply and reverts), `Karpatkey K8` (malicious user blocks the operator from
stopping deposits).

### 5. Check the exit returns the *right* asset, to the *right* place

An exit that fires but sends the wrong thing is still a loss:

- `Altitude SA3` — `redeem` takes a `borrowAsset` parameter and is callable by
  anyone; an attacker passes a different approved-but-wrong token, the rewards
  are swapped into it, and `recogniseRewardsInBase` can no longer see them.
- `Altitude SA6` — "wrong token swapped on redeem."
- `Gnosis XDFB1` — two identical entry points accept the same bridged message;
  whoever calls first decides whether the user receives DAI or USDS.
- `Blindex P9` — `redeem1t1BD` pays out the same collateral as
  `redeemFractionalBdStable` but *without* the BDX tokens. Using the wrong one
  of two valid functions silently forfeits value.

### 6. Check the residue

After every normal operation completes, what is left over, and can it be
recovered?

- `OpusEdu PP3` — overpay when buying a course and the excess "will be lost
  forever."
- `Gnosis-Hashi G4` — "Ether can remain locked in contracts."
- `Gnosis PUP1` — dust that both bricks the contract and cannot be swept.
- `dxGovernance V3` — ETH sent by a caller with unset refund parameters is
  stuck.
- `Altitude SA3` again — the recovery path exists, but only via an owner
  reconfiguring `setVault`, which Omega notes "could cause serious disruptions
  in the operation of the vault." A recovery path that requires breaking the
  system is not a recovery path.

---

## Reporting

State it as a lifecycle, not a line number:

> `A1. NFT that did not get any bids will be stuck in the contract [high]`
>
> The auction contract has only a single way to send out an NFT, and that is to
> the winner of the auction. However, if no bids for the NFT were sent to the
> contract, the original depositor has no way to claim back their NFT.

Mechanism (only one exit), the state that has no exit (no bids), consequence
(stuck forever). Then the recommendation names the missing path:

> Allow the depositor of the NFT to withdraw the NFT if the auction ends and no
> bids were placed.

Severity per Omega's ladder: permanently unrecoverable value is **high**;
recoverable-but-only-by-privileged-intervention or improbable-precondition is
**medium** (`Fragmint A6`: "although loss of funds is possible, the scenario of
having so many shareholders seems quite improbable").

---

## Checklist

- [ ] Every asset the contract *can* hold inventoried, including unexpected ones
- [ ] Ways-in / ways-out table built per asset
- [ ] Exit checked in every reachable state: empty, expired, cancelled, paused,
      liquidated, partially filled, sanctioned, post-upgrade
- [ ] For each state, the account that can exit the asset is named
- [ ] No batch exit depends on *every* element succeeding
- [ ] No exit pushes value to an address an attacker chooses
- [ ] No exit iterates an unbounded or user-extendable list
- [ ] Exits deliver the right asset to the right recipient
- [ ] Entry cannot be griefed by direct transfers or supply front-running
- [ ] Residue, dust and rounding remainders are sweepable
- [ ] Recovery paths do not require disrupting the system to use

**Pairs with:** **[Q]** `dos-griefing-analysis` for push-vs-pull and 63/64
gas mechanics · **[L]** `depth-edge-case` for systematic zero/max/boundary
enumeration of the state space.
