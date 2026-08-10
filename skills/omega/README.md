# Omega Skills

Eight custom skills distilled from **Team Omega**'s public audit archive
([OmegaAudits/audits](https://github.com/OmegaAudits/audits)) — 55 reports
published between March 2021 and July 2026, covering 834 catalogued findings
(2 critical, 46 high, 105 medium, 332 low, 349 info).

Unlike `sources/`, this directory is **not** a mirror. These are originally
authored skill files that encode the *reasoning patterns and process* observed
across the archive. Every non-obvious claim cites the report and finding ID it
was derived from, so you can go read the primary source. See
[ATTRIBUTION.md](ATTRIBUTION.md).

## Why a separate Omega set

The three mirrored collections (pashov, plamen, quillshield) are organized by
**bug class** — reentrancy, oracle, arithmetic, proxy. Omega's archive
suggests a different decomposition. Their highest-yield findings rarely come
from a bug-class checklist; they come from a small number of *questions asked
about every contract*:

- Can every asset that goes in come back out, in every reachable state?
- Does this check actually constrain the party it names?
- Is every counter updated on *every* path that changes what it counts?
- What exactly are we trusting this off-chain value for?

These cut across bug classes, and none of the three mirrored sources organizes
around them. That is the gap this set fills.

## The skills

| Skill | Question it asks | Omega evidence |
|---|---|---|
| [omega-audit-workflow](omega-audit-workflow/SKILL.md) | How do I run and write up the whole engagement? | Methodology + report structure, all 55 reports |
| [omega-asset-exit-paths](omega-asset-exit-paths/SKILL.md) | Can every asset get back out? | ~⅓ of all high-severity findings |
| [omega-enforceability-check](omega-enforceability-check/SKILL.md) | Does this guard actually guard? | Blindex P2, dxGov W1/P6, Toucan T2, Giza CPP1/M1 |
| [omega-accounting-consistency](omega-accounting-consistency/SKILL.md) | Is this counter right on every path? | Altitude HM1, dxDAO VM13–VM15, PrimeDAO S4/S17 |
| [omega-external-data-trust](omega-external-data-trust/SKILL.md) | What am I trusting, and for what? | Giza (5 reports), Inverter iTRY G1/G2, Blindex F1 |
| [omega-upgrade-diff-review](omega-upgrade-diff-review/SKILL.md) | What changed, and what did the change break? | Backed ×9, Altitude ×5, Gnosis Bridge ×4 |
| [omega-ordering-and-approval-races](omega-ordering-and-approval-races/SKILL.md) | Who profits from reordering this? | Spectre V1/B2, Karpatkey K2, Gnosis XDFB1 |
| [omega-repo-hygiene-sweep](omega-repo-hygiene-sweep/SKILL.md) | Is the repo itself sound? | The "General" section of every report, 2021→2026 |

## How to use

Start with `omega-audit-workflow` — it is the orchestrator and tells you when
to reach for the other seven. Or load an individual lens directly:

```
@solidity-audit-skills/skills/omega/omega-asset-exit-paths/SKILL.md
```

## Relationship to the mirrored sources

Complementary, not competing. Omega's lenses are *framing* questions; the
mirrored collections have deeper per-bug-class catalogues. Suggested pairings:

| Omega skill | Pair with |
|---|---|
| `omega-asset-exit-paths` | **[Q]** `dos-griefing-analysis` (push-vs-pull mechanics) |
| `omega-enforceability-check` | **[Q]** `semantic-guard-analysis` (finds missing guards; Omega's finds *ineffective* ones) |
| `omega-accounting-consistency` | **[Q]** `state-invariant-detection` (invariant inference) |
| `omega-external-data-trust` | **[Q]** `oracle-flashloan-analysis` (manipulation mechanics) |
| `omega-upgrade-diff-review` | **[Q]** `proxy-upgrade-safety` (storage-collision mechanics) |
| `omega-ordering-and-approval-races` | **[P]** `hacking-agents/` attacker framing |
| `omega-audit-workflow` | **[P]** `solidity-auditor` (12-agent parallel sweep as the breadth phase) |

Source tags follow [../../library/README.md](../../library/README.md):
**[P]** pashov · **[L]** plamen · **[Q]** quillshield.
