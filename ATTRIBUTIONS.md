# Attributions — File-Level Provenance

This file records, per directory under `sources/`, exactly what was copied
from which upstream repo. It exists so that any redistribution of this
repository can satisfy the MIT license obligation ("The above copyright
notice and this permission notice shall be included in all copies or
substantial portions of the Software") with a single auditable record.

All upstream content was fetched via `git clone --depth 1` on
**2026-07-20**. The "commit-ish" column is the default branch HEAD at fetch
time; for a guaranteed reproducible build, pin to a tag from the upstream
repo.

---

## `sources/pashov/` — from `pashov/skills`

| Path under `sources/pashov/` | Upstream URL |
|---|---|
| `LICENSE` | https://github.com/pashov/skills/blob/main/LICENSE |
| `README.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `.gitignore` | https://github.com/pashov/skills/tree/main |
| `solidity-auditor/` (full) | https://github.com/pashov/skills/tree/main/solidity-auditor |
| `x-ray/` (full) | https://github.com/pashov/skills/tree/main/x-ray |
| `fizz/LICENSE`, `fizz/SKILL.md`, `fizz/README.md`, `fizz/VERSION`, `fizz/agents/`, `fizz/references/`, `fizz/skills/`, `fizz/templates/`, `fizz/evals/` | https://github.com/pashov/skills/tree/main/fizz |

**Copyright notice** (preserved verbatim at `sources/pashov/LICENSE`):

> Copyright (c) 2024 AI Skills Contributors

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
| `agents/` (full) — 6 depth + 2 analyzer/verifier agent definitions | https://github.com/PlamenTSV/plamen/tree/main/agents |
| `skills/` — `audit-prep/` orchestrator | https://github.com/PlamenTSV/plamen/tree/main/skills |
| `agents/skills/soroban/` (19 skills) | https://github.com/PlamenTSV/plamen/tree/main/agents/skills/soroban |
| `agents/skills/sui/` (22 skills) | https://github.com/PlamenTSV/plamen/tree/main/agents/skills/sui |
| `agents/skills/niche/` (9 cross-language niche agents) | https://github.com/PlamenTSV/plamen/tree/main/agents/skills/niche |
| `agents/skills/injectable/` (10 + 25 L1 skills) | https://github.com/PlamenTSV/plamen/tree/main/agents/skills/injectable |
| `rules/` (full — orchestrator rules, finding format, phase prompts, skill index) | https://github.com/PlamenTSV/plamen/tree/main/rules |
| `prompts/` (full — per-language phase prompts) | https://github.com/PlamenTSV/plamen/tree/main/prompts |
| `commands/` (slash commands) | https://github.com/PlamenTSV/plamen/tree/main/commands |
| `docs/` (full) | https://github.com/PlamenTSV/plamen/tree/main/docs |
| `custom-mcp/`, `mcp-packages/`, `opengrep-rules/` | https://github.com/PlamenTSV/plamen/tree/main |

**Copyright notice** (preserved verbatim at `sources/plamen/LICENSE`):

> Copyright (c) 2025-2026 Plamen Contributors

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
| `plugins/semantic-guard-analysis/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/semantic-guard-analysis |
| `plugins/state-invariant-detection/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/state-invariant-detection |
| `plugins/reentrancy-pattern-analysis/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/reentrancy-pattern-analysis |
| `plugins/oracle-flashloan-analysis/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/oracle-flashloan-analysis |
| `plugins/proxy-upgrade-safety/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/proxy-upgrade-safety |
| `plugins/input-arithmetic-safety/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/input-arithmetic-safety |
| `plugins/external-call-safety/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/external-call-safety |
| `plugins/signature-replay-analysis/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/signature-replay-analysis |
| `plugins/dos-griefing-analysis/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/dos-griefing-analysis |
| `plugins/defender/` | https://github.com/quillai-network/quillshield_skills/tree/main/plugins/defender |

**Copyright notice** (preserved verbatim at `sources/quillshield/LICENSE`):

> Copyright (c) 2025 QuillShield (https://github.com/quillai-network)

**Omitted from the mirror:** `.git/`, `.github/`. Everything else in the
upstream repo is included; the repo is small (~848 KB total).

---

## Aggregation-layer files (this repo's root and `library/`)

These files are original editorial work by Daoism Systems, released under
the same MIT terms as the upstream sources for convenience. Skill content is
**never** duplicated here — these files only point into `sources/`.

| File | Original authorship |
|---|---|
| `LICENSE` (root) | MIT notice — Daoism Systems aggregator + 3 upstreams attributed. |
| `README.md` (root) | © 2026 Daoism Systems. |
| `ATTRIBUTIONS.md` (this file) | © 2026 Daoism Systems. |
| `CORRELATIONS.md` | © 2026 Daoism Systems. The correlation analysis is original editorial work; skill descriptions paraphrase upstream SKILL.md frontmatter. |
| `library/*.md` | © 2026 Daoism Systems. Indexes point to upstream files; no skill content is duplicated. |
| `skills/omega/**` | © 2026 Daoism Systems. Original skill prose derived from the methodology in [OmegaAudits/audits](https://github.com/OmegaAudits/audits); no Team Omega report text is reproduced. Full derivation record in [`skills/omega/ATTRIBUTION.md`](skills/omega/ATTRIBUTION.md). |

---

## Reproducing this mirror

If you want to rebuild `sources/` from scratch:

```bash
mkdir -p sources && cd sources
git clone --depth 1 https://github.com/pashov/skills.git              pashov
git clone --depth 1 https://github.com/PlamenTSV/plamen.git           plamen
git clone --depth 1 https://github.com/quillai-network/quillshield_skills.git quillshield
# Then remove the omitted directories listed above for each repo.
```

Pin to a specific tag (`--branch vX.Y.Z`) for reproducible provenance.
