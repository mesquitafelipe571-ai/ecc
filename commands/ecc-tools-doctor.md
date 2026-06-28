---
description: Diagnose drift, missing files, and inconsistencies in the ECC install-state for the current context.
argument-hint: "[--target claude|codex|opencode|cursor|all] [--json]"
---

# ECC Tools — Doctor

Diagnose the health of the current ECC installation. Compares install-state records against the actual file system to detect drift, missing managed files, and version mismatches. Wraps `node scripts/doctor.js` with an agent-friendly interface.

## What This Command Does

1. **Load install-state** — read `.ecc-install-state.json` for the current context
2. **Diff against file system** — detect added, removed, or modified managed files
3. **Check script health** — verify hook scripts are executable and syntactically valid
4. **Verify agent/skill integrity** — confirm frontmatter is well-formed for all installed items
5. **Report drift** with clear fix instructions for each issue

## Usage

```
/ecc-tools doctor [--target claude|codex|opencode|cursor|all] [--json]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--target` | auto-detected | Limit check to one harness install |
| `--json` | false | Emit machine-readable JSON |

## Step 1: Locate Install State

Find the install-state file:
1. `.claude/.ecc-install-state.json` (project-local install)
2. `~/.claude/.ecc-install-state.json` (global install)
3. Platform equivalent for other harnesses

If no install-state file is found, report `NOT INSTALLED` and suggest running `/ecc-tools setup`.

## Step 2: File System Diff

For every file recorded in install-state:

| Status | Meaning | Action |
|--------|---------|--------|
| `ok` | File exists and matches recorded checksum | No action needed |
| `missing` | File was deleted after install | Run `/ecc-tools repair` |
| `modified` | File was edited manually after install | Review change or repair |
| `extra` | File exists but not in install-state | Was it added manually? |

## Step 3: Hook Script Check

For each hook script referenced in `hooks.json`:
- Verify the file exists at the referenced path
- Check that the script has a valid shebang or is a `.js` file
- Run a syntax check (`node --check script.js` for JS hooks)
- Report scripts that would fail silently at runtime

## Step 4: Agent & Skill Frontmatter Check

For every installed `.md` agent or skill, parse frontmatter and verify:
- `name` field is present
- `description` field is present and non-empty
- No unknown frontmatter keys that could break the harness parser
- Model references (if any) point to valid model IDs

## Step 5: Version Check

Compare the installed ECC version (from `.ecc-install-state.json`) against the current repo `VERSION` file. Report if the install is outdated and suggest `/ecc-tools setup` to upgrade.

## Step 6: Output Report

```
ECC Doctor Report
=================
Context:  project (.claude/)
Version:  installed 1.9.2 / current 2.0.0  ← OUTDATED

File System Drift
─────────────────
  MISSING   .claude/agents/tdd-guide.md
  MODIFIED  .claude/rules/typescript/security.md  (manual edit detected)
  OK        .claude/agents/code-reviewer.md
  OK        .claude/hooks/pre-commit.sh

Hook Scripts
────────────
  OK        .claude/hooks/pre-commit.sh
  BROKEN    .claude/hooks/session-end.sh → syntax error on line 23

Frontmatter
───────────
  OK        42 agents
  INVALID   .claude/agents/custom.md → missing 'description' field

Summary
───────
  Issues: 1 BROKEN, 1 MISSING, 1 INVALID, 1 MODIFIED, 1 OUTDATED
  Run `/ecc-tools repair` to fix BROKEN, MISSING, and INVALID issues.
```

## Underlying Script

This command wraps the existing `scripts/doctor.js` utility:

```bash
node scripts/doctor.js --target <harness> [--json]
```

When run as a command, the agent adds contextual explanations and next-step recommendations that the raw script does not provide.

## Integration

- Run after `/ecc-tools setup` to confirm a clean install
- Run after manual edits to ECC files to check for regressions
- Run `/ecc-tools repair` to fix any issues found
- Run `/ecc-tools analyze` for a broader coverage gap report

## Arguments

$ARGUMENTS:
- optional `--target` harness name
- optional `--json` flag
