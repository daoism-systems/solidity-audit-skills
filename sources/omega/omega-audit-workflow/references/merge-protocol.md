# Merge Protocol

How the orchestrator reconciles Pass A, Pass B and Pass R into one finding set.

## What makes this different from a specialist fan-out

A 12-specialist sweep expects overlap to be *noise*: the math agent and the
invariant agent both flag the same line because their remits intersect, so
agreement carries little information and the merge is mostly deduplication.

Two independent generalist passes are the opposite. Both saw the whole scope and
had the opportunity to find everything. So:

- **Agreement is evidence.** Two reviews converging independently is the
  strongest confidence signal available in the engagement — stronger than either
  pass's own stated confidence.
- **Disagreement is a finding generator.** When one pass says a guard holds and
  the other says it does not, one of them is wrong about the code, and resolving
  which is usually where the real bug is.
- **A single-pass finding is normal.** Do not treat "only one auditor found it"
  as a weak signal. It is the expected case for a large share of real findings,
  and discounting it destroys the value of running two passes at all.

That last point is the failure mode this protocol exists to prevent. An
orchestrator that quietly prefers corroborated findings will converge on the
*intersection* of two reviews, which is strictly worse than either review alone.
**You want the union, adjudicated — not the intersection.**

## Procedure

### 1. Normalize

Parse every FINDING and LEAD from both passes into the common format. Key each
on `(file, function, mechanism)`.

**Key on mechanism, not on title.** Two passes will describe the same defect in
different words, and will describe different defects in similar words. The test
for "same finding" is whether the *same code change* fixes both.

### 2. Classify each item

| Class | Meaning | Action |
|---|---|---|
| **Corroborated** | Both passes, same mechanism | Merge. Keep the better write-up; keep *both* PoCs if they differ. Note the corroboration internally — it raises confidence, and it does not go in the client report. |
| **Single-pass** | One pass only | Adjudicate on evidence alone. Its provenance is not a factor. |
| **Contested** | Both passes reached the same code and disagree | Resolve in the code (§3). Never average, never drop. |
| **Divergent mechanism** | Same function, genuinely different defects | Keep both as separate findings. Do not collapse. |
| **Lead** | Neither pass could close it | §4. |

### 3. Resolve contested items

Go to the code and settle it yourself. Read the guard both passes discussed, the
paths that reach it, and the state it depends on.

Three outcomes:

- **One pass is right.** Take that finding, and record the mistaken reading in
  your notes — if a competent reviewer misread this code, that is worth a line in
  the report even when the code is correct.
- **Both are partly right.** Usually means the behaviour is state-dependent and
  neither pass enumerated the states. That is itself the finding: write it as one.
- **You cannot settle it.** Then the code is genuinely unclear, and that is
  reportable at `info` at minimum: two independent competent reviews reached
  opposite conclusions from the same source.

Do not resolve a contest by asking either pass. They are finished; re-engaging
one of them anchors the answer to whichever you asked first.

### 4. Promote or retire leads

A lead becomes a finding when you can close it: name the mechanism, name the
consequence, and produce a path or a PoC. Spend real effort here — leads are
where the highest-severity findings usually hide, precisely because they were
hard enough that one pass could not finish them.

A lead that survives your attempt and describes real uncertainty about the code
belongs in the report as `info`. A lead that is simply wrong gets dropped, and
you should be able to say why in one sentence.

**A lead raised by both passes independently is not a lead — it is a finding you
have not closed yet.** Prioritize those first.

### 5. Fold in Pass R

Add Pass R's rows to the report as their own section. Two rules:

- `REGRESSED` items are **findings**, not history. File each with the current
  severity, cross-referenced to the original report and ID.
- `CONTINGENT` items need the contingency re-verified against the current
  deployment plan. A mitigation that was true at the last audit may not be true
  now. If it no longer holds, it is a live finding again.

If a regression collides with a finding from Pass A or B, merge them and keep
the history — "this was reported as X in the prior report, fixed, and has since
returned" is a materially stronger finding than either half.

### 6. Merge the cleared lists

Intersect them. An area belongs in the report's "checked and found sound"
section when **both** passes examined it and neither raised anything.

Where only one pass cleared an area, it was reviewed once, not twice. Either
review it yourself before claiming it, or leave it out. Do not let the cleared
section inherit a confidence the review did not earn — it is the part of the
report a reader is least able to check.

### 7. Cross-reference and rank

Root causes get one finding; consequences point at it. Where one defect produces
several downstream symptoms — and independent passes often report the symptoms
separately — collapse them into one finding with the others cross-referenced.

Then rank by severity per the Phase 4 ladder, and number by file per Phase 5.

## Completeness check

Before writing the report, print this and confirm it:

```
Pass A:      N findings, N leads, N cleared
Pass B:      N findings, N leads, N cleared
Pass R:      N rows  (N regressed, N still open, N contingent)

Corroborated:      N
Single-pass (A):   N
Single-pass (B):   N
Contested:         N   → all resolved? yes/no
Leads promoted:    N
Leads retired:     N   → each with a stated reason?

Final findings:    N
```

Every raw item from every pass must appear in exactly one bucket. A raw finding
that is in no bucket has been silently dropped — go back and find it.

## What not to do

- **Do not review the code yourself before the passes return.** An orchestrator
  with its own view anchors every adjudication that follows.
- **Do not let either pass see the other's output**, during the review or during
  reconciliation. There is no legitimate reason to, and it retroactively destroys
  the independence you paid for.
- **Do not weight by pass.** Neither entry point is more authoritative. If you
  find yourself trusting one systematically, that is a signal about your prompt
  symmetry, not about the code.
- **Do not report corroboration counts to the client.** "Both auditors found
  this" is internal calibration. The client needs the mechanism, the consequence
  and the severity.
