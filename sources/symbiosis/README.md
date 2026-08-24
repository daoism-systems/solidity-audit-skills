# Symbiosis — merged duplicate skills

Five skills that each merge two or three skills from the pashov, plamen and
quillshield collections that covered the same ground. Merging removes the
per-agent re-reading of the whole codebase bundle — one leaf agent now covers
what two or three covered before — while the merged skill is a **union** of
its constituents: every checklist item, taxonomy, reference pack, severity
table and rationalization list from the originals survives, and only
substantively identical rows are collapsed.

Each merged skill records which constituent lens (pashov / plamen /
quillshield) each section derives from, so the Tier 3 cross-verification
tier can still attribute a finding to its originating methodology — one
merged agent can still cast "pashov" and "quillshield" votes.

## Why these five

They are the pairs and triples where content genuinely duplicated rather
than complemented (see `CORRELATIONS.md` at the repo root for the full
matrix). Complementary overlaps — reentrancy (quillshield deep, pashov one
boundary case), pre-audit (x-ray produces architecture, audit-prep produces
a scored report) — were left in their home collections.

## Merge map

| Merged skill | Absorbed from pashov | Absorbed from plamen | Absorbed from quillshield |
|---|---|---|---|
| `oracle-flashloan-analysis` | — | `agents/skills/evm/oracle-analysis`, `agents/skills/evm/flash-loan-interaction` | `plugins/oracle-flashloan-analysis` |
| `invariant-conservation` | — | `agents/depth-state-trace.md` | `plugins/state-invariant-detection` |
| `signature-replay` | — | `agents/skills/niche/signature-verification-audit` | `plugins/signature-replay-analysis` |
| `external-call-safety` | `hacking-agents/boundary-agent.md` | — | `plugins/external-call-safety` |
| `guard-consistency` | `hacking-agents/access-control-agent.md` | — | `plugins/semantic-guard-analysis` |

The originals were deleted from the mirrors; each mirror's `.provenance`
records the removals in `known_differences`.

## Licensing

Upstream content retains its original copyright and MIT license:

- pashov-derived sections: MIT, (c) 2024 AI Skills Contributors
- plamen-derived sections: MIT, (c) 2025-2026 Plamen Contributors
- quillshield-derived sections: MIT, (c) 2025 QuillShield
- the merge itself: MIT, (c) 2026 Daoism Systems

See `/ATTRIBUTIONS.md` at the repo root and the pinned commits in
`.provenance`.
