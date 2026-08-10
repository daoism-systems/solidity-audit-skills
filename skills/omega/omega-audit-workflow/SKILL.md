---
name: omega-audit-workflow
description: Run a full smart-contract audit engagement — scope as a commit diff, two independent passes merged into one report, compile/deploy/test the code rather than only reading it, triage static-analysis output instead of pasting it, organize findings per-file with ID prefixes and a General section, and close the loop with a preliminary report, a client fix commit, and a verified per-issue Resolution. Use when starting an audit, deciding how to structure a review, writing up findings, re-auditing a codebase you have reviewed before, or reviewing fixes.
---

# Audit Workflow

The process layer. Use this to run the engagement; use the other seven skills
in this set as the lenses applied during Phase 3.

```
1. Fix scope        → exact repo, exact commit(s), exact file list
2. Build & run      → compile, deploy, test; run static analysis; triage
3. Review           → two independent passes, per-file, applying the lenses
4. Preliminary      → deliver findings before any fixes exist
5. Client fixes     → client returns a commit hash
6. Verify & close   → re-audit the fix commit, write per-issue Resolution
```

Steps 4–6 are not optional polish. Fix commits routinely contain new bugs: a
patch written against a narrow symptom description, in code the author has
re-entered after a gap, arriving when reviewer attention is lowest. Treat the
fix commit as a fresh, in-scope codebase.

---

## Phase 1 — Fix the scope, in writing

Never write "we audited the protocol." Write:

- **Repository URL**, and the branch if not the default
- **Commit hash** the review is based on — full 40 characters
- For a re-audit or upgrade: **both** hashes, and an explicit statement that
  scope is the diff
- **File list**, explicitly. Prefer **normalized lines of code** (excluding
  blanks and comments) over file count as the size signal — it is the honest
  measure of what was reviewed
- **Anything non-code the client supplied** — specifications, architecture
  notes, design docs. Cite them by URL

For a repeat engagement, also record the **audit history**: every prior review
and its date. You will need it in Phase 3.

> **Multi-repo scope is normal** for systems with off-chain components. Audit
> the contracts and the backend that drives them together, and cross-reference
> findings between repositories — the same defect often appears in both, and a
> finding that is low severity in one may be critical in the other.

## Phase 2 — Build it and run it

Do this before reading closely. It is a precondition, not a formality.

1. **Compile.** Warnings are findings. A repo that does not compile in its
   documented configuration is itself a finding.
2. **Run the test suite.** Failing tests are findings. So is a suite that passes
   while its coverage command is broken.
3. **Measure coverage.** Name the uncovered paths that matter; do not quote a
   percentage.
4. **Deploy to a local test environment.** This is what separates reading from
   auditing — it is how you check a claimed behaviour rather than inferring it,
   and how you write a proof of concept later.
5. **Run static analysis**, then **triage it**. Tool output belongs in the
   report only after a human has decided each item is real, is in scope, and is
   correctly rated. Fold survivors into the appropriate per-file sections. Never
   paste raw output — it is the fastest way to lose a client's trust in the
   whole document. The same discipline applies to AI-assisted analysis.
6. **Run the dependency advisory audit.**

**Write a proof of concept when the mechanism is non-obvious.** A working
attacker contract converts an argument into a demonstration, and it forecloses
the "that isn't exploitable in practice" response. It also lets you characterise
the finding accurately — a griefing vector that can be held for ransom is an
extortion finding, and you only learn that by building it.

**Verify external behaviour empirically.** Where the system trusts a third-party
API or protocol, call it directly with adversarial parameters and record the
response. Observed behaviour beats documented behaviour.

## Phase 3 — Two independent passes

> Two reviewers who have not discussed the code find different things.

Review independently, then merge. If you are one agent, simulate this honestly:
two passes with different entry points — one bottom-up from the data structures
and state variables, one top-down from the external entry points — each written
down *before* reading the other. The value is in the independence, so do not
let the second pass read the first's notes.

**Order the review by file, not by bug class.** Walk the files; apply the lenses
to each:

| Lens | Load |
|---|---|
| Can assets get back out? | `omega-asset-exit-paths` |
| Does this check actually bind? | `omega-enforceability-check` |
| Is this counter right on every path? | `omega-accounting-consistency` |
| What are we trusting, for what? | `omega-external-data-trust` |
| Who profits from reordering? | `omega-ordering-and-approval-races` |
| What did the diff break? | `omega-upgrade-diff-review` |
| Is the repo itself sound? | `omega-repo-hygiene-sweep` |

**Read the context, not just the code.** For every integrated protocol —
lending market, AMM, bridge, yield wrapper — read its integration documentation
and check the code against the caveats it states. Rate functions correct for one
class of underlying and wrong for another, TWAP windows shorter than
recommended, and market configurations assumed rather than verified are all
invisible from the Solidity alone. This is the highest-yield activity that pure
code review cannot produce.

**For repeat engagements, re-check the prior reports.** Two checks: findings
still open, and findings previously resolved that have since **regressed**. Both
belong in the report, and the second is the one only a carried-forward review
can catch.

## Phase 4 — Severity

Four levels:

| | |
|---|---|
| **High** | Vulnerabilities that can lead to loss of assets or data manipulations |
| **Medium** | Vulnerabilities that are essential to fix, but that do not lead to asset loss or data manipulation |
| **Low** | Issues that do not represent a direct exploit — poor implementations, deviations from best practice, high gas costs |
| **Info** | Matters of opinion |

Three habits matter more than the ladder:

1. **Justify the rating in one clause, inline.** Not "Severity: Medium" but
   "Medium — loss of funds is possible, but the precondition is improbable," or
   "High — the counter gates the boost threshold, conceivably halting governance
   entirely." Likelihood and impact, stated, every time. Where the two pull in
   opposite directions, say which dominates.
2. **Off-chain consequences count.** An attack that costs no on-chain funds but
   breaks a contractual obligation the operator has to a counterparty is a real
   finding. Rate it on consequence, not on whether value moved.
3. **Report privileged-actor risk even when intended.** "The owner can withdraw
   user deposits," "the blacklister can disable transfers," "the oracle updater
   can set any price" — file them at whatever severity fits. Do not suppress a
   finding in anticipation of "that is by design." Let the client say so, in the
   Resolution, on the record.

## Phase 5 — Report structure

```
Summary                  ← who the client is, what the system does
Scope of the audit       ← repos, commits, files, normalized LOC
Methodology              ← independent reviewers; what you actually ran
Liability / Disclaimer
Severity definitions
Summary of findings      ← prose + severity × (found / resolved) table
Resolution               ← the fix commit; how you verified
Findings
  General                ← G1…Gn: repo-wide and cross-cutting
  path/to/File.sol       ← per-file section
    XX1. Title [severity] [status]
    XX2. …
```

**Derive finding IDs from the file name.** `VaultCore.sol` → `VC1, VC2…`;
`db/sql_reader.py` → `SQLR1, SQLR2…`. An ID is then self-locating in a client
conversation, and stays stable when findings are added or renumbered.

**Every finding has three parts, in order:**

- **Description** — the mechanism, then the consequence. Name the function,
  quote the line. For arithmetic findings, walk a concrete numeric scenario the
  reader can check with a calculator; a worked example survives disagreement
  about the model in a way that a proof sketch does not.
- **Recommendation** — what to do. When the obvious fix has a serious drawback,
  say so and propose the structural alternative. When there is no clean fix, say
  *that*, and give the least-bad option rather than dressing a partial
  mitigation as a resolution.
- **Severity** — with its one-clause justification.

**Cross-reference relentlessly.** Root causes get one finding; consequences
point at it. Where findings in one contract apply verbatim to a near-identical
sibling, say so by reference instead of duplicating the writeup.

**The Summary of findings is prose plus a table.** The prose states what you
looked for and what you deliberately excluded. Scope decisions — "this review
focused on loss-of-funds, so informational issues are only sampled" — belong in
the report, not in your head.

## Phase 6 — Resolution

Deliver preliminary → client returns a commit → **re-audit that commit** →
append a `Resolution:` line to every finding.

| Status | Meaning |
|---|---|
| `[resolved]` | Fixed in code; you verified it |
| `[partially resolved]` | Fixed for the reported case, not the general one |
| `[not resolved]` | Unchanged |
| `[acknowledged]` | Client accepts the risk; record their reasoning |
| `[will be resolved]` | Committed to, not yet done |
| `[resolved*]` | Not fixed in code, but unreachable given a scope or deployment decision — always footnote what that decision was |

The last one is worth adopting deliberately. When a finding is neutralised by a
plan rather than by code — "only one chain at launch," "this will be a fresh
deployment, not an upgrade" — mark it as contingent and record the contingency.
Plans change; a flat "resolved" hides that the safety depends on one.

**Quote the client verbatim when they disagree.** Their reasoning belongs in the
record next to yours, unparaphrased. A reader in two years needs to see both.

**Audit the rest of the fix commit**, not only the fixes — clients bundle
unrelated work into them.

---

## Checklist

Scope
- [ ] Repo, branch, full commit hash(es), explicit file list, normalized LOC
- [ ] Prior reviews enumerated; their open findings re-checked
- [ ] Spec and design docs obtained and cited

Build
- [ ] Compiles; warnings captured as findings
- [ ] Test suite runs; failures captured as findings
- [ ] Coverage measured; uncovered critical paths named
- [ ] Deployed to a local test environment
- [ ] Static analysis run and **triaged**, not pasted
- [ ] Dependency advisories checked
- [ ] PoC written for every non-obvious mechanism

Review
- [ ] Two independent passes with different entry points, merged
- [ ] Every file walked; findings ID'd per-file, plus a General section
- [ ] Integrated protocols' documentation read and assumptions checked
- [ ] All seven lenses applied
- [ ] Previously-resolved findings checked for regression

Write-up
- [ ] Every finding: mechanism → consequence → Recommendation → Severity + why
- [ ] Numeric scenarios worked through for arithmetic findings
- [ ] Root causes cross-referenced; consequences not duplicated
- [ ] Severity table with found/resolved counts
- [ ] Deliberate exclusions stated in the summary

Close
- [ ] Preliminary delivered before fixes existed
- [ ] Fix commit re-audited in full, including unrelated changes
- [ ] Per-issue Resolution written; contingent mitigations marked and footnoted
- [ ] Client disagreements quoted, not paraphrased
