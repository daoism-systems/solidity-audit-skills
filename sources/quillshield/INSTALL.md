# Install QuillShield Skills in Claude Code

## One-Line Install

Clone the repo and run the install script:

```bash
git clone https://github.com/quillai-network/qs_skills.git
cd qs_skills && bash install.sh
```

## Manual Install

### Step 1: Create the commands directory

```bash
mkdir -p ~/.claude/commands
```

### Step 2: Copy all skills

```bash
cp plugins/behavioral-state-analysis/skills/behavioral-state-analysis/SKILL.md ~/.claude/commands/behavioral-state-analysis.md
cp plugins/reentrancy-pattern-analysis/skills/reentrancy-pattern-analysis/SKILL.md ~/.claude/commands/reentrancy-pattern-analysis.md
cp plugins/proxy-upgrade-safety/skills/proxy-upgrade-safety/SKILL.md ~/.claude/commands/proxy-upgrade-safety.md
cp plugins/input-arithmetic-safety/skills/input-arithmetic-safety/SKILL.md ~/.claude/commands/input-arithmetic-safety.md
cp plugins/dos-griefing-analysis/skills/dos-griefing-analysis/SKILL.md ~/.claude/commands/dos-griefing-analysis.md
```

> This library's mirror carries 5 plugins; the other 5 upstream plugins were
> merged into `../symbiosis/` and dispatch from there.

### Step 3: Verify

```bash
ls ~/.claude/commands/
```

You should see 5 `.md` files.

## Usage

In any Claude Code session, type `/` and select a skill:

| Command | What It Audits |
|---------|---------------|
| `/behavioral-state-analysis` | Full multi-dimensional security audit |
| `/reentrancy-pattern-analysis` | All reentrancy variants (classic, cross-function, read-only) |
| `/proxy-upgrade-safety` | Storage collisions, uninitialized impls, selector clashes |
| `/input-arithmetic-safety` | Input validation, precision loss, ERC4626 inflation |
| `/dos-griefing-analysis` | Unbounded loops, gas griefing, force-feeding |

### Example

```
> /behavioral-state-analysis
> Audit this contract: [paste contract or file path]
```

## Uninstall

```bash
rm ~/.claude/commands/behavioral-state-analysis.md
rm ~/.claude/commands/reentrancy-pattern-analysis.md
rm ~/.claude/commands/proxy-upgrade-safety.md
rm ~/.claude/commands/input-arithmetic-safety.md
rm ~/.claude/commands/dos-griefing-analysis.md
```
