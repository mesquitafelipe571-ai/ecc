# ECC Tools — Custom Fork Additions

This file is an index of the **fork-specific additions** layered on top of the upstream
[`affaan-m/ECC`](https://github.com/affaan-m/ECC) project.

The six entries listed below do **not** exist in the upstream repository. They were
hand-authored in this fork (`mesquitafelipe571-ai/ecc`) and must be preserved when
syncing from upstream. Do not overwrite or delete them during a merge or rebase against
`affaan-m/ECC`.

---

## Custom Additions

| # | Name | File path | One-line description | Underlying script |
|---|------|-----------|----------------------|-------------------|
| 1 | `ecc-tools-analyze` | `commands/ecc-tools-analyze.md` | Scan the ECC install (agents, skills, commands, hooks, rules, MCP configs) and report coverage gaps, unused entries, and health metrics. | — |
| 2 | `ecc-tools-audit` | `commands/ecc-tools-audit.md` | Full security & quality audit: prompt-injection risks, hook script safety, MCP supply-chain exposure, secret scanning, and rule completeness. | — |
| 3 | `ecc-tools-doctor` | `commands/ecc-tools-doctor.md` | Diagnose drift, missing files, and inconsistencies in the ECC install-state for the current context. | `scripts/doctor.js` |
| 4 | `ecc-tools-repair` | `commands/ecc-tools-repair.md` | Rebuild missing or broken ECC-managed files from install-state, fix hook scripts, and restore the install to a consistent state. | `scripts/repair.js` |
| 5 | `ecc-tools-setup` | `commands/ecc-tools-setup.md` | Interactive setup wizard: detect the project stack, select harness targets, and install agents/skills/rules/hooks. | `scripts/install-apply.js`, `scripts/install-plan.js` |
| 6 | `orcid-database` *(skill)* | `skills/scientific-db-orcid/SKILL.md` | ORCID iD researcher profile lookup, works/publication retrieval, employment/funding data, and Public API workflows. Not a slash command — activated by referencing the skill in an agent prompt or via `/skill orcid-database`. | — |

---

## Usage Snippets

### `/ecc-tools analyze`

```
/ecc-tools analyze [--target claude|codex|opencode|cursor|all] [--json]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--target` | auto-detected | Limit analysis to one harness install |
| `--json` | false | Emit machine-readable JSON instead of Markdown |

### `/ecc-tools audit`

```
/ecc-tools audit [--min-severity low|medium|high|critical] [--surface agents|hooks|mcp|rules|all] [--json]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--min-severity` | `medium` | Minimum severity level to report |
| `--surface` | `all` | Limit audit to one surface type |
| `--json` | false | Emit machine-readable JSON |

### `/ecc-tools doctor`

```
/ecc-tools doctor [--target claude|codex|opencode|cursor|all] [--json]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--target` | auto-detected | Limit check to one harness install |
| `--json` | false | Emit machine-readable JSON |

### `/ecc-tools repair`

```
/ecc-tools repair [--target claude|codex|opencode|cursor|all] [--dry-run] [--json]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--target` | auto-detected | Limit repair to one harness install |
| `--dry-run` | false | Show what would be repaired without writing files |
| `--json` | false | Emit machine-readable JSON output |

### `/ecc-tools setup`

```
/ecc-tools setup [--target claude|codex|opencode|cursor|all] [--dry-run] [--minimal]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--target` | auto-detected | Which harness(es) to install into |
| `--dry-run` | false | Show what would be installed without writing files |
| `--minimal` | false | Install only core surfaces (agents + common rules), skip optional skills |

---

## Underlying Scripts

Each `ecc-tools-*` command that wraps a Node.js script calls it directly via `node`:

| Command | Script | Confirmed present |
|---------|--------|-------------------|
| `ecc-tools-doctor` | `scripts/doctor.js` | ✅ |
| `ecc-tools-repair` | `scripts/repair.js` | ✅ |
| `ecc-tools-setup` | `scripts/install-apply.js` | ✅ |
| `ecc-tools-setup` | `scripts/install-plan.js` | ✅ |

`ecc-tools-analyze` and `ecc-tools-audit` perform their analysis through agent
reasoning and file-system inspection rather than a dedicated script; they have no
underlying `.js` wrapper.

The ORCID skill (`skills/scientific-db-orcid/SKILL.md`) is a pure knowledge/workflow
document and does not wrap any local script.

---

## Upstream Sync Note

When pulling updates from `affaan-m/ECC` (the upstream origin), the following files
are **not present upstream** and must be kept:

```
commands/ecc-tools-analyze.md
commands/ecc-tools-audit.md
commands/ecc-tools-doctor.md
commands/ecc-tools-repair.md
commands/ecc-tools-setup.md
commands/ECC-TOOLS-README.md   ← this file
skills/scientific-db-orcid/SKILL.md
```

If a merge or rebase reports these as conflicts or deletions, always keep the fork
version. A simple safeguard is to check `git status` after the merge and restore any
of the above that were removed before committing.
