# Attribution — Omega Skills

## What was derived, and from what

The skill files in this directory are **original prose written by Daoism
Systems**. They were derived by reading the public audit archive at
[github.com/OmegaAudits/audits](https://github.com/OmegaAudits/audits)
(repository state at commit `a9891e4`, 14 July 2026) and extracting the
recurring methodology and reasoning patterns.

**No Team Omega report text is reproduced here.** What is carried over is
factual and methodological: which questions Omega asks, which failure shapes
recur across engagements, how they structure scope and severity, and how they
run the preliminary→resolution→final loop. Finding identifiers (`Blindex P2`,
`Karpatkey K1`, …) are cited as *pointers into the primary source* so a reader
can verify each claim against the original PDF.

## Corpus

| | |
|---|---|
| Reports | 55 (2021-03 → 2026-07) |
| Findings catalogued | 834 |
| By severity | 2 critical · 46 high · 105 medium · 332 low · 349 info |
| Named auditors | Jelle, Ben (per the Methodology sections) |
| Report format | PDF; text extracted with `pypdf` for analysis only |

Six reports could not be auto-parsed into a finding index and were read
manually: `202106-API3DAO`, `202106-Prime DAO-v1`, `202107-PrimeDAO-v2`
(2021 reports predating Omega's `ID.` numbering scheme), and `202307-Backed`,
`202312-ToucanProtocol`, `202407-Inverter IssuanceToken` (summary-only reports
with no or trivial findings).

## Clients and codebases represented

Derived from the scope sections. This is the population the patterns
generalize over, and its shape matters when judging transferability:

- **Tokenized RWA / stablecoins** — Backed Finance (9 engagements), Blindex,
  lsdai, Inverter iTRY
- **Lending / yield vaults** — Altitude (5), Euler, Yieldnest, Karpatkey
- **Bridges / cross-chain** — Gnosis Bridge (4), Gnosis Hashi, Toucan Celo,
  Backed CCIP
- **DAO governance** — dxDAO (3), PrimeDAO (4), API3, EnterDAO
- **Off-chain agents & backends** — Giza (5 engagements; Python, not Solidity)
- **NFT / fractionalization** — Spectre, Fragmint, Everbloom, Society Protocol

Note the last two rows. Omega audits Python backends and frontends alongside
Solidity, and several skills here reflect that — see
`omega-external-data-trust`.

## Licensing

Team Omega's repository carries **no LICENSE file**; the PDF reports are their
copyrighted work. Nothing in this directory copies that expression.

The skill files here are © 2026 Daoism Systems, MIT, consistent with the
aggregation layer described in the repository root [LICENSE](../../LICENSE).
They are **not** covered by the `sources/<repo>/LICENSE` notices, which apply
only to the verbatim upstream mirrors.

If Team Omega would prefer different treatment of this derivation, the contact
listed in their repository README is `omegaaudits@protonmail.com`.
