---
name: external-call-safety
description: Detects unsafe external call patterns and token integration vulnerabilities via disciplined per-call-site enumeration — unchecked call/delegatecall/staticcall returns, no-code receivers, non-standard and weird ERC20 behavior (fee-on-transfer, rebasing, USDT void returns, false-returning tokens, ERC-777 hooks), sentinel-address branches, payable edge cases, corrupted bytes inputs, and push-vs-pull payment patterns. Use when auditing contracts that call external contracts, integrate arbitrary ERC20 tokens, or distribute payments.
---

# External Call Safety

Exploits the gap between assumed and actual behavior at external
boundaries. Two lenses merged: **[P]** pashov's attacker discipline — walk
every call site, branch and input source and apply a fixed set of
corner-case questions until none are unexamined — and **[Q]** quillshield's
cataloged vulnerability classes and weird-ERC20 reference. Record the lens
tag in each finding's `rationale` field.

Other agents specialize by bug category. The [P] lens specializes in
**methodology**: the same questions applied to EVERY boundary point. The [Q]
lens supplies the token-behavior catalog the questions resolve against.

## When to Use

- Auditing any contract that calls external contracts (token transfers, cross-contract interactions)
- Reviewing protocols that support arbitrary/user-supplied ERC20 tokens
- Analyzing ETH payment distribution logic (airdrops, reward distribution, refunds)
- Verifying low-level call safety (`call`, `delegatecall`, `staticcall`)
- When a protocol claims to support "any ERC20 token"

## When NOT to Use

- Reentrancy-specific analysis (use reentrancy-pattern-analysis — though there is overlap)
- Oracle/price feed analysis (use oracle-flashloan-analysis)
- Pure access control review (use guard-consistency)

## Part A — Boundary enumeration [P]

### Step 1 — Enumerate every boundary

For each contract in scope, list every:
- External call site (`target.foo(...)`, `.call{...}(...)`, `.staticcall(...)`, `.delegatecall(...)`)
- Payable function (`external payable` or `public payable`)
- Function with a sentinel-address branch (`if (addr == address(0))`, `if (token == _ETH_ADDRESS_)`, similar)
- Function that takes a token/contract address as parameter (from caller, decoded message, or storage)
- Function with a `bytes` / `bytes calldata` input that is decoded
- Any place an external return value is consumed by caller logic

This list is your work plan. Apply the following steps to every entry.

### Step 2 — For every external call: ten corner cases

1. **No code at receiver.** What if `address.code.length == 0`? Low-level
   `.call(...)` returns `(true, "")`. `IERC20(addr).approve(...)` reverts.
   `safeTransfer(addr, ...)` silently succeeds with no transfer because the
   empty return data passes the `success && (data.length == 0 || decoded == true)` check.
2. **Non-standard token.** Void return (USDT-style) breaks
   `require(token.transferFrom())`. Fee-on-transfer makes received amount ≠
   requested amount. Rebasing makes cached balance stale. Blacklist/pausable
   makes standard transfers revert unexpectedly. Some tokens revert on
   zero-value approval.
3. **Empty / zero / max input.** Zero amount — does the code skip, revert,
   or proceed wrongly? Empty bytes — does abi.decode revert? Max uint —
   does the math overflow before the check?
4. **Return-value handling.** Does the caller validate the return? Ignored
   bool return = silent failure. Misinterpreted custom-error returndata.
5. **Sentinel-placeholder used in IERC20 op.** Native-token placeholder
   addresses (`_ETH_ADDRESS_`, `address(0xeee...)`) flow into raw
   `IERC20(token).approve(...)` and revert because the placeholder has no
   contract code. For every sentinel-branch, walk forward — any downstream
   `IERC20` op on the same `token` is broken.
6. **False-returning ERC20.** Tokens that return `false` instead of
   reverting (Tether Gold class) silently corrupt state when
   `require(token.transfer(...))` is omitted. Distinct from USDT-style void
   return — both must be checked.
7. **ERC165 dispatch fallback.** Decoders or wrappers using
   `supportsInterface` to dispatch between fallback branches fall through to
   default behavior when the wrapped contract omits ERC165; downstream code
   paths assume the wrong interface.
8. **ERC721 hook re-entry.** `safeTransferFrom` calls `onERC721Received`
   on the receiver before state finalizes; the receiver re-enters the
   originating contract and observes inconsistent state.
9. **Unrestricted external call from custody.** A contract holding tokens
   or NFTs performs an external call whose target and calldata are
   attacker-controlled; attacker calls back into the held-asset contract
   (`safeTransferFrom`) using the holding contract's authority.
10. **Caller-supplied fee/bonus has no upper bound.** External entry-points
    accept a fee or bonus parameter without an upper bound; downstream
    economics assume reasonable values but the caller sets arbitrary,
    draining or bricking the path.

For every call site that fails any question in a way the calling code
doesn't account for — finding.

### Step 3 — For every payable function: branch cases

1. `msg.value > 0` — is the value spent, refunded, or forwarded? Where does it end up?
2. `msg.value == 0` — does the operation still proceed when no native was sent? Does it skip a fee that should have been paid? Does it pull tokens it shouldn't?
3. `msg.value != amount` (when both exist) — is the relationship enforced? `msg.value > amount` (excess stuck in contract). `msg.value < amount` (under-payment proceeds while accounting believes amount was paid).
4. **Native-path fee not deducted.** When both `amount` and `fee` exist, the native branch often forwards `msg.value` raw while the ERC20 branch deducts `fee` from `amount`. Downstream consumers assume pre-fee value was paid.

### Step 4 — For every sentinel-address branch: walk both sides

1. Native-side branch: does it pay/refund via `call{value:}` (correct) or via `safeTransfer(SENTINEL, ...)` (silent no-op)?
2. ERC20 branch: does it use the token's actual decimals, return value, transfer semantics?
3. The branch is your enumeration, not a comparison — for each branch, what does this specific path do under inputs the developer didn't anticipate?

### Step 5 — For every bytes input / abi.decode: corruption cases

1. Empty input — panic? Bypass a loop? Return defaults that look like valid empty state?
2. Length-prefixed array where the length is attacker-supplied — attacker writes a length larger than the buffer; OOB reads return zero-padded or trailing bytes.
3. `bytes20(longerBytes)` cast — silent truncation. Source can be longer than 20 (BTC bech32, Solana 32-byte, attacker-chosen length).
4. `abi.encodePacked` followed by `abi.decode` — packed encoding is ambiguous; decode returns wrong field boundaries.
5. Field-order mismatches across encode and decode sites in different files — silent reinterpretation of attacker bytes.

## Part B — External call classes [Q]

### Class 1: Unchecked return values

```
For each low-level call expression:
  1. Is the return value captured? (bool success, bytes memory data) = ...
  2. Is the success boolean checked? require(success) or if(!success) revert
  3. If not captured or not checked → UNCHECKED RETURN VALUE

Severity:
  - ETH transfer unchecked → CRITICAL (funds lost)
  - Token operation unchecked → HIGH (state desync)
  - Non-financial call unchecked → MEDIUM
```

### Class 2: Gas stipend limitations

`transfer()` and `send()` forward only 2300 gas — reverts if the recipient's
`receive()`/`fallback()` does more than emit an event (EIP-1884 SLOAD
repricing broke existing contracts; multi-sig and smart-contract wallets
often need more). Use `call{value: amount}("")` with the return checked.

### Class 3: Return data bomb

A malicious contract returns extremely large data to consume the caller's
gas: `(bool success, bytes memory data) = untrustedContract.call(calldata)`
copies 1MB+ at the caller's expense. Protection: ignore the data, or cap
the returndatacopy size in assembly (see
[references/call-safety-patterns.md](references/call-safety-patterns.md)).

### Class 4: Delegatecall to untrusted contract

`target.delegatecall(data)` executes untrusted code in OUR storage context —
attacker overwrites ANY storage slot. Delegatecall should ONLY be used with
trusted, immutable targets.

## Part C — Token integration safety ("weird ERC20") [Q]

1. **Fee-on-transfer** (STA, PAXG, USDT-capability, RFI/SAFEMOON forks):
   recipient receives less than `amount` — credit the measured
   balance-delta (`balanceAfter - balanceBefore`), never the input
   parameter.
2. **Rebasing** (stETH, AMPL, OHM, YAM, BASED): balances change without
   transfers — storing absolute amounts desynchronizes accounting.
   Mitigate with shares, wrapping (wstETH pattern), or explicit
   non-support.
3. **Missing return values** (USDT, BNB, OMG, legacy KNC):
   `bool success = token.transfer(...)` reverts on tokens returning
   nothing — use SafeERC20.
4. **Tokens with callbacks** (ERC-777 `tokensToSend`/`tokensReceived`):
   any state change after a transfer that triggers callbacks is
   reentrancy-exposed (cross-reference reentrancy-pattern-analysis).
5. **Unsafe approve** — the classic race (spender uses old allowance then
   new allowance = double spend); some tokens (USDT) revert on
   non-zero-to-non-zero approve — reset to 0 first or use
   `forceApprove`/`safeIncreaseAllowance`.
6. **Blacklists** (USDC, USDT, TUSD, BUSD): transfers to/from blacklisted
   addresses revert — one blacklisted recipient bricks an entire push
   distribution; handle per-recipient failure (try/catch, pull pattern).
7. **Max supply / transfer limits** (anti-whale tokens): protocols that
   batch large transfers can silently hit limits.

The full ten-category catalog with affected tokens and the compatibility
matrix is in [references/weird-erc20.md](references/weird-erc20.md).

## Part D — Payment pattern analysis [Q]

```
PUSH (dangerous): contract sends funds TO recipients
  - Fails if recipient is a contract that reverts
  - One malicious recipient DoS's the whole distribution
PULL (safe): recipients claim FROM contract
  - Each claim independent; one user's failure doesn't affect others

For each function sending ETH/tokens to external addresses:
  Sending to user-supplied addresses in a loop → PUSH → HIGH DoS risk
```

## Discipline [P]

For each finding, state THREE things:
- The **boundary** you exercised (which call site / branch / input)
- The **assumption** the calling code makes about the boundary's behavior
- The **actual behavior** under the corner-case input you supply

Without all three, it's a LEAD.

## Workflow

```
Task Progress:
- [ ] Step 1: Build the Part A boundary enumeration (call sites, payables, sentinel branches, bytes inputs)
- [ ] Step 2: Apply the ten corner cases to every external call site
- [ ] Step 3: Apply payable / sentinel / bytes checks to those boundary types
- [ ] Step 4: Classify token assumptions per interaction against the weird-ERC20 catalog
- [ ] Step 5: Verify SafeERC20 usage, balance-before-after, approve patterns
- [ ] Step 6: Analyze payment distribution pattern (push vs pull)
- [ ] Step 7: Score findings and generate report
```

## Output format

```markdown
### Finding: [Title]

**Function:** `functionName()` at `Contract.sol:L42`
**Category:** [Unchecked Return | No-Code Receiver | Non-Standard Token | Fee-on-Transfer | Rebasing | Missing Return | Callback | Approve Race | Sentinel Branch | Data Corruption | DoS]
**Severity:** [CRITICAL | HIGH | MEDIUM]
**Lens:** [P] / [Q]

**Boundary:** which call site / branch / input was exercised
**Assumption:** what the calling code assumes the boundary does
**Actual:** what the boundary actually does under the corner-case input

**Affected Tokens:** [USDT, USDC, stETH, ...]

**Attack Scenario:**
1. [Step-by-step exploitation]

**Recommendation:** [SafeERC20, balance-before-after, pull pattern, require success, ...]
```

## Quick detection checklist

- [ ] Are ALL low-level `call` return values checked (`require(success)`)?
- [ ] Does the protocol use `SafeERC20` for all token interactions?
- [ ] Does deposit use the balance-before-after pattern for fee-on-transfer tokens?
- [ ] Does the protocol explicitly handle or reject rebasing tokens?
- [ ] Does `approve()` reset to 0 before setting a new allowance (USDT compatibility)?
- [ ] Are batch payment operations using the pull pattern (not push)?
- [ ] Is `delegatecall` only used with trusted, immutable targets?
- [ ] Are return data sizes from untrusted contracts limited?
- [ ] Does the protocol handle token blacklisting gracefully?
- [ ] For every sentinel-address branch: walked both sides, downstream IERC20 ops on the sentinel traced?
- [ ] For every `bytes` decode: empty/oversized/truncated inputs considered?

## Rationalizations to reject

- "We only support standard ERC20 tokens" → USDT is the most used token and it's non-standard (no return value, fee capability)
- "The call will always succeed" → Smart contract wallets, blacklisted addresses, and gas changes can cause failures
- "We trust the token contract" → Token contracts can be upgraded (proxies) or have hidden features
- "transfer() is safe enough" → 2300 gas stipend breaks with gas repricing EIPs; use call()
- "We checked the token before listing" → Fee-on-transfer can be toggled on after listing (USDT has this capability)
- "Rebasing tokens are rare" → stETH is one of the largest tokens by TVL

## References

- [references/weird-erc20.md](references/weird-erc20.md) — ten-category token behavior catalog with compatibility matrix
- [references/call-safety-patterns.md](references/call-safety-patterns.md) — call/delegatecall/staticcall patterns, ETH transfer comparison, safe patterns
