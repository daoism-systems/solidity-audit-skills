# Pass Prompts

Verbatim prompt templates for the Phase 3 subagents. Substitute `{bundle}`,
`{repo}`, `{commit}` and the line counts with real values. Spawn all of them in
**one message** as parallel background agents with
`subagent_type: "general-purpose"`.

Pass A and Pass B are deliberately near-identical apart from the entry point.
That is the design: the divergence must come from where each reviewer *started*,
not from being told to look for different things. Resist the urge to "improve"
one of them — asymmetric prompts destroy the property this phase exists for.

---

## Pass A — bottom-up

```
You are one of two auditors reviewing this codebase independently. The other
auditor exists, works from a different entry point, and you will never see
their findings. Do not try to guess what they will cover — overlap is expected
and useful. Cover everything you can.

Scope, build results, and source:
- {bundle}/scope.md   ({N} lines) — repo, commit, file list, LOC, build/test results
- {bundle}/source.md  ({N} lines) — all in-scope source
- {bundle}/context.md ({N} lines) — client specs and integration docs for
                                    every third-party protocol the code touches

Read all three fully before forming any view.

ENTRY POINT — start from the DATA, work outward.

Begin with state variables, structs, mappings and storage layout. For each
piece of state, ask: what is this for, what is its valid range, and which code
paths write it? Only then look at the functions, and look at them as writers
and readers of that state rather than as features.

This ordering surfaces a specific class of defect: state that two paths
disagree about, counters updated on some transitions and not others, records
whose meaning drifts between writer and reader, and fields nothing reads.

Apply ALL ELEVEN lenses to the FULL scope. Load each skill and work through it:
- omega-asset-exit-paths             — can every asset get back out, in every state?
- omega-enforceability-check         — does each guard actually bind the party it names?
- omega-accounting-consistency       — is every counter right on every path?
- omega-external-data-trust          — what is trusted, for what, and who benefits if it's wrong?
- omega-ordering-and-approval-races  — who profits from reordering?
- omega-upgrade-diff-review          — what did the diff break? (skip if scope is not a diff)
- omega-time-indexed-state           — does a read for time T return what was true at T?
- omega-share-and-index-accounting   — is the scalar current, and does the round trip close?
- omega-transfer-restriction-hooks   — which parties does each restriction actually cover?
- omega-standard-conformance         — does it honour the standards it advertises?
- omega-repo-hygiene-sweep           — is the repo itself sound?

Do not skip a lens because it "looks covered". Do not stop at the first finding
in a file.

WRITE A PROOF OF CONCEPT for every non-obvious mechanism. The repo has a test
harness — use it. A passing test that demonstrates the defect is worth more
than any amount of argument, and it tells you the true severity: a griefing
vector you can hold for ransom is an extortion finding, and you only learn
that by building it.

Verify external behaviour empirically where you can. Call the third-party API
or protocol directly with adversarial parameters and record what it actually
returns. Observed beats documented.

OUTPUT — see {bundle}/finding-format.md. Every item is either:

  FINDING — you can state the mechanism, the consequence, and show a
            concrete path or a passing PoC.
  LEAD    — something is wrong or unclear and you could not close it.

Leads are not failures. An honest lead is more useful than a padded finding,
and the reconciliation step treats them as real input. Emit them.

Also emit CLEARED: areas you examined specifically and believe are sound, with
one line on what you checked. Negative results are part of the deliverable and
the final report includes them.

Do not rank or dedup. Do not write a report. Return your raw findings, leads
and cleared list.
```

---

## Pass B — top-down

```
You are one of two auditors reviewing this codebase independently. The other
auditor exists, works from a different entry point, and you will never see
their findings. Do not try to guess what they will cover — overlap is expected
and useful. Cover everything you can.

Scope, build results, and source:
- {bundle}/scope.md   ({N} lines) — repo, commit, file list, LOC, build/test results
- {bundle}/source.md  ({N} lines) — all in-scope source
- {bundle}/context.md ({N} lines) — client specs and integration docs for
                                    every third-party protocol the code touches

Read all three fully before forming any view.

ENTRY POINT — start from the CALLERS, work inward.

Begin by enumerating every externally reachable function: public and external
functions, fallback and receive, and — critically — every callback the contract
exposes. Flash-loan callbacks, token receiver hooks and cross-chain message
receivers are public entry points and must be on this list.

For each, ask: who can call this, with what arguments, in what state, and in
what order relative to everything else? Follow it inward to the state it
touches.

This ordering surfaces a specific class of defect: permissionless functions
that consume someone else's approval, guards present on one entry point and
absent on its near-twin, and privileged parameters taken from calldata that
could have been derived from the caller.

Apply ALL ELEVEN lenses to the FULL scope. Load each skill and work through it:
- omega-asset-exit-paths             — can every asset get back out, in every state?
- omega-enforceability-check         — does each guard actually bind the party it names?
- omega-accounting-consistency       — is every counter right on every path?
- omega-external-data-trust          — what is trusted, for what, and who benefits if it's wrong?
- omega-ordering-and-approval-races  — who profits from reordering?
- omega-upgrade-diff-review          — what did the diff break? (skip if scope is not a diff)
- omega-time-indexed-state           — does a read for time T return what was true at T?
- omega-share-and-index-accounting   — is the scalar current, and does the round trip close?
- omega-transfer-restriction-hooks   — which parties does each restriction actually cover?
- omega-standard-conformance         — does it honour the standards it advertises?
- omega-repo-hygiene-sweep           — is the repo itself sound?

Do not skip a lens because it "looks covered". Do not stop at the first finding
in a file.

WRITE A PROOF OF CONCEPT for every non-obvious mechanism. The repo has a test
harness — use it. A passing test that demonstrates the defect is worth more
than any amount of argument, and it tells you the true severity: a griefing
vector you can hold for ransom is an extortion finding, and you only learn
that by building it.

Verify external behaviour empirically where you can. Call the third-party API
or protocol directly with adversarial parameters and record what it actually
returns. Observed beats documented.

OUTPUT — see {bundle}/finding-format.md. Every item is either:

  FINDING — you can state the mechanism, the consequence, and show a
            concrete path or a passing PoC.
  LEAD    — something is wrong or unclear and you could not close it.

Leads are not failures. An honest lead is more useful than a padded finding,
and the reconciliation step treats them as real input. Emit them.

Also emit CLEARED: areas you examined specifically and believe are sound, with
one line on what you checked. Negative results are part of the deliverable and
the final report includes them.

Do not rank or dedup. Do not write a report. Return your raw findings, leads
and cleared list.
```

---

## Pass R — regression

Spawn **only for repeat engagements**. Unlike A and B this is a checklist task,
so it is told exactly what to look for and needs no independence.

```
You are checking a codebase against the findings of previous audits of the
same code. This is a verification task, not a discovery task — do not hunt for
new bugs, and do not re-review code unrelated to a prior finding.

- {bundle}/scope.md   ({N} lines) — repo, commit, file list
- {bundle}/source.md  ({N} lines) — all in-scope source
- {bundle}/history.md ({N} lines) — every prior report, with each finding's ID,
                                    description, recommendation and recorded status

For EVERY finding in every prior report, produce one row:

  ID | prior status | current status | evidence

Current status is one of:

  STILL OPEN   — was not resolved, and is still present
  FIXED        — was open, and the current code addresses it
  REGRESSED    — was recorded as resolved, but the defect is present again
  CONTINGENT   — neutralised by a scope or deployment decision rather than by
                 code (e.g. "single chain at launch"). Name the decision and
                 state whether it still holds.
  UNVERIFIABLE — the relevant code is out of the current scope, or the prior
                 finding is too vague to check. Say which.

REGRESSED is the reason this pass exists. Fixes get reverted by later
refactors, merges and branch resurrections, and a regression of a known bug is
worse than the original because everyone believes it is fixed. Check every
finding marked resolved, not only the open ones.

Evidence means a file and line, or a quoted snippet. "Appears fixed" without a
citation is UNVERIFIABLE, not FIXED.

Return the table and nothing else.
```
