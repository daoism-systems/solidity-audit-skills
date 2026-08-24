# Attributions — File-Level Provenance

This file records, per directory under `sources/`, exactly what was copied
from which upstream repo. It exists so that any redistribution of this
repository can satisfy the MIT license obligation ("The above copyright
notice and this permission notice shall be included in all copies or
substantial portions of the Software") with a single auditable record.

`sources/` holds three kinds of directory, and the distinction is load-bearing
for licensing:

| Directory | Kind | Copyright |
|---|---|---|
| `sources/pashov/` | **Mirror** of `pashov/skills` (partial — see below) | © 2024 AI Skills Contributors |
| `sources/plamen/` | **Mirror** of `PlamenTSV/plamen` (partial — see below) | © 2025-2026 Plamen Contributors |
| `sources/quillshield/` | **Mirror** of `quillai-network/quillshield_skills` (partial — see below) | © 2025 QuillShield |
| `sources/omega/` | **Original** — authored by Daoism Systems | © 2026 Daoism Systems |
| `sources/orchestrator/` | **Original** — authored by Daoism Systems | © 2026 Daoism Systems |
| `sources/symbiosis/` | **Derived** — union-merged from the three mirrors | content © respective upstreams, merge © 2026 Daoism Systems |

Each mirror carries its upstream `LICENSE` verbatim. `sources/omega/` carries
none, because it copies nothing; see
[`sources/omega/ATTRIBUTION.md`](sources/omega/ATTRIBUTION.md) for what it was
derived from and how. `sources/orchestrator/` likewise mirrors nothing.
`sources/symbiosis/` is a *derived* collection: every skill in it is a
union-merge of duplicated skills that were removed from the three mirrors, so
its copyright is layered — upstream notices carry over per file, and the merge
itself is © 2026 Daoism Systems (see [`sources/symbiosis/README.md`](sources/symbiosis/README.md)).

The mirrors are **partial**: where two or more upstreams shipped substantively
overlapping skills, the overlapping files were removed from the mirrors and
union-merged into `sources/symbiosis/`. Every removal and edit is recorded in
that mirror's `.provenance` under `known_differences`, with reason
`merged into sources/symbiosis/<skill>`.

## Pinned upstream commits

Upstream content was fetched on **2026-07-20**. The mirrors carry no `.git`, so
each commit below was recovered by content match — hashing every mirrored file
and walking upstream history for the commit whose tree matches — and is
re-verified by `scripts/check_provenance.py`.

| Source | Commit | Dated | Files | Byte-identical |
|---|---|---|---|---|
| `sources/pashov/` | [`c577eb7799c349de0acb187ba00ca98e14e436fd`](https://github.com/pashov/skills/commit/c577eb7799c349de0acb187ba00ca98e14e436fd) | 2026-07-09 | 73 | 69 / 73 ¹ ² |
| `sources/plamen/` | [`795962b96e254f2e423a2635fe7f8cb8ea1e6d69`](https://github.com/PlamenTSV/plamen/commit/795962b96e254f2e423a2635fe7f8cb8ea1e6d69) | 2026-07-15 | 411 | 402 / 411 ² |
| `sources/quillshield/` | [`8bdd3c058704cd855ce29b8e2385708b59152606`](https://github.com/quillai-network/quillshield_skills/commit/8bdd3c058704cd855ce29b8e2385708b59152606) | 2026-03-30 | 60 | 51 / 60 ² |
| `sources/omega/` | — | — | 12 skills | original content, no upstream |
| `sources/symbiosis/` | — | derived 2026-08-19 | 16 | derived, see below |

¹ `mcp-packages/run-node-mcp.cmd` differs only in line endings: CRLF normalised
to LF on checkout. Content is otherwise identical. Recorded in that source's
`known_differences`.

² The gap is the symbiosis merge: duplicated skills were **removed** from the
mirror and every file that named them was **edited** to point at the
`symbiosis` equivalent. Each removal and edit is itemized in that mirror's
`.provenance` under `known_differences`. No file's audit methodology was
changed — edits are limited to cross-references, counts, and install lists.

Each figure is also held machine-readably in `sources/<name>/.provenance`, which
CI validates against what is actually on disk. As of the last verification all
three mirrors were level with their upstream default branch; a weekly job
reports drift.

---

## `sources/pashov/` — from `pashov/skills`

| Path under `sources/pashov/` | Upstream URL |
|---|---|
| `LICENSE` | https://github.com/pashov/skills/blob/main/LICENSE |
| `README.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `.gitignore` | https://github.com/pashov/skills/tree/main |
| `solidity-auditor/` (partial, see removals below) | https://github.com/pashov/skills/tree/main/solidity-auditor |
| `x-ray/` (full) | https://github.com/pashov/skills/tree/main/x-ray |
| `fizz/LICENSE`, `fizz/SKILL.md`, `fizz/README.md`, `fizz/VERSION`, `fizz/agents/`, `fizz/references/`, `fizz/skills/`, `fizz/templates/`, `fizz/evals/` | https://github.com/pashov/skills/tree/main/fizz |

**Copyright notice** (preserved verbatim at `sources/pashov/LICENSE`):

> Copyright (c) 2024 AI Skills Contributors

**Removed and edited for the symbiosis merge** (upstream still has them;
install pashov/skills directly if you need the originals):

- removed `solidity-auditor/references/hacking-agents/access-control-agent.md`
  → [`sources/symbiosis/guard-consistency/`](sources/symbiosis/guard-consistency/)
- removed `solidity-auditor/references/hacking-agents/boundary-agent.md`
  → [`sources/symbiosis/external-call-safety/`](sources/symbiosis/external-call-safety/)
- edited `solidity-auditor/SKILL.md` — bundle table 12→10 agents, spawn
  instructions and dedup phase updated, symbiosis pointer added
- edited `numerical-gap-agent.md`, `asymmetry-agent.md` — boundary-agent
  references now name the symbiosis external-call-safety lens

**Omitted from the mirror** (kept upstream; install pashov/skills directly
if you need them):

- `fizz/scripts/` — JS executors for forge/medusa/echidna (~156 KB)
- `x-ray/scripts/` — Python + bash enumerators (~100 KB)
- `static/`, `.github/`, `.git/`

---

## `sources/plamen/` — from `PlamenTSV/plamen`

| Path under `sources/plamen/` | Upstream URL |
|---|---|
| `LICENSE` | https://github.com/PlamenTSV/plamen/blob/main/LICENSE |
| `README.md`, `SETUP.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `VERSION`, `.plamen-manifest.json`, `pyproject.toml`, `requirements*.txt`, `mcp.json.example`, `settings.json.example` | https://github.com/PlamenTSV/plamen/tree/main |
| `agents/` (partial, see removals below) — 6 depth + 2 analyzer/verifier agent definitions | https://github.com/PlamenTSV/plamen/tree/main/agents |
| `skills/` — `audit-prep/` orchestrator | https://github.com/PlamenTSV/plamen/tree/main/skills |
| `agents/skills/soroban/` (19 skills) | https://github.com/PlamenTSV/plamen/tree/main/agents/skills/soroban |
| `agents/skills/sui/` (22 skills) | https://github.com/PlamenTSV/plamen/tree/main/agents/skills/sui |
| `agents/skills/niche/` (9 cross-language niche agents, 8 after the merge) | https://github.com/PlamenTSV/plamen/tree/main/agents/skills/niche |
| `agents/skills/injectable/` (10 + 25 L1 skills) | https://github.com/PlamenTSV/plamen/tree/main/agents/skills/injectable |
| `rules/` (partial — orchestrator rules, finding format, phase prompts, skill index) | https://github.com/PlamenTSV/plamen/tree/main/rules |
| `prompts/` (partial — per-language phase prompts) | https://github.com/PlamenTSV/plamen/tree/main/prompts |
| `commands/` (slash commands) | https://github.com/PlamenTSV/plamen/tree/main/commands |
| `docs/` (full) | https://github.com/PlamenTSV/plamen/tree/main/docs |
| `custom-mcp/`, `mcp-packages/`, `opengrep-rules/` | https://github.com/PlamenTSV/plamen/tree/main |

**Copyright notice** (preserved verbatim at `sources/plamen/LICENSE`):

> Copyright (c) 2025-2026 Plamen Contributors

**Removed and edited for the symbiosis merge** (upstream still has them;
install PlamenTSV/plamen directly if you need the originals):

- removed `agents/skills/evm/oracle-analysis/SKILL.md`
- removed `agents/skills/evm/flash-loan-interaction/SKILL.md`
  → both union-merged into
  [`sources/symbiosis/oracle-flashloan-analysis/`](sources/symbiosis/oracle-flashloan-analysis/)
- removed `agents/skills/niche/signature-verification-audit/SKILL.md`
  → union-merged into
  [`sources/symbiosis/signature-replay/`](sources/symbiosis/signature-replay/)
  (kept authoritative for non-EVM languages upstream; EVM dispatches through
  symbiosis)
- edited `rules/skill-index.md` — EVM skill table 18→15, symbiosis row added
- edited `prompts/evm/generic-security-rules.md` (R15/R16 pointers),
  `prompts/evm/phase1-recon-prompt.md`,
  `prompts/evm/v2/phase1-recon-prompt.md`,
  `prompts/evm/self-check-checklists.md` — FLASH_LOAN/ORACLE references now
  name the symbiosis merged lens
- kept `agents/depth-state-trace.md` — it is load-bearing in plamen's own
  pipeline; its EVM dispatch is superseded by
  [`sources/symbiosis/invariant-conservation/`](sources/symbiosis/invariant-conservation/)

**Omitted from the mirror** (kept upstream; install PlamenTSV/plamen
directly if you need them):

- `plamen.py` (300 KB Python driver) and `plamen`, `plamen.sh`, `plamen.bat`,
  `_avm_installer.py`, `_solana_installer.py`, `_sui_installer.py`,
  `write_dedup.py`
- `scripts/` (~8 MB of one-off helpers, validators, installers)
- `codex-adapter/`, `plamen_l1/`, `CHANGELOG.md`, `agent-transcripts/`,
  `.git/`, `.github/`

---

## `sources/quillshield/` — from `quillai-network/quillshield_skills`

| Path under `sources/quillshield/` | Upstream URL |
|---|---|
| `LICENSE` | https://github.com/quillai-network/quillshield_skills/blob/main/LICENSE |
| `README.md`, `INSTALL.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `.gitignore`, `.markdownlint.yaml`, `install.sh` | https://github.com/quillai-network/quillshield_skills/tree/main |
| `.claude-plugin/` (marketplace manifest) | https://github.com/quillai-network/quillshield_skills/tree/main/.claude-plugin |
| `plugins/behavioral-state-analysis/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/behavioral-state-analysis |
| `plugins/reentrancy-pattern-analysis/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/reentrancy-pattern-analysis |
| `plugins/proxy-upgrade-safety/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/proxy-upgrade-safety |
| `plugins/input-arithmetic-safety/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/input-arithmetic-safety |
| `plugins/dos-griefing-analysis/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/dos-griefing-analysis |
| `plugins/defender/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/defender |

**Copyright notice** (preserved verbatim at `sources/quillshield/LICENSE`):

> Copyright (c) 2025 QuillShield (https://github.com/quillai-network)

**Removed and edited for the symbiosis merge** (upstream still has them;
install quillai-network/quillshield_skills directly if you need the
originals). Five plugins were removed and union-merged:

- `plugins/oracle-flashloan-analysis/`
  → [`sources/symbiosis/oracle-flashloan-analysis/`](sources/symbiosis/oracle-flashloan-analysis/)
- `plugins/state-invariant-detection/`
  → [`sources/symbiosis/invariant-conservation/`](sources/symbiosis/invariant-conservation/)
- `plugins/signature-replay-analysis/`
  → [`sources/symbiosis/signature-replay/`](sources/symbiosis/signature-replay/)
- `plugins/external-call-safety/`
  → [`sources/symbiosis/external-call-safety/`](sources/symbiosis/external-call-safety/)
- `plugins/semantic-guard-analysis/`
  → [`sources/symbiosis/guard-consistency/`](sources/symbiosis/guard-consistency/)

Edited to match: `README.md`, `INSTALL.md`, `install.sh`,
`.claude-plugin/marketplace.json` (plugin lists 10→5) and the five remaining
plugin `SKILL.md` files (cross-references now name the symbiosis
equivalents).

**Omitted from the mirror:** `.git/`, `.github/`. Everything else in the
upstream repo is included; the repo is small (~848 KB total).

---

## `sources/symbiosis/` — derived from the three mirrors

Not a mirror: there is no `symbiosis` upstream. Each of its five skills is a
**union-merge** of duplicated skills removed from the three mirrors, so the
collection never loads two agents for the same topic. Full merge map and
lens-attribution scheme: [`sources/symbiosis/README.md`](sources/symbiosis/README.md).

| Symbiosis skill | Union of | Lens tags |
|---|---|---|
| `oracle-flashloan-analysis/` | plamen `oracle-analysis` + plamen `flash-loan-interaction` + quillshield `oracle-flashloan-analysis` | `[L]` `[Q]` |
| `invariant-conservation/` | plamen `depth-state-trace` (superseded, kept upstream) + quillshield `state-invariant-detection` | `[L]` `[Q]` |
| `signature-replay/` | plamen `signature-verification-audit` (EVM) + quillshield `signature-replay-analysis` | `[L]` `[Q]` |
| `external-call-safety/` | pashov `boundary-agent` + quillshield `external-call-safety` | `[P]` `[Q]` |
| `guard-consistency/` | pashov `access-control-agent` + quillshield `semantic-guard-analysis` | `[P]` `[Q]` |

Merge rules: only substantively identical checklist rows were deduplicated;
every unique row, question, and reference survives. Each finding keeps its
lens tag so Tier-3 cross-verification still counts findings by originating
methodology. `references/` directories are the quillshield reference packs
copied verbatim.

**Copyright** is layered: content © the respective upstream contributors
(pashov/plamen/quillshield, MIT — their notices are preserved in each mirror's
`LICENSE`), merge © 2026 Daoism Systems, MIT. The collection records its
upstreams and per-file mapping in
[`sources/symbiosis/.provenance`](sources/symbiosis/.provenance) with
`kind: "derived"`.

---

## Originally-authored files (root, `library/`, and `sources/omega/`)

These are original work by Daoism Systems, released under the same MIT terms
as the upstream sources for convenience. No upstream skill content is
duplicated in any of them.

| File | Original authorship |
|---|---|
| `LICENSE` (root) | MIT notice — Daoism Systems aggregator + 3 upstreams attributed. |
| `README.md` (root) | © 2026 Daoism Systems. |
| `ATTRIBUTIONS.md` (this file) | © 2026 Daoism Systems. |
| `CORRELATIONS.md` | © 2026 Daoism Systems. The correlation analysis is original editorial work; skill descriptions paraphrase upstream SKILL.md frontmatter. |
| `library/*.md` | © 2026 Daoism Systems. Indexes point to upstream files; no skill content is duplicated. |
| `sources/orchestrator/**` | © 2026 Daoism Systems. Original routing/cross-verification layer; holds no audit knowledge of its own. |
| `sources/omega/**` | © 2026 Daoism Systems. Original skill prose derived from the methodology in [OmegaAudits/audits](https://github.com/OmegaAudits/audits); no Team Omega report text is reproduced. Full derivation record in [`sources/omega/ATTRIBUTION.md`](sources/omega/ATTRIBUTION.md). |

---

## Reproducing this mirror

To rebuild the mirrored parts of `sources/` **at the exact commits this repo
carries**, clone and check out the pinned SHAs above:

```bash
mkdir -p sources && cd sources

git clone https://github.com/pashov/skills.git pashov
git -C pashov checkout c577eb7799c349de0acb187ba00ca98e14e436fd

git clone https://github.com/PlamenTSV/plamen.git plamen
git -C plamen checkout 795962b96e254f2e423a2635fe7f8cb8ea1e6d69

git clone https://github.com/quillai-network/quillshield_skills.git quillshield
git -C quillshield checkout 8bdd3c058704cd855ce29b8e2385708b59152606

# Then remove the omitted directories listed above for each repo,
# and the .git directories.
```

That reproduces the **full** upstream trees. To arrive at what this repo
carries, additionally apply the symbiosis merge documented in each mirror's
`known_differences`: delete the duplicated skills listed above and apply the
listed cross-reference edits (or re-derive them from
[`sources/symbiosis/README.md`](sources/symbiosis/README.md), which holds the
full merge map).

Cloning without the checkout step reproduces *whatever upstream HEAD is today*,
which is a different artifact. Verify a rebuild with:

```bash
python3 scripts/check_provenance.py
```

`sources/omega/` and `sources/orchestrator/` are not reproducible this way —
they are original writing, not clones, and have no upstream to fetch.
`sources/symbiosis/` is reproducible only from the three mirrors at their
pinned commits, via the merge map in its README.

Pin to a specific tag (`--branch vX.Y.Z`) for reproducible provenance.
