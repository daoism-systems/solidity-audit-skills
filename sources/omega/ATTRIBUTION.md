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
off-chain service surface, which most Solidity-only skill sets omit.

## Licensing

Team Omega's repository carries **no LICENSE file**; the PDF reports are their
copyrighted work. Nothing in this directory copies that expression.

The skill files here are © 2026 Daoism Systems, MIT, consistent with the
aggregation layer described in the repository root [LICENSE](../../LICENSE).
They are **not** covered by the `sources/<repo>/LICENSE` notices, which apply
only to the verbatim upstream mirrors.

If Team Omega would prefer different treatment of this derivation, the contact
listed in their repository README is `omegaaudits@protonmail.com`.
