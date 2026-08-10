---
name: omega-audit-workflow
description: Run a full smart-contract audit engagement the way Team Omega does — scope as a commit diff, two independent passes merged into one report, compile/deploy/test the code rather than only reading it, triage static-analysis output instead of pasting it, organize findings per-file with ID prefixes and a General section, and close the loop with a preliminary report, a client fix commit, and a verified per-issue Resolution. Use when starting an audit, deciding how to structure a review, writing up findings, re-auditing a client you have audited before, or reviewing fixes.
---

# Omega Audit Workflow

The process layer. Distilled from the structure common to all 55 Team Omega
reports (2021-03 → 2026-07). Use this to run the engagement; use the other
seven Omega skills as the lenses you apply during Phase 3.

## The shape of an Omega engagement

```
1. Fix scope        → exact repo, exact commit(s), exact file list
2. Build & run      → compile, deploy, test; run static analysis; triage
3. Review           → two independent passes, per-file, applying the lenses
4. Preliminary      → deliver findings before any fixes exist
5. Client fixes     → client returns a commit hash
6. Verify & close   → re-audit the fix commit, write per-issue Resolution
```

Steps 4–6 are not optional polish. In the archive they routinely surface new
bugs: dxDAO `VM13` ("stakers who lost can redeem their stake") was *introduced
by the fix* for `VM1` and only caught because Omega re-audited the fix commit.
Treat the fix commit as a fresh, in-scope codebase.

---

## Phase 1 — Fix the scope, in writing

Never write "we audited the protocol." Write what Omega writes:

- **Repository URL**, and the branch if not default.
- **Commit hash** the review is based on. Full 40 chars.
- For a re-audit or upgrade: **both** hashes, and state that scope is the diff.
  `202501-Altitude-parallel-farming` scopes to a single PR: "the diff between
  commit `5b3026b…` and `4ce09aa…`".
- **File list**, explicitly. `202508-Giza-Pendle` names directories and then
  says "about 6000 normalised lines of Python code (excluding blank lines and
  comments)" — normalized LOC, not file count, is the honest size signal.
- **Anything the client gave you that isn't code**: spec documents, architecture
  notes. Omega cites the Google Doc URL in the report.

If the client is a repeat engagement, also record the **audit history** — every
prior report and its date. You will need it in Phase 3.

> **Multi-repo scope is normal.** Giza engagements span an agent repo and a
> contract-interactions repo; Omega audits both and cross-references findings
> between them (`202508-Giza-Pendle` BE1 notes the same bug in `stargate.py`
> in the other repository).

## Phase 2 — Build it and run it

Omega's Methods Used section says the same thing in every report since 2021:

> "The contracts were compiled, deployed, and tested in a test environment."

This is a *precondition*, not a formality. Do it before reading closely.

1. **Compile.** Compiler warnings are findings — Omega files them (`Delphia G3`,
   `PrimeDAO-Seed G4`, `EnterDAO Y5`). A repo that does not compile in its
   documented configuration is itself a finding (`202510-Backed-Token-Bridge
   G3`).
2. **Run the test suite.** Failing tests are findings (`dxGovernance G3`,
   `dxDAO-governance G3`, `Gnosis-Hashi G2`). So is a suite that passes but
   whose *coverage command* is broken (`DXdao-staking G1`).
3. **Measure coverage.** "Test coverage is incomplete" appears in roughly two
   thirds of the archive. Name the uncovered paths that matter, not the
   percentage.
4. **Run static analysis** — Slither, and historically MythX/Remix. Then
   **triage it**. Omega's standard sentence is that the tools found no true
   high-severity issues and mostly flagged pragma and visibility issues, which
   are then folded into the appropriate per-file sections. Never paste tool
   output into a report. Recent reports (`202605-Backed-Token-ERC4626`) add
   "and AI tools" to the same triage discipline.
5. **`npm audit` / dependency audit.** Omega files advisories against Solidity
   dependencies as findings, at *medium* when the advisory is critical
   (`Blindex G2`, citing a TimelockController advisory in OpenZeppelin).

Write a PoC when the mechanism is non-obvious. `Fragmint A3` includes a full
attacker contract — a `receive()` that calls `assert(false)`, plus a
`deleteContract()` that self-destructs on payment of a ransom — to show the
finding is extortion, not mere griefing.

## Phase 3 — Two independent passes

> "The audit report has been compiled on the basis of the findings from
> different auditors. The auditors work independently."

The independence is the point: two reviewers who have not discussed the code
find different things. Merge afterwards. If you are one agent, simulate this
honestly — two passes with different entry points (one bottom-up from the data
structures, one top-down from the external entry points), each written down
before you read the other.

**Order the review by file, not by bug class.** Walk the files. For each,
apply the lenses:

| Lens | Load |
|---|---|
| Can assets get back out? | `omega-asset-exit-paths` |
| Does this check actually bind? | `omega-enforceability-check` |
| Is this counter right on every path? | `omega-accounting-consistency` |
| What are we trusting, for what? | `omega-external-data-trust` |
| Who profits from reordering? | `omega-ordering-and-approval-races` |
| What did the diff break? | `omega-upgrade-diff-review` |
| Is the repo itself sound? | `omega-repo-hygiene-sweep` |

**Read the context, not just the code.** `202410-Backed-token-bridge` records
that Omega "also inspected the context in which the contracts were developed,
namely the Chainlink Bridge architecture." When a contract integrates Pendle,
Morpho, Aave, CCIP or Stargate, read that protocol's documentation and check
the integration's assumptions against it. `202505-Altitude SPP1` is exactly
this: Pendle's own docs say SY tokens are not always 1:1 wrappers, so
`getPtToSyRate` is the wrong oracle call — a finding that is invisible from
the Solidity alone.

**For repeat clients, re-check the old reports.** `202508-Giza-Pendle G1`
("Unaddressed issues from older reports") is filed at medium and lists prior
findings still open, including one that had been resolved and *re-appeared*.
`202505-Altitude` has a whole "Fixes from older reports" section. Make this a
standing agenda item.

## Phase 4 — Severity

Omega's four levels, verbatim from their reports:

| | |
|---|---|
| **High** | Vulnerabilities that can lead to loss of assets or data manipulations. |
| **Medium** | Vulnerabilities that are essential to fix, but that do not lead to assets loss or data manipulations. |
| **Low** | Issues that do not represent direct exploit, such as poor implementations, deviations from best practice, high gas costs, etc. |
| **Info** | Matters of opinion. |

Three habits that matter more than the ladder itself:

1. **Justify the rating in one clause, inline.** Not "Severity: Medium" but
   `Fragmint A6`: "Medium — although loss of funds is possible, the scenario of
   having so many shareholders seems quite improbable." `dxGovernance V2`:
   "High. The `orgBoostedProposalsCnt` regulates the amount of stakes needed to
   reach the boosted state … conceivably halting the operation of the DAO."
   Likelihood and impact, stated, every time.
2. **Off-chain consequences count.** `PrimeDAO-Seed S4` is rated medium
   explicitly because "the attack may have off-chain financial consequences" —
   contractual obligations to an investor — even though no user loses funds
   on-chain.
3. **Report privileged-actor risk even when intended.** "Owner can steal",
   "blacklister can disable mint and burn", "oracle owner can manipulate price
   results" are all filed, at whatever severity fits. Do not suppress a finding
   because the client will say it is by design; let them say it, in the
   Resolution.

## Phase 5 — Report structure

```
Summary                  ← who the client is, what the system does
Scope of the audit       ← repos, commits, files, normalized LOC
Methodology              ← independent auditors; what you actually ran
Liability / Disclaimer
Severity definitions
Summary of findings      ← prose + severity × (found / resolved) table
Resolution               ← the fix commit; how you verified
Findings
  General                ← G1…Gn: repo-wide and cross-cutting
  path/to/File.sol       ← per-file section
    XX1. Title [sev] [status]
    XX2. …
```

**Finding IDs are derived from the file name.** `ERC20StakingRewardsDistribution.sol`
→ `D1…D14`. `api/adapters/business_adapter.py` → `ABA1…ABA3`.
`business/impl/db/sql_reader.py` → `SQLR1, SQLR2`. This makes an ID
self-locating in conversation with the client, and it survives re-numbering
when findings are added.

**Every finding has the same three parts**, in this order:

- **Description** — the mechanism, then the consequence. Concrete: name the
  function, quote the line, give a worked numeric scenario if the bug is
  arithmetic (`PrimeDAO-Seed S17` walks three classes through a sale to show
  the shortfall; `Spectre B2` prices out the front-run in dollars).
- **Recommendation** — what to do. Give the alternative when the obvious fix is
  bad: `Karpatkey K2` proposes a delay, explains why the delay is itself
  harmful, then points at a better structural fix in `K4`.
- **Severity** — with its one-clause justification.

**Cross-reference relentlessly.** `Giza-Pendle O2` opens with "Issue O2 is a
direct result from this issue" pointing back to `O1`; `Spectre I4` says
"Issues B5, B6, B7, B8 apply for the Broker contract as well" rather than
duplicating four findings. Root causes get one finding; consequences point at
it.

**The Summary of findings is prose plus a table.** The prose says what you
looked for and what you deliberately excluded — `202508-Giza-Pendle` states
that because the review focused on loss-of-funds, "we only include some very
generic issues of the category 'info'." Scope decisions belong in the report.

## Phase 6 — Resolution

Deliver preliminary → client returns a commit → **re-audit that commit** →
append a `Resolution:` line to every finding. Statuses used in the archive:

| Status | Meaning |
|---|---|
| `[resolved]` | Fixed in code; you verified it |
| `[partially resolved]` | Fixed for the reported case, not the general one |
| `[not resolved]` | Unchanged |
| `[acknowledged]` | Client accepts the risk; record their reasoning |
| `[will be resolved]` | Committed to, not yet done |
| `[resolved*]` | Not fixed in code, but no longer reachable given a scope or deployment decision — always footnote what that decision was |

That last one is worth adopting. `202508-Giza-Pendle` marks four cross-chain
findings `[resolved*]` because the team confirmed the first release runs on a
single chain — and says so explicitly. It records the mitigation as
*contingent*, which is honest in a way that "resolved" is not.

Quote the client verbatim when they disagree. `Giza-Pendle ABA3` reproduces a
five-line rebuttal from the Giza team explaining why the wallet must run in
non-`ACTIVATED` states. Their reasoning belongs in the record next to yours.

Also: **audit the rest of the fix commit**, not only the fixes.
`202605-Backed-Token-ERC4626` states "We also audited the other changes that
were made in this commit."

---

## Checklist

Scope
- [ ] Repo, branch, full commit hash(es), explicit file list, normalized LOC
- [ ] Prior reports for this client listed and their open findings re-checked
- [ ] Spec/design docs from the client obtained and cited

Build
- [ ] Compiles; warnings captured as findings
- [ ] Test suite runs; failures captured as findings
- [ ] Coverage measured; uncovered critical paths named
- [ ] Static analysis run and triaged (not pasted)
- [ ] Dependency advisories checked

Review
- [ ] Two independent passes, merged
- [ ] Every file walked; findings ID'd per-file with a General section
- [ ] Integrated third-party protocol docs read and assumptions checked
- [ ] All seven Omega lenses applied

Write-up
- [ ] Every finding: mechanism → consequence → Recommendation → Severity+why
- [ ] Root causes cross-referenced, consequences not duplicated
- [ ] Severity table with found/resolved counts

Close
- [ ] Preliminary delivered before fixes existed
- [ ] Fix commit re-audited, including unrelated changes in it
- [ ] Per-issue Resolution written; contingent mitigations marked and footnoted
- [ ] Client disagreements quoted, not paraphrased
