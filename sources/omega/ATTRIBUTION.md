# Attribution — Omega Skills

## What was derived, and from what

The skill files in this directory are **original prose written by Daoism
Systems**. They were derived by studying the public audit archive at
[github.com/OmegaAudits/audits](https://github.com/OmegaAudits/audits)
(repository state at commit `a9891e4`, 14 July 2026) — 55 reports published
between March 2021 and July 2026 — and extracting the recurring methodology and
reasoning patterns.

**No Team Omega report text is reproduced here, and no skill references a
specific report, client or finding.** The skills are written at the level of
abstraction of pattern recognition: recognizable code shapes, structural
questions, detection heuristics and fix patterns. What carries over is
methodological — which questions are worth asking of every contract, which
failure shapes recur across unrelated codebases, and how to structure scope,
severity and the resolution loop.

That choice is deliberate. A skill that teaches an agent to recognize *the
shape* generalizes to code it has never seen; a skill that catalogues past
findings teaches it to look backwards.

## Corpus studied

| | |
|---|---|
| Reports | 55 (2021-03 → 2026-07) |
| Findings read | 834 |
| By severity | 2 critical · 46 high · 105 medium · 332 low · 349 info |
| Report format | PDF; text extracted with `pypdf` for analysis only |

The codebases represented span tokenized RWA and stablecoins, lending and yield
vaults, bridges and cross-chain messaging, DAO governance, NFT
fractionalization, and — notably — Python agent backends and frontends audited
alongside the contracts. That last category is why
[`omega-external-data-trust`](omega-external-data-trust/SKILL.md) covers
off-chain service surface, which most Solidity-only skill sets omit. The heavy
weighting toward permissioned RWA tokens is likewise why
[`omega-transfer-restriction-hooks`](omega-transfer-restriction-hooks/SKILL.md)
exists as its own lens.

The corpus was analysed twice. The first pass read every critical, high and
medium finding in full and the low/info findings by title. The second pass read
all 787 extractable finding bodies and clustered them, which surfaced two
families the first pass had missed — compliance/transfer-restriction gating and
standard-conformance — now covered by the two skills named above. Clusters that
were considered and deliberately **not** given their own skill: reentrancy and
CEI ordering (covered by **[Q]** `reentrancy-pattern-analysis`), decimal/scale
mismatch (covered by **[P]** `math-precision-agent` and **[L]**
`dimensional-analysis`), and mechanism/incentive design (covered by **[P]**
`economic-security-agent`, though it is the strongest remaining candidate).

A third pass re-read the DAO and governance engagements specifically. This
corrected an error: the first two passes concluded governance was "a protocol
domain, not a pattern", but that conclusion rested on incomplete evidence —
**three of the governance reports had never parsed into the finding index and
so had never been read at all.** They use a 2021-era format with no `ID.`
prefix on findings.

Reading them surfaced a pattern family with no home in the library:
time-indexed state — checkpoints, snapshots, epochs, delegation records and
accrual windows. It is not governance-specific; the same defects appear in any
staking, vesting or reward system. It is now covered by
[`omega-time-indexed-state`](omega-time-indexed-state/SKILL.md). The original
judgement was right that *governance* is a domain rather than a pattern, and
wrong to stop there.

A fourth pass audited the extraction itself rather than the reports, by
comparing the findings recovered per report against the totals each report
states in its own severity table. That exposed systematic under-extraction —
worst in the largest report in the archive, where 22 of 96 findings had never
been read, including four rated high. Re-reading the shortfalls surfaced one
more family, now covered by
[`omega-share-and-index-accounting`](omega-share-and-index-accounting/SKILL.md):
balances derived from a shared scalar rather than stored. It appears
independently across three unrelated clients — index-based interest tokens,
multiplier-based rebasing tokens, and a rebasing savings wrapper — which is what
distinguishes it from a single client's quirk.

The same pass added the fail-open access-control default to
[`omega-enforceability-check`](omega-enforceability-check/SKILL.md) §4.

**Method note.** Each pass found material the previous one missed, and the
failures were in extraction rather than in reading: reports whose format the
parser did not match, and findings the regex dropped silently. Anyone extending
this work should reconcile against the reports' own stated totals *first* — it
is the cheapest available check on coverage, and it should have been the first
thing done rather than the fourth.

## Licensing

Team Omega's repository carries **no LICENSE file**; the PDF reports are their
copyrighted work. Nothing in this directory copies that expression.

The skill files here are © 2026 Daoism Systems, MIT, consistent with the
aggregation layer described in the repository root [LICENSE](../../LICENSE).
They are **not** covered by the `sources/<repo>/LICENSE` notices, which apply
only to the verbatim upstream mirrors.

If Team Omega would prefer different treatment of this derivation, the contact
listed in their repository README is `omegaaudits@protonmail.com`.
