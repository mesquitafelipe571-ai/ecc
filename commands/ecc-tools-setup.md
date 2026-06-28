---
description: Interactive setup wizard for ECC — detect the project stack, select harness targets, install agents/skills/rules/hooks, and validate the result.
argument-hint: "[--target claude|codex|opencode|cursor|all] [--dry-run] [--minimal]"
---

# ECC Tools — Setup

Bootstrap or reconfigure an ECC install for the current project. Detects the stack, recommends the right surfaces, and installs everything needed in one pass.

## What This Command Does

1. **Detect project stack** — language, framework, package manager, CI provider
2. **Recommend surfaces** — agents, skills, rules, and MCP configs relevant to this project
3. **Confirm with user** — show the install plan and wait for approval
4. **Run installation** — copy or link all selected surfaces
5. **Validate** — verify the install is consistent and report any issues

## Usage

```
/ecc-tools setup [--target claude|codex|opencode|cursor|all] [--dry-run] [--minimal]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--target` | auto-detected | Which harness(es) to install into |
| `--dry-run` | false | Show what would be installed without writing files |
| `--minimal` | false | Install only core surfaces (agents + common rules), skip optional skills |

## Step 1: Stack Detection

Scan the project root for stack indicators:

| File | Inferred Stack |
|------|---------------|
| `package.json` + `tsconfig.json` | TypeScript / Node.js |
| `package.json` + React dep | React |
| `package.json` + Next.js dep | Next.js |
| `pyproject.toml` / `requirements.txt` | Python |
| `Cargo.toml` | Rust |
| `go.mod` | Go |
| `pom.xml` / `build.gradle` | Java / Kotlin |
| `pubspec.yaml` | Flutter / Dart |
| `.github/workflows/` | GitHub Actions CI |

Report detected stacks before proceeding.

## Step 2: Harness Detection

Identify installed AI harnesses by checking for:
- `.claude/` → Claude Code
- `.codex/` → Codex
- `.opencode/` → OpenCode
- `.cursor/rules/` → Cursor
- `--target` flag overrides auto-detection

## Step 3: Build Install Plan

For each detected stack, map to recommended surfaces:

```
Stack: TypeScript + React + GitHub Actions
─────────────────────────────────────────
Agents:   typescript-reviewer, react-reviewer, code-reviewer,
          tdd-guide, security-reviewer, build-error-resolver
Skills:   react-patterns, react-testing, typescript patterns,
          git-workflow, github-ops, tdd-workflow
Rules:    typescript/, react/, web/
Hooks:    pre-commit quality gate, session persistence
MCP:      context7 (docs lookup)
```

Show the plan in a table and **WAIT for user confirmation** before writing any files.

## Step 4: Install

Execute installation using `node scripts/install-apply.js`:

```bash
node scripts/install-apply.js --target <harness> --surfaces <list>
```

Show a progress indicator for each surface installed.

## Step 5: Post-Install Validation

Run `/ecc-tools doctor` automatically after install and report:
- ✅ All files installed correctly
- ⚠️ Any drift or missing files
- ❌ Any broken references

## Dry-Run Output Example

```
ECC Setup — Dry Run
===================
Detected stack:  TypeScript, React, GitHub Actions
Target harness:  claude (.claude/)

Would install:
  agents/typescript-reviewer.md  →  .claude/agents/
  agents/react-reviewer.md       →  .claude/agents/
  agents/tdd-guide.md            →  .claude/agents/
  rules/typescript/              →  .claude/rules/
  rules/react/                   →  .claude/rules/
  skills/tdd-workflow/           →  .claude/skills/
  hooks/hooks.json (merged)      →  .claude/hooks.json

No files written (--dry-run active). Run without --dry-run to apply.
```

## Integration

- Use `/ecc-tools analyze` before setup to understand what's already installed
- Use `/ecc-tools doctor` after setup to verify the result
- Use `/ecc-tools repair` if setup is interrupted or partially applied

## Arguments

$ARGUMENTS:
- optional `--target` harness name(s)
- optional `--dry-run` flag
- optional `--minimal` flag
