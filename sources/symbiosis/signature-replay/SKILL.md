---
name: signature-replay
description: Detects signature verification and replay vulnerabilities in smart contracts across chains — covering the five replay types (same-chain, cross-chain, cross-contract, nonce-skip, expired-signature), ecrecover edge cases, EIP-712 domain separation, permit/Permit2, ERC-1271, malleability, and signature-to-derived-ID binding on multi-sig, Merkle-aggregate and cross-VM systems. Use when auditing contracts with ecrecover, ECDSA, EIP-712, permit, meta-transactions, signed orders, or any off-chain signature verification.
---

# Signature & Replay Analysis

Signature bugs span nine-plus sub-classes that interact with each other (a
missing nonce plus missing chain binding equals cross-chain replay). This
skill traces every signature from construction through verification to
consumption: **[Q]** quillshield's cataloged replay taxonomy and EVM check
lists, **[L]** plamen's multi-chain checklist set (including the
ID-binding and Merkle-linkage classes no other skill covers). Record the
lens tag in each finding's `rationale` field.

**Prevalence**: 19.63% of Ethereum contracts using signatures contain replay
vulnerabilities.

## When to Use

- Auditing contracts that verify signatures (`ecrecover`, ECDSA, EIP-712)
- Reviewing ERC-20 `permit()` / Uniswap Permit2 implementations
- Analyzing meta-transaction / gasless relay systems
- Verifying multi-sig signature aggregation
- Checking off-chain order books or signed message execution
- Any system where a signed object also carries a derived ID (tx hash, commitment, attestation root)

## When NOT to Use

- Contracts without any signature verification
- Pure on-chain access control (use guard-consistency)
- Token standard compliance (use external-call-safety)

## The signature trust model [Q]

A signature proves a specific private key holder authorized a specific
action. To be secure it must be:

1. **Bound to context** — specific chain, contract, and version (domain separation)
2. **Used exactly once** — nonce prevents replay
3. **Time-limited** — deadline/expiry prevents late execution
4. **Correctly verified** — ecrecover edge cases handled

Any gap creates a replay vulnerability.

## The five replay types [Q]

**Type 1 — Same-chain replay**: the exact same signature submitted multiple
times to the same contract. No nonce → signature works forever. Safe: nonce
in the signed hash, checked and incremented.

**Type 2 — Cross-chain replay**: signature valid on one chain replayed on
another where the same contract is deployed. Signed message without
`chainId` → identical hash on every chain. Safe: EIP-712 domain separator
including `block.chainid`.

**Type 3 — Cross-contract replay**: signature for Contract A replayed on
Contract B on the same chain when both accept the same message format.
Signed message without `verifyingContract` → same hash for any contract on
this chain. Safe: domain separator includes `address(this)`.

**Type 4 — Nonce-skip replay**: nonce implementation allows gaps or
out-of-order execution — bitmap nonce without invalidation means a skipped
nonce can be used anytime in the future (may be intentional or a
vulnerability depending on context). Safer for strict ordering: sequential
`require(nonce == nonces[signer]); nonces[signer]++;`.

**Type 5 — Expired-signature replay**: signature without a deadline held
and executed at an arbitrary future time when conditions have changed.
Safe: `require(block.timestamp <= deadline, "Expired");`.

Full vulnerable/safe Solidity for each type:
[references/replay-taxonomy.md](references/replay-taxonomy.md).

## ecrecover safety [Q]

1. **Returns address(0)** on invalid signature instead of reverting — if
   `owner == address(0)` an invalid signature passes. Require
   `signer != address(0)` explicitly, or use OpenZeppelin's
   `ECDSA.recover()` which reverts.
2. **Malleability**: for every valid ECDSA (r, s, v), (r, n-s, v^1) is also
   valid. If signatures are used as unique identifiers (mapping keys,
   dedup), malleability allows replay. Enforce lower-s (`s <= n/2` —
   OpenZeppelin's ECDSA does this; EIP-2 enforced it at protocol level).
3. **v value**: should be 27 or 28; some implementations use 0/1. Not
   normalizing v causes verification failures.

## CHECK 1 — Validation completeness [L]

For EACH verification call site:

| Call Site | Invalid Signature Handled? | Signer Recovery Validated? | Nonce Verified? | Deadline Checked? | Scope Bound? | Gap? |
|-----------|--------------------------|---------------------------|-----------------|-------------------|-------------|------|

Chain-specific verification functions:
- **EVM**: `ecrecover` returns `address(0)` on invalid — must check return
  != address(0). `ECDSA.recover` reverts on invalid (safer).
- **Solana**: `ed25519_program` instruction introspection — verify the
  instruction exists in the transaction AND the signed data matches
  expectations. Missing verification = anyone can claim any signature.
- **Aptos**: `ed25519::signature_verify_strict` returns bool — must check
  return value. `multi_ed25519::verify` for multisig schemes.
- **Sui**: `ecdsa_k1::secp256k1_verify` / `ed25519::ed25519_verify` return
  bool — must check return value.

## CHECK 2 — Replay protection / nonce management [L]

For EACH nonce-based or flag-based replay guard:

| Replay Guard | Type (nonce/mapping/bitmap/flag) | Incremented/Set Before Use? | Can Be Reused? | Shared Across Functions? | Gap? |
|-------------|--------------------------------|---------------------------|----------------|------------------------|------|

- Sequential nonces: verify increment happens BEFORE or DURING validation, not after external calls
- Mapping-based (used[hash]): verify the key is unique per message, not just per signer
- Can a signature be used across different functions that share the same replay protection space?
- **Solana**: if using instruction introspection for ed25519 verification, check the SAME transaction cannot include the ed25519 instruction twice with different signed data
- **Aptos/Sui**: if replay protection uses a `Table` or `VecMap`, check for key collision across different message types

## CHECK 3 — Scope binding [L]

Verify each signature is bound to the intended chain, contract/program/module, and operation:

| Signature | Chain-Bound? | Contract-Bound? | Function-Bound? | Gap? |
|-----------|-------------|-----------------|----------------|----|

- **EVM (EIP-712)**: Domain separator must include `chainId` (recomputed on
  fork) and `verifyingContract` (`address(this)`). Cached at deployment and
  not recomputed on `block.chainid` change → cross-chain replay. Hardcoded
  address → breaks on proxy upgrade.
- **Solana**: signed message must include the program ID — otherwise a
  signature from one program can be replayed on another.
- **Aptos**: signed message should include the module address
  (`@module_addr`); resource-account addresses are deterministic — verify
  signed data cannot be replayed on a different resource account with the
  same seed.
- **Sui**: signed message should include the package ID; after package
  upgrade, signatures from old versions must not work on the new package.

General: signed message omitting the function/operation identifier →
signature valid for different operations within the same contract; omitting
a unique identifier (nonce, timestamp, tx hash) → replayable. For
meta-transaction relayers: does the relayed call include the target address
in the signed data?

### EIP-712 domain separator [Q]

| Field | Purpose | Missing = |
|-------|---------|-----------|
| `name` | Identifies the signing domain | MEDIUM risk |
| `version` | Prevents replay across upgrades | MEDIUM risk |
| `chainId` | **Prevents cross-chain replay** | HIGH risk |
| `verifyingContract` | **Prevents cross-contract replay** | HIGH risk |
| `salt` (optional) | Additional disambiguation | LOW risk |

Common mistakes: hardcoded `CHAIN_ID` (after a fork, signatures valid on
both chains — use `block.chainid` at verification time or recalculate the
domain separator); empty name/version (weak binding); missing struct
typeHash in the message. Full checklist:
[references/eip712-checklist.md](references/eip712-checklist.md).

## CHECK 4 — Off-chain approval patterns [L]

If the protocol accepts off-chain authorizations (permits, gasless
approvals, signed orders, meta-transactions):

| Approval Type | Front-Run Resistant? | Fallback on Failure? | Deadline Enforced? | Revocable? | Gap? |
|-------------|---------------------|---------------------|-------------------|-----------|----|

- **EVM (EIP-2612 permit)**: `permit() + transferFrom()` in same tx can be
  front-run — attacker calls `permit()` first, user's tx reverts. Safe
  pattern: wrap permit in try/catch, fall back to existing allowance.
- **Solana (signed orders)**: can an attacker submit the signed instruction
  before the intended user? Can the order be partially filled and replayed?
- **Aptos/Sui (signed messages)**: can the message be submitted by anyone,
  or only the signer? Is there an expiry?
- **All chains**: does the protocol REQUIRE the off-chain authorization to
  succeed, or gracefully handle front-running/races?

**Permit2 [Q]**: nonce-bitmap approach (unordered nonces), batch permits
and transfer-with-permit; still requires deadline, domain separator, nonce
management; integrations must verify the permit2 contract address.

## CHECK 5 — Malleability [L]

| Verification | Malleable? | Signatures Used as Keys/IDs? | Framework-Wrapped? | Gap? |
|-------------|-----------|------------------------------|-------------------|------|

- **ECDSA (EVM)**: for any valid (r, s, v), (r, n-s, v^1) is also valid.
  If signatures are used as unique identifiers (mapping keys, dedup sets),
  malleability allows bypass. OpenZeppelin's `ECDSA.recover` enforces
  `s <= n/2` — check if the protocol uses raw `ecrecover` without this
  bound.
- **Ed25519 (Solana/Aptos/Sui)**: NOT malleable with strict verification
  (`ed25519_dalek` `verify_strict`); non-strict verification may accept
  multiple valid signatures for the same message. Check which function is
  used.
- **All chains**: if signatures are stored/compared as bytes, ANY
  malleability allows bypass; if only used for signer recovery, it is not
  exploitable.

## CHECK 6 — Cross-chain and cross-protocol replay [L]

| Signature | Chain-Bound? | Protocol-Bound? | Version-Bound? | Gap? |
|-----------|-------------|-----------------|---------------|----|

- Signed data without a chain identifier → valid on any chain with the same protocol deployed
- Signed data without the protocol/program/module address → valid on any protocol using the same message format
- Multi-chain deployments: are signatures from Chain A replayable on Chain B?

## CHECK 7 — Deadline and expiry [L]

| Signature Type | Has Deadline? | Deadline Enforced On-Chain? | Can Be 0 or MAX? | Gap? |
|---------------|--------------|----------------------------|-----------------|------|

- Signatures without deadlines are valid forever — even after key rotation, role revocation, permission changes
- Can the deadline be set to maximum (`type(uint256).max`, `u64::MAX`), effectively permanent?
- Time-source correctness: EVM `block.timestamp` (off-by-one `>=` vs `>` extends validity by 1 block); Solana `Clock::unix_timestamp` (slot-vs-timestamp confusion); Aptos `now_seconds()` vs `now_microseconds()` (unit mismatch); Sui `clock::timestamp_ms()` (milliseconds, not seconds).

## CHECK 8 — Consumption ordering [L]

| Operation | Signature Checked Before State Change? | External Callbacks Safe? | Gap? |
|-----------|---------------------------------------|-------------------------|------|

- Verify signature validation occurs BEFORE any state changes (checks-effects-interactions)
- If signature verification involves external calls, check reentrancy:
  EVM `isValidSignature` (ERC-1271) calls an external contract — reentrancy
  vector if state is modified before the call; Solana CPI to ed25519
  program is safe (system program) but CPI to a custom verification program
  could be malicious; Aptos/Sui external module calls — check friend
  functions / public entry points for re-entry.

## CHECK 9 — Signature-to-derived-ID binding [L]

**Catches**: systems where a transaction, block, or message has both a
signature field AND an ID field derived from "the signed content," but the
verifier does NOT recompute the derived ID. An adversary with a valid
signature over payload P can set the ID field to any value; the signature
still verifies.

**Why this is a whole class**:
1. Bitcoin signature malleability (BIP-62, BIP-141, pre-2015): ECDSA (r, s)
   has trivial second valid form (r, n−s). Pre-BIP-62, a malleated
   signature produced a DIFFERENT `txid` with the SAME semantics.
   (Mt. Gox cited this class for accounting confusion in 2014; attribution
   to its losses is contested — Decker & Wattenhofer arxiv 1403.6676 — cite
   as mechanism demonstration only.) BIP-141 SegWit fixed it structurally
   by moving signatures outside the txid preimage.
2. Ethereum EIP-2 (homestead): restricted ECDSA `s` to the lower half of
   the curve order at protocol level. Every pre-EIP-2 contract deriving an
   ID from `keccak(r, s, v)` had this bug.
3. Cosmos SDK #9723: secp256r1 handling where the derived identifier was
   not recomputed under the enforced canonical form. `hash(TxRaw) ≠
   hash(SignDoc)` — the verifier MUST reconstruct `SignDoc` and recompute
   anything derived from it.

**Check**:
1. For every signed object type, enumerate `signature`/`sig`/`signatures[]`,
   the derived `id`/`hash`/`tx_id`/`block_hash`/`commitment_id`, and the
   canonicalized `payload`/`body`/`sign_doc` the signer actually signed.
2. Locate the verifier and verify EACH:
   - Signature verify: `verify(pubkey, payload_bytes, signature)` returns true
   - ID recompute: `recomputed_id = hash(payload_bytes)` (protocol canonical form)
   - **ID binding**: `assert recomputed_id == provided_id` — the critical line. Missing → producer sets arbitrary `id` while signature verifies.
   - Malleability normalization: if the scheme admits multiple valid encodings (low-s, DER vs compact, trailing zeros), the normalized form must be used in `recomputed_id`.
3. Multi-sig/threshold: the preimage MUST include the aggregated signature
   OR an order-canonicalized list — order-sensitive hashing of signatures
   is itself a malleability vector.

**Tag**: `[SIG-ID-NOT-BOUND:{struct_name}.{id_field}]` — CRITICAL-default
when the ID is used as a cache key, dedup key, dependency reference, or any
cross-system identifier.

## CHECK 10 — Aggregate signature ↔ Merkle leaf linkage [L]

**Catches**: a signed aggregate (block, batch, commitment bundle) carries a
Merkle root R and a signature σ over R. Downstream consumers receive (σ, R,
leaves[], proofs[]) and MUST verify σ authenticates R AND each leaf is
proven under R. Bug class: verifier checks σ binds R but (a) never re-roots
the leaves, (b) accepts an alternate leaf with valid proof under R when R
commits to something else (second-preimage / alternate-encoding), or (c)
checks the proof but not leaf-index monotonicity, allowing leaf replacement
at the same index.

**Check**:
1. For each message shape `{signature, root, leaves, proofs}` (block+txs,
   checkpoint+attestations, commitment+chunks, batch+items), locate the
   verifier.
2. Verify ALL THREE: σ verifies against R with the correct signer key set;
   each leaf `L[i]` passes `merkle_verify(R, L[i], proof[i], index[i])`;
   `hash(L[i])` uses the exact canonicalization the producer used (flag
   variable-length integers, endianness, padding, domain-separator bytes
   missing from either side).
3. Second-preimage guards: leaf hashes MUST be domain-separated from
   internal-node hashes (`0x00` leaves, `0x01` nodes); tree depth fixed or
   length-prefixed (variable depth without length binding → same R, multiple
   valid leaf sets).
4. Index binding: if leaf index has semantic meaning ("tx at index 0 is
   coinbase"), the index must be bound into the leaf's hash input.
5. BLS consensus aggregates: the signed message must bind slot/epoch/view
   AND the root — a signature over bare R without the view is replayable
   across any view that ever produced the same R.

**Tag**: `[SIG-MERKLE-LINK:{field}:{missing-check}]`. Severity High by
default; Critical when the corrupted set influences consensus weight, reward
distribution, or inclusion proofs consumed by other chains.

## Processing protocol (mandatory for every CHECK)

1. **ENUMERATE targets**: list every entity the CHECK applies to as a
   numbered list before analysis begins.
2. **PROCESS exhaustively**: analyze each numbered entity; mark each "DONE"
   or "N/A (reason)" before moving on.
3. **COVERAGE GATE**: count enumerated vs processed. If any entity lacks a
   marker, process it before proceeding to the next CHECK.

## Workflow

```
Task Progress:
- [ ] Step 1: Find all signature verification code (ecrecover, ECDSA, EIP-712, chain-native verify)
- [ ] Step 2: Check same-chain replay protection (nonce management)
- [ ] Step 3: Check cross-chain replay protection (chainId / program ID in domain/message)
- [ ] Step 4: Check cross-contract replay protection (verifyingContract / module address)
- [ ] Step 5: Check deadline/expiry enforcement and time-source units
- [ ] Step 6: Verify ecrecover safety (address(0), s-value, v-value)
- [ ] Step 7: Verify EIP-712 domain separator completeness
- [ ] Step 8: Check off-chain approvals (permit, Permit2, signed orders) for front-running
- [ ] Step 9: CHECK 9/10 for any signed object with derived ID or Merkle root
- [ ] Step 10: ERC-1271 support for contract wallets (if applicable)
- [ ] Step 11: Score findings and generate report
```

## Output format

```markdown
### Finding: [Title]

**ID:** [SIG-N]
**Function:** `functionName()` at `Contract.sol:L42`
**Replay Type:** [Same-Chain | Cross-Chain | Cross-Contract | Nonce-Skip | Expired | Malleability | ID-Binding | Merkle-Linkage]
**Severity:** [CRITICAL | HIGH | MEDIUM]
**Lens:** [Q] / [L]

**Issue:**
[Description of the replay vulnerability or verification flaw]

**Signed Message Fields:**
- [x] to/from addresses
- [x] amount/value
- [ ] chainId ← MISSING
- [ ] verifyingContract ← MISSING
- [x] nonce
- [ ] deadline ← MISSING

**Attack Scenario:**
1. User signs message for [intended purpose]
2. Attacker captures signature from [source]
3. Attacker replays on [target chain/contract/time]
4. [Unauthorized action occurs]

**Recommendation:**
[Add EIP-712 domain separator, add nonce, add deadline, use ECDSA.recover, bind ID]
```

**Quality gate**: every finding MUST cite the specific verification code
(file:line) AND the missing/broken protection. Do NOT flag patterns that
framework-provided safe wrappers already handle (OpenZeppelin ECDSA.recover,
Anchor's ed25519 instruction parsing) — verify whether the protocol uses the
raw primitive or a safe wrapper.

## Quick detection checklist

- [ ] Does every signature include a nonce? (Same-chain replay)
- [ ] Does the signed message include `chainId`? (Cross-chain replay)
- [ ] Does the signed message include `address(this)`? (Cross-contract replay)
- [ ] Is there a deadline with `block.timestamp` check? (Late execution)
- [ ] Is `ecrecover` result checked against `address(0)`?
- [ ] Is the s-value enforced lower-half? (Malleability)
- [ ] Is the domain separator recalculated on chain fork? (Fork replay)
- [ ] Is OpenZeppelin's ECDSA used instead of raw `ecrecover`?
- [ ] For permit: is the nonce incremented before state changes?
- [ ] For contract wallets: is ERC-1271 `isValidSignature` supported?
- [ ] For signed objects with IDs: is the derived ID recomputed and asserted?
- [ ] For aggregates with Merkle roots: are leaves re-rooted, canonicalized, and index-bound?

## Rationalizations to reject

- "We use nonces so replay is impossible" → Check cross-chain and cross-contract replay (nonce doesn't prevent those)
- "No one would replay on another chain" → Attackers monitor all chains; automated bots scan for replayable signatures
- "ecrecover is a built-in, so it's safe" → It returns address(0) on failure, not revert; it doesn't enforce s-value
- "The signature includes all the parameters" → Without chainId and contract address, it's still replayable
- "We hardcoded chainId = 1" → Chain forks create two live chains with the same chainId; use block.chainid
- "Permit is a standard, so it's safe" → The standard defines the interface, not the implementation

## References

- [references/replay-taxonomy.md](references/replay-taxonomy.md) — the five types with vulnerable/safe code
- [references/eip712-checklist.md](references/eip712-checklist.md) — domain separator verification
