---
description: Rebuild missing or broken ECC-managed files from install-state, fix hook scripts, and restore the install to a consistent state.
argument-hint: "[--target claude|codex|opencode|cursor|all] [--dry-run] [--json]"
---

# ECC Tools — Repair

Restore the ECC installation to a consistent state by rebuilding any missing, broken, or drift-detected files recorded in install-state. Wraps `node scripts/repair.js` with contextual guidance, dry-run preview, and post-repair validation.

## What This Command Does

1. **Run doctor check** — identify all issues before making changes
2. **Confirm repair scope** — show exactly what will be changed
3. **Rebuild managed files** — restore missing or broken files from ECC source
4. **Fix hook scripts** — correct permissions and syntax errors
5. **Repair frontmatter** — add missing required fields to agents/skills
6. **Re-run doctor** — verify all issues are resolved after repair

## Usage

```
/ecc-tools repair [--target claude|codex|opencode|cursor|all] [--dry-run] [--json]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--target` | auto-detected | Limit repair to one harness install |
| `--dry-run` | false | Show what would be repaired without writing files |
| `--json` | false | Emit machine-readable JSON output |

## Step 1: Pre-Repair Doctor Scan

Always run `/ecc-tools doctor` first (automatically) to build the list of issues. Do not proceed if no install-state is found — prompt the user to run `/ecc-tools setup` instead.

## Step 2: Categorize Issues

Group detected issues by repair strategy:

| Issue Type | Repair Strategy |
|------------|----------------|
| `MISSING` managed file | Restore from ECC source tree |
| `BROKEN` hook script | Fix syntax error or restore from source |
| `INVALID` frontmatter | Add missing required fields |
| `MODIFIED` file | Ask user — preserve edit or restore original? |
| `OUTDATED` version | Suggest `/ecc-tools setup` for full upgrade |

**For `MODIFIED` files**: always ask the user before overwriting. Show a diff of the manual changes and offer three options:
- Keep the manual edit (skip this file)
- Restore the original (discard edit)
- Merge (open file for manual review)

## Step 3: Dry-Run Preview

When `--dry-run` is active, show the complete repair plan without executing:

```
ECC Repair — Dry Run
====================
Target: .claude/ (claude harness)

Planned repairs (4 issues):
  RESTORE   .claude/agents/tdd-guide.md          (MISSING)
  RESTORE   .claude/hooks/session-end.sh          (BROKEN — syntax error)
  PATCH     .claude/agents/custom.md              (INVALID — add 'description')
  SKIP      .claude/rules/typescript/security.md  (MODIFIED — user edit, skipping)

No files written (--dry-run active).
Run without --dry-run to apply repairs.
```

## Step 4: Execute Repairs

Run repairs using the underlying script:

```bash
node scripts/repair.js --target <harness> [--dry-run] [--json]
```

Show a live progress line for each file repaired:
```
  ✔ Restored  .claude/agents/tdd-guide.md
  ✔ Restored  .claude/hooks/session-end.sh
  ✔ Patched   .claude/agents/custom.md
  — Skipped   .claude/rules/typescript/security.md (user edit preserved)
```

## Step 5: Post-Repair Validation

After all repairs, automatically re-run `/ecc-tools doctor` and confirm:
- Zero `MISSING` files remain
- Zero `BROKEN` hooks remain
- Zero `INVALID` frontmatter entries remain
- Report any issues that could not be repaired automatically

## Step 6: Output Summary

```
ECC Repair Complete
===================
Target:  .claude/ (claude harness)

Repaired
────────
  ✔  .claude/agents/tdd-guide.md        (restored from source)
  ✔  .claude/hooks/session-end.sh       (restored from source)
  ✔  .claude/agents/custom.md           (added missing 'description' field)

Skipped
───────
  —  .claude/rules/typescript/security.md  (manual edit preserved)

Post-repair doctor: ✅ CLEAN (1 skip noted above)

Next steps:
  • Review the skipped file manually: .claude/rules/typescript/security.md
  • Run `/ecc-tools analyze` to check for any remaining coverage gaps
```

## When Repair Cannot Fix an Issue

Some issues require manual intervention:

| Situation | Guidance |
|-----------|---------|
| Source file not in ECC tree | Run `/ecc-tools setup` to re-install from latest ECC |
| Corrupt install-state JSON | Delete `.ecc-install-state.json` and re-run setup |
| Outdated ECC version | Pull latest ECC and re-run `/ecc-tools setup` |
| Hook requires missing dependency | Install the dependency manually, then re-run repair |

## Underlying Script

This command wraps the existing `scripts/repair.js` utility:

```bash
node scripts/repair.js --target <harness> [--dry-run] [--json]
```

The agent wrapper adds pre-repair doctor scanning, user confirmation for modified files, and post-repair validation that the raw script does not provide.

## Integration

- Always run `/ecc-tools doctor` first to understand scope
- Use `/ecc-tools analyze` after repair to check coverage
- Use `/ecc-tools audit` after repair to verify security posture
- For full reinstall: `/ecc-tools setup --target <harness>`

## Arguments

$ARGUMENTS:
- optional `--target` harness name
- optional `--dry-run` flag
- optional `--json` flag
