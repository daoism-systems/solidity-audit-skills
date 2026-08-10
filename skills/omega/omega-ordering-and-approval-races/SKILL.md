---
name: omega-ordering-and-approval-races
description: Find transaction-ordering vulnerabilities by cataloguing the six shapes Team Omega repeatedly finds — permissionless functions that consume someone else's approval, front-running an operator's approval to change the terms, racing between two entry points that accept the same authorization, imposing a penalty or state on a victim before their transaction lands, reverting a victim's transaction by consuming a bound first, and claiming a pending payout before an asset transfer. Use when auditing any two-step approve-then-act flow, auction or buyout, operator-approved request queue, bridged message execution, or capped mint or sale.
---

# Ordering & Approval Races

Front-running findings recur throughout the Omega archive, and they are not
generic "MEV" observations. They cluster into six recognisable shapes. Learning
the shapes is faster than reasoning from first principles at every call site.

> The governing question: **between the moment a user commits value or
> authorization and the moment it is consumed, what can a third party do?**

---

## Shape 1 — Approval granted, then anyone may act on it

The user approves a contract, intending a specific subsequent call. The call is
permissionless and its parameters are attacker-controlled.

- `Spectre V1` **[high]** — `fractionalize` on the vault is callable by anyone
  and requires the ERC-721 to be pre-approved. "An attacker could front-run the
  `fractionalize` call to the vault after the ERC721 owner approved the Vault
  for transfer, and pass their own parameters — for example, setting himself as
  admin and broker and essentially giving himself power to steal the token."
  Fix: restrict `fractionalize` to the NFT's owner.
- `Backed-atomic-swap A1` — "The signer can take all tokens that are approved to
  the swap contract."
- `Altitude AF1` **[high]** — the Aave flash-loan strategy's `flashLoan` is
  callable by anyone with any parameters. Aave calls back into
  `executeOperation`, which calls `executeFlashLoanLenderMigration` on the
  vault, migrating funds to an attacker-supplied `newStrategy` — bypassing the
  approval check in `migrateLender` entirely. The callback *is* the entry point.
- `Altitude SA1` **[high]** — `deposit` callable by anyone with any supported
  asset, corrupting `supplyPrincipal`.

**Test:** for every function reachable while a third party holds an outstanding
approval, ask who may call it and whether they choose the parameters. Also
enumerate *callback* entry points — flash-loan callbacks, ERC-721/1155 receivers,
CCIP receivers — as first-class attack surface.

## Shape 2 — Front-run the operator's approval

A privileged actor approves a request the user submitted. The user changes the
request between the approval transaction being broadcast and mined.

`Karpatkey K2` **[high]** — `updateRedemptionRequest` is allowed within
`updateRequestTtl`. The attack:

> 1. Create a redemption request
> 2. Wait for the operator to send a transaction in which the request is approved
> 3. Front-run the operator's transaction and call `updateRedemptionRequest` to
>    steal as much as is approved from the `portfolioSafe`
> 4. The operator's transaction will now be mined using the updated data.

Omega's recommendation analysis is worth copying. They propose the obvious fix
(require the update TTL to elapse before approval), then argue against it — "it
would mean each order would have to sit for likely at least a few days before it
can be safely approved, meanwhile the shares and assets prices are very likely
to change" — and point to a structural fix instead (`K4`: treat the user's price
as a *limit*, not an exact price, so the operator setting the price at execution
is safe). The resolution removed updates entirely.

**Test:** any queue where an operator authorizes an item the submitter still
controls. The operator's transaction must commit to the *content*, not just the
identifier — a hash of the request, or immutability from submission.

## Shape 3 — Two doors, one authorization

Two entry points accept the same proof; whoever calls first picks the outcome.

`Gnosis XDFB1` (filed twice, in `202504` and `202510`) —
`executeSignaturesUSDS` and `executeSignatures` "are essentially identical, with
the only difference being the former calls `onExecuteMessageUSDS` and the latter
calls `onExecuteMessage`. Because both functions have identical checks on the
signatures … an attacker can front-run a call to one function by calling the
other. The result is that the user will receive the wrong token."

Fix: bind the choice into the signed message. `Blindex P9` is the non-adversarial
cousin — two redeem paths, one silently worse, and the user picks wrong.

**Test:** find entry points with identical authorization checks and different
effects. The authorization must determine the effect.

## Shape 4 — Impose state on a victim before their transaction lands

The attacker does not steal directly; they move the victim into a state where
the victim's own pending transaction harms them.

`Blindex U2` **[high]** — `swap` sends 90% to the treasury if called before a
minimum delay has elapsed, and the delay counter is keyed on the *receiving*
address:

> An attacker can front-run a swap transaction from a user with a tiny swap to
> the "to" address that the user has specified. This will cause the counter of
> the minimum delay to be set for the victim, and if her transaction is mined
> before the minimum delay passed, it will send 90% of the swap value to the
> treasury address.

Omega's fix names the root cause: "set the counter based on the message sender
instead of the receiving address." Per-recipient state is attacker-writable
because anyone can send you something.

**Test:** any state keyed by an address that a third party can cause to be
written. `msg.sender` is attacker-controlled only for the attacker's own key;
`to`, `receiver`, `beneficiary` are writable for *anyone*.

## Shape 5 — Consume the bound so the victim's transaction reverts

Griefing by exhausting a cap or a precondition first.

- `Everbloom EN3` — mints beyond total supply revert. "If there are 1000 NFTs
  available, and a user is now sending a transaction to purchase all of them, an
  attacker could front-run their transaction and buy just 1 NFT, which will make
  the original buyer's transaction revert." Fix: let users request "up to N",
  not exactly N.
- `Altitude SG1` — `depositAll()` deposits the contract's whole Curve LP balance
  but approves only the newly-minted amount; send 1 wei of LP token directly and
  every deposit reverts.
- `Gnosis PUP1` — a strict `balanceOf(this) == amount` equality check that
  anyone can break permanently by sending 1 wei.
- `Altitude BV1` — "An attacker can DoS `borrowOnBehalfOf`."
- `Karpatkey K8` — "Malicious user can block the operator from stopping token
  deposits."

**Test:** every exact-match `require` and every cap. Ask who can change the
compared quantity from outside. Prefer `>=` over `==`, and "up to" over "exactly"
in user-facing amounts.

## Shape 6 — Claim the pending payout before the asset moves

An asset that carries an attached claim, transferred without settling it.

`EnterDAO L1` — a LandWorks NFT carries the right to claim accrued, unclaimed
rent. "A malicious buyer could negotiate a price with a seller to pay for the
NFT together with the claim on outstanding rent; subsequently, the seller can
front-run the token sale transaction and claim the outstanding rent fees before
the NFT is transferred." Fix: "assign the rent fee to the owner/consumer instead
of the asset, or automatically claim the rent fee before a transfer occurs."
The resolution added the payout into `transferFrom`/`safeTransferFrom`.

Related: `Altitude C3` and `C8` — transfer moving the harvest joining block for
both parties, so both forfeit their share of the current harvest.

**Test:** does any transferable asset carry an unsettled entitlement? Settle it
in the transfer hook, or key the entitlement to the account rather than the
asset.

---

## Adjacent: economic ordering, not just mechanical

Omega also files findings where reordering is legal but the *economics* are
broken:

- `Spectre B2` **[high]** — a buyout at a >100% multiplier pays a premium pro
  rata to holders, so an attacker mints new shares in front of the buyout to
  dilute into the premium. Omega prices it: an NFT bought out for $1000 against
  100 shares at $5 and a cap of 1000 — the attacker issues 50 shares to herself
  at $5.50 for $275 and claims $333. Severity: "all existing token holders will
  effectively be robbed of their share of the buyout premium."

  The recommendation is honest about difficulty: "This issue does not seem to
  have a straightforward fix. A possible solution would be to have a certain
  recurring time period … where trading and issuance are paused." Acknowledged,
  not fixed.
- `Spectre B3` — arbitrageurs, not LPs, capture the buyout premium.
- `Delphia C5` — bonding-curve mint/burn front-running.

When the fix is genuinely hard, say so and offer the least-bad option. Do not
pretend a mitigation is a fix.

---

## Recommendation patterns

Ranked as Omega applies them:

1. **Bind the authorization to the effect.** Put the token, the price, the
   recipient into the signed message or the approved request hash. (`XDFB1`,
   `K2`, `Society SPB3`.)
2. **Restrict the caller.** If only one party ever legitimately calls it, say
   so. (`Spectre V1`, `Altitude AF1`, `SA1`.)
3. **Key state to `msg.sender`, not to a recipient.** (`Blindex U2`.)
4. **Settle entitlements in the transfer path.** (`EnterDAO L1`.)
5. **Express user intent as a bound, not an exact value** — "at least X out",
   "up to N minted". (`Karpatkey K4`, `Everbloom EN3`.)
6. **Remove the racy feature.** Omega recommends this more often than not, and
   clients usually take it. (`Blindex P2`, `Karpatkey K2` resolution,
   `Giza-Pendle G5`.)

---

## Checklist

- [ ] Every function reachable while a third party holds an approval: caller
      restricted, or parameters not attacker-chosen
- [ ] Callback entry points (flash loans, ERC-721/1155/CCIP receivers) treated
      as public entry points
- [ ] Operator-approved requests immutable between submission and approval, or
      the approval commits to content
- [ ] No two entry points share an authorization but differ in effect
- [ ] No state is keyed by an address a third party can cause to be written
- [ ] No exact-equality check on a quantity an outsider can change
- [ ] Caps expressed as "up to", not "exactly"
- [ ] Transferable assets carrying entitlements settle them on transfer
- [ ] Premium/discount mechanics checked for dilution and arbitrage in front
- [ ] Where no clean fix exists, the least-bad mitigation is stated as such

**Pairs with:** **[Q]** `signature-replay-analysis` for EIP-712 domain binding
and nonce management, which is the standard fix for shapes 2 and 3 ·
**[P]** `hacking-agents/` for adversarial framing of the economic cases.
