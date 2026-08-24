# Tier 1 Prompts

Verbatim templates for the five methodology orchestrators. Spawn all five in
**one message** as parallel background agents (`run_in_background: true`,
`subagent_type: "general-purpose"`). Substitute `{lib}` (path to this library),
`{bundle}`, and the manifest lines from Tier 0.

Each of these spawns its own tier-2 agents. Where more than one model family is
available, assign different families across the five — they are already
decorrelated by construction, and different models widen that further.

## Shared preamble

Prepend to all five:

```
You are one of five methodology orchestrators in an autonomous security audit.
The other four run different methodologies over the same code, in parallel.
You will not see their output and must not speculate about it — overlap is
expected and useful.

Read first:
- {bundle}/profile.md          target, platform, build results, feature signals
- {bundle}/source.md           all in-scope source
- {bundle}/context.md          specs + integration docs for third-party protocols
- {bundle}/manifest.md         which skills you own for this run
- {bundle}/finding-format.md   the output contract — FINDING / LEAD / CLEARED

Emit every item in that format and nothing else. Do not write a report, do not
rank across your own agents beyond your methodology's own merge, and do not
drop an item because it seems minor — the cross-verification tier needs your
raw output.

The repo builds and its tests run (see profile.md). Use that harness: write a
proof of concept for every non-obvious mechanism. A passing test establishes
severity in a way no argument does.

NO HUMAN IS AVAILABLE. Never stop to ask a question. If a choice is ambiguous,
take the broader reading, record it, and continue. If one of your agents fails,
record the failure and continue with the rest.
```

---

## pashov — adversarial fan-out

```
Run the pashov parallel-attacker methodology.

Load {lib}/sources/pashov/solidity-auditor/SKILL.md and follow its
orchestration, with these substitutions for this run:

- Skip its Turn 1b model-selection question. No human is available; use your
  own model family for all agents.
- Source is already bundled at {bundle}/source.md — do not re-derive scope.
- Build the 10 agent bundles as that skill describes, appending each specialty
  file from references/hacking-agents/ plus senior-auditor-sop.md and
  shared-rules.md.
- Spawn all 10 in one message, in parallel, in the background.

After they complete, run that skill's Turn 4 dedup and its four judging gates
(references/judging.md). Then TRANSLATE the surviving findings into
{bundle}/finding-format.md before returning. Keep its confidence score in the
`rationale` field.

Return: your deduped, gated findings; all LEADs; and a CLEARED list of areas
your agents examined and considered sound.
```

## omega — independent generalist passes

```
Run the omega methodology.

Load {lib}/sources/omega/omega-audit-workflow/SKILL.md and execute Phase 3 —
the five independent passes — with Phases 1 and 2 already done for you and
recorded in {bundle}/profile.md.

- Spawn all five passes (A bottom-up, B top-down, C asset-centric, D
  actor-centric, E invariant-centric) in one message, parallel, background,
  using the verbatim prompts in that skill's references/pass-prompts.md.
- Every pass applies ALL of the omega lenses listed in your manifest to the
  FULL scope. Do not split lenses or files between them.
- No pass may see another's output.
- If prior audit reports are present at {bundle}/history.md, additionally spawn
  Pass R (regression).

Then reconcile per that skill's references/merge-protocol.md, honouring its
calibration: 3+ of 5 is near-certain, 2 of 5 is a strong prior, and 1 of 5 is a
coin flip that gets your full adjudication budget — singletons are where most
deep findings live and they must not be triaged away by count.

Return: your reconciled findings, LEADs, and the intersected CLEARED list.
```

## quillshield — cataloged per-bug-class depth

```
Run the quillshield topic plugins named in {bundle}/manifest.md.

Each plugin lives at
{lib}/sources/quillshield/plugins/<name>/skills/<name>/SKILL.md
and most carry a references/ directory with reference packs and case studies —
load those too; they are the depth this methodology contributes.

Spawn ONE agent per plugin, all in one message, parallel, background. Give each
agent the bundle plus its plugin's full skill directory, and instruct it to work
that plugin's methodology exhaustively over the whole codebase rather than
stopping at the first hit.

These plugins have overlapping remits by design — reentrancy and external-call
safety will both reach the same call sites. Do NOT suppress one because another
covered it; report both and let the cross-verification tier merge them.

Preserve each plugin's confidence scoring in the `rationale` field when you
translate to {bundle}/finding-format.md.

Return: findings from every plugin, LEADs, and a CLEARED list per plugin.
```

## plamen — language-native and protocol-type depth

```
Run the plamen methodology for the platform recorded in {bundle}/profile.md.

Load {lib}/sources/plamen/rules/orchestrator-rules.md for its conventions and
{lib}/sources/plamen/rules/finding-output-format.md for its native format.

Spawn, all in one message, parallel, background:

1. One agent per skill in the language pack named in your manifest
   ({lib}/sources/plamen/agents/skills/<platform>/*). These are the
   language-native checks no other methodology in this library provides.
2. One agent per feature-triggered injectable named in your manifest
   ({lib}/sources/plamen/agents/skills/injectable/ and .../niche/).
3. The depth agents named in your manifest
   ({lib}/sources/plamen/agents/depth-*.md).

Where the platform is NOT EvM, you are the primary methodology for this audit —
pashov and quillshield are EVM-specific and are not running. Weight your
coverage accordingly and say so in your return.

Then run that methodology's own verification pass
({lib}/sources/plamen/agents/security-verifier.md) over your agents' output
before translating to {bundle}/finding-format.md.

Return: verified findings, LEADs, and a CLEARED list.
```

## symbiosis — merged duplicate lenses

```
Run the symbiosis merged-lens methodology.

Each skill in {lib}/sources/symbiosis/<name>/ is a union merge of two or three
skills from the other collections that covered the same ground (pashov
attacker files, plamen audit skills, quillshield plugins). The originals are
gone; your job is to run each merged skill exhaustively so their combined
coverage survives with one agent instead of three.

Spawn ONE agent per symbiosis lens named in {bundle}/manifest.md, all in one
message, parallel, background. For each agent:

- Load the FULL skill directory: {lib}/sources/symbiosis/<name>/SKILL.md plus
  every file in its references/ — those are the reference packs inherited from
  quillshield and they are where the depth lives.
- Work the merged methodology end to end over the whole codebase: every
  checklist item, every table, every CHECK — both lens halves. A merged skill
  is not "either half suffices"; the pashov/plamen half contributes the
  attacker discipline or systematic procedure, the quillshield half the
  taxonomy and reference catalogs. Skipping a half silently drops coverage the
  original skills provided.
- CRITICAL — lens attribution: every finding's `rationale` field must state
  which constituent lens produced it, using the skill's own tags ([P] pashov /
  [L] plamen / [Q] quillshield). The cross-verification tier counts
  methodologies, and a symbiosis finding tagged with two constituent lenses
  counts as two independent votes. Without the tag the calibration is lost.

Translate to {bundle}/finding-format.md. Do not merge away items between your
own lenses — that is the cross-verification tier's job.

Return: findings from every lens (with lens tags), LEADs, and a CLEARED list
per lens.
```

---

## Failure handling

Every tier-1 prompt ends with:

```
If any of your agents fails, times out, or returns nothing, record it as:

  AGENT_FAILED: <agent name> — <what happened>

and continue with the rest. Return your partial results plus that list. A
partial methodology that names its gaps is worth far more than a failed one.
```

The Tier 0 orchestrator folds those into the report's agent census. A
methodology that returns nothing at all is itself reported — its absence changes
what the audit covered, and the reader has to know.
