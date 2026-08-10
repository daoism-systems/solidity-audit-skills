---
name: omega-repo-hygiene-sweep
description: Run Team Omega's General-section sweep — the repository-level pass that opens every one of their reports. Covers dependency pinning and advisories, licensing and copyright compliance, test coverage and CI, compiler and linter warnings, Solidity version policy, dead code, visibility, event indexing, custom errors, and documentation drift. Produces the G-series findings. Use at the start of any audit, before reading contract logic, or when writing up the General section of a report.
---

# Repo Hygiene Sweep

Every Team Omega report opens with a **General** section — `G1`, `G2`, `G3` …
— covering the repository rather than any one contract. It is the most stable
part of their methodology: the 2021 DXdao report and the 2026 Backed report
share most of the same items.

It is also cheap. Run it first, in parallel with the build, before you read any
logic. It produces most of the `low` and `info` volume, gets it out of the way,
and tells you a lot about how much to trust the code you are about to read.

> Omega sometimes just says it. `lsdai G1` is titled "General state of the
> repository": "The project's setup is not complete. The project mixes foundry
> with hardhat and npm — we recommend you choose either framework. There is no
> documentation for developers for installing or running…" Filed as its own
> finding.

---

## Dependencies

**Pin everything.** The single most repeated finding in the archive — it appears
in over half the reports (`Fragmint G3`, `Altitude G4`, `Euler G1`,
`Karpatkey G1`, `Backed G2`, `Society G1`, `freename-snap G2`, `Yieldnest G1`,
and on). `Giza-Pendle PP` gives the reasoning for a Python project, and it
generalises:

> This makes the exact builds unpredictable, and makes it easier for
> compromised new package versions to be included in the build.

Pin Solidity dependencies in `package.json` / `foundry.toml` / `remappings`, and
pin the compiler (`Spectre G2`, `Toucan G1`, `PrimeDAO-LBP M1` — "a floating
pragma is set instead of a fixed pragma"). `Karpatkey G1` is filed at **medium**:
"OpenZeppelin dependency is not pinned to a release" — pointing at a git ref
rather than a published version is worse than a loose semver range.

**Run the dependency audit and act on it.** `Blindex G2` **[medium]** reports
122 npm vulnerabilities and quotes the two OpenZeppelin `TimelockController`
advisories by GHSA ID. `EnterDAO G3`, `PrimeDAO-LBP G2` ("OpenZeppelin
dependency has a critical vulnerability"), `Delphia`. Cite the advisory ID.

**Remove what you do not use.** `Backed G3`, `OpusEdu G4` ("move dependencies to
devDependencies and remove unused dependencies"), `Backed-token-bridge G4`.

**No `draft-` or alpha code in production.** `Spectre E3` — inheriting
`draft-ERC20PermitUpgradeable.sol`; `dxGovernance F1` **[medium]** — "do not use
'draft' contracts in production"; `Blindex G4` — "do not include solidity code
released in 'alpha'."

## Licensing and copyright

Omega is unusually rigorous here and rates violations at **medium**. No other
collection in this library covers it at all.

- `EnterDAO Y1` **[medium]** — the staking contract is adapted from Synthetix,
  published under MIT. "You can republish the files under MIT, but you must, as
  the LICENSE file in that repository says, maintain the original copyright
  statements … **What you definitely cannot do is claim copyright yourself.**"
- `Blindex G1` **[medium]** — "GPL code is published under MIT license."
- `EnterDAO G1` / `Delphia G1` / `Fragmint G4` — licensing violations, incomplete
  licensing information.
- `Altitude G2` **[medium]** — the fullest statement. Six different SPDX
  identifiers across the repo (`BUSL-1.1`, `agpl-3.0`, `AGPL-3.0`, `MIT`,
  `GPL-2.0-or-later`, plus `ISC` in `package.json`), third-party files whose
  original licence terms are not respected, and no `LICENSE.txt`. Omega's
  requirements: include the full text of each licence; claim copyright for code
  you wrote ("claiming authorship of the code is a necessary condition for
  granting any further rights to the code"); preserve third-party copyright
  notices; check the licences are mutually compatible; use only valid SPDX
  identifiers.
- `Toucan G3` — "Invalid SPDX license identifier."
- `Backed-Finance G2` / `Spectre G1` / `PrimeDAO-LBP G1` — "no copyright is
  claimed", "assert copyright". Absence is a finding, not just non-compliance.

**Checks:** `LICENSE` file present; SPDX identifier on every file and all valid;
one consistent licence, or documented exceptions; every vendored or adapted file
retains its upstream copyright notice; licences compatible with the project's own.

## Tests and CI

- **Coverage measured and adequate.** In roughly two thirds of reports
  (`Fragmint G2`, `Everbloom G1`, `Karpatkey G2`, `Society G2`,
  `Backed-Forwarder G1`, …). Name the uncovered paths that matter.
- **Tests actually pass.** `dxGovernance G3`, `dxDAO G3`, `Gnosis-Hashi G2`
  ("tests are failing, and test coverage is incomplete"), `Gnosis G1` ("tests in
  CI are failing").
- **Coverage is part of CI.** `EnterDAO G6`, `dxGovernance G10`.
- **The coverage tooling works.** `DXdao-staking G1` — "command for running test
  coverage is broken"; `EnterDAO G7` — "some tests fail under coverage";
  `EnterDAO G8`/`Y4` — "coverage is configured improperly"; `PrimeDAO-LBP G6` —
  "solcover settings file refers to non-existing contract"; `dxGovernance G9` —
  "command for running gas report does not work".
- **CI exists at all.** `Fragmint G1`, `DXdao-staking G6`, `Gnosis-Bridge G1`
  ("no working tests or continuous integration"), `Backed-Token-Bridge G2`.
- **Formal verification, if claimed, works.** `Euler G3` — "Certora formal
  verification is broken and lacks invariants."

## Compiler, linter, language version

- **Zero compiler warnings.** `Delphia G3`, `PrimeDAO-Seed G4`, `EnterDAO Y5`,
  `dxGovernance G12`, `Blindex E3`.
- **Zero linter warnings.** `Delphia G4`, `PrimeDAO-LBP G4` ("linter emits 25
  warnings"), `dxGovernance G13` ("eslint configuration is invalid").
- **Current Solidity, and a locked pragma.** `Delphia G5`, `Altitude G3`,
  `EnterDAO G5`, `Stackly G4`, `dxGovernance G5` ("do not use deprecated
  compiler versions"), `Altitude-parallel G2` (naming a specific target version
  to move to). Note `DXdao-Carrot G4` recommends moving *to* 0.8.x — the point
  is a deliberate, current, pinned version, not novelty.
- **It compiles in the documented configuration.** `Backed-Token-Bridge G3`.

## Dead and duplicated code

Consistently filed, and Omega gives the reason each time — not style, but the
risk of calling something half-implemented:

> Remove unused code and avoid leaving code which is not properly implemented to
> avoid mistakenly calling it in the future. — `Giza-Pendle W2`, `SQLW2`, `SQLR2`

- Unused variables, imports, functions, constants: `Altitude RD1`/`HM5`/`SG9`,
  `Karpatkey K22`, `Backed-wrapped-tokens G5`, `Blindex P3`–`P5`,
  `dxDAO VM9`, `Gnosis PDP1`, `Backed-Finance G3`.
- Duplication: `DXdao-staking D13` ("significant code duplication across
  multiple functions"), `Karpatkey K16`, `Altitude OM1`/`VV2`/`SBP3`,
  `Gnosis-Hashi H1`, `Giza-Pendle T2`, `Spectre B7`/`B8`.
- Leftovers: `dxGovernance P14` — "forgotten 'TODO' statement".

Duplication is not merely cosmetic in this archive: `Spectre I4` exists because
four findings in one contract apply verbatim to its near-identical twin, and
`Gnosis XDFB1` is a *high-impact* bug that exists only because two functions
were copies.

## Interfaces, visibility, events, errors

- **Contracts inherit their own interfaces.** `DXdao-staking G3`/`ID1`/`IF1`
  ("interface does not correspond with the implementation"), `Everbloom EDM2`/
  `EN7`, `Giza GS1`, `Backed BAFT1`, `Backed-Forwarder G3`. The point is that
  inheritance makes drift a compile error.
- **`external` over `public`** where not called internally. Ubiquitous —
  `Blindex G11`, `Karpatkey K19`, `Euler F3`, `Yieldnest P2`, `Altitude SA12`,
  and a dozen more.
- **Custom errors over revert strings.** `dxDAO G1`, `Altitude G5`,
  `Backed-wrapped-tokens G3`, `Giza GMC1`.
- **Index event parameters; emit on every state change.** `Everbloom G4`,
  `DXdao-Carrot G3`, `OpusEdu G5`, `EnterDAO G12`, `Blindex G12` ("emit events
  when parameters change"), `Karpatkey K11` ("initialize does not emit events
  for initial values"), `Altitude RIC1`, `Society SPB7`.
- **`immutable` / `constant` where applicable.** `Altitude G6`, `Delphia C11`,
  `Fragmint F3`, `Altitude MV3`, `dxDAO VM12`, `Blindex D1`.
- **Explicit visibility on state variables.** `Delphia C12`/`S4`,
  `PrimeDAO-Seed S12`, `Karpatkey K21`, `Backed-wrapped-tokens WC1`.

## Documentation and configuration

- **Docs match the code.** `EnterDAO G2`, `Backed-atomic-swap G3`,
  `Backed-Forwarder G2` ("documentation is not consistent with the code"),
  `Stackly D3` ("start time minimum requirement does not match documentation"),
  `Society CR2` ("permissions as described in docstring do not correspond with
  actual permissions"), `freename-snap G3`.
- **Specs complete and unambiguous.** `Delphia G2`, `Delphia C9`.
- **README instructions work.** `Blindex G6`, `Fragmint G5`, `EnterDAO R1`
  ("explain that you need a config file before giving the `npx hardhat compile`
  instruction"), `dxGovernance G6`/`G7` (deploy script fails on local network;
  deploy script is not tested).
- **Comments are not misleading.** `Spectre I1`, `EnterDAO S2`/`S3`,
  `Karpatkey K20`, `Blindex P13`.
- **One build system.** `lsdai G1`.

## Roles and configuration hygiene

Borderline between hygiene and trust modelling — Omega files them in General:

- `Blindex G3` — "many roles control critical parameters";
  `Spectre G8` / `Society G3` — "permission management is complex";
  `InverterDLF G3` — "improper roles management".
- `Blindex G7` — "no sanity checks, value limits, and change delays when
  changing parameters."
- `Giza D2` — "use a multisig with a non-trivial threshold for ownership";
  `Giza D1` — set the delegate to the multisig.
- `Giza O1` — "use OpenZeppelin `Ownable2Step` instead of your own
  implementation."
- `Blindex G8` — "use of precision for fractions is overly complex" (and
  `Blindex G10`, "use `Ownable` instead of defining your own" — prefer the
  audited standard implementation over a hand-rolled one).

---

## Output

Number them `G1…Gn` in the report's **General** section, before any per-file
section. Most land at `low` or `info`. Escalate to **medium** for: licensing
violations, unpinned or advisory-affected dependencies, and anything that makes
the delivered artifact non-reproducible.

Keep each finding one line of mechanism plus one line of recommendation. This
section should be fast to read and fast to fix; it is not where the interesting
work is, and padding it obscures the findings that matter.

---

## Checklist

- [ ] All dependencies pinned to released versions (not git refs, not ranges)
- [ ] Compiler pragma locked; version current and deliberate
- [ ] Dependency advisory audit run; advisories cited by ID
- [ ] Unused dependencies removed; no `draft-`/alpha code in production
- [ ] `LICENSE` present; SPDX on every file and valid; single consistent policy
- [ ] Copyright claimed for own code; upstream notices preserved on vendored code
- [ ] Licence compatibility checked
- [ ] Tests pass; coverage measured, adequate, wired into CI; tooling works
- [ ] CI exists and is green
- [ ] Zero compiler warnings; zero linter warnings
- [ ] Repo compiles in its documented configuration; one build system
- [ ] No unused variables, imports, functions, constants; no TODOs
- [ ] Duplication noted — and checked for divergence between the copies
- [ ] Contracts inherit their declared interfaces
- [ ] `external` vs `public`; explicit state-variable visibility; `immutable`/
      `constant` applied
- [ ] Custom errors; events emitted on every state change, parameters indexed
- [ ] Docs, comments and docstrings match the code; README instructions work
- [ ] Privileged roles enumerated; parameter setters have sanity bounds and
      delays; ownership is 2-step and multisig-held
