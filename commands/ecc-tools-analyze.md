---
description: Analyze the ECC installation — agents, skills, commands, hooks, rules, and MCP configs — and report coverage gaps, unused entries, and health metrics.
argument-hint: "[--target claude|codex|opencode|cursor|all] [--json]"
---

# ECC Tools — Analyze

Deep-scan the current ECC install and produce a structured health report covering every surface: agents, skills, commands, hooks, rules, and MCP configurations.

## What This Command Does

1. **Inventory installed surfaces** — count agents, skills, commands, hooks, rules, and MCP servers
2. **Detect coverage gaps** — surfaces referenced by commands or hooks that are not installed
3. **Flag dead entries** — installed items that are never referenced
4. **Measure rule coverage** — languages with code in the repo that lack matching `rules/` entries
5. **Report MCP health** — servers listed in `mcp-servers.json` that are missing or misconfigured
6. **Output a prioritized action list**

## Usage

```
/ecc-tools analyze [--target claude|codex|opencode|cursor|all] [--json]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--target` | auto-detected | Limit analysis to one harness install |
| `--json` | false | Emit machine-readable JSON instead of Markdown |

## Step 1: Detect Install Root

Locate the ECC install root by checking (in order):
1. `$ECC_ROOT` environment variable
2. `.claude/` present in current directory
3. `~/.claude/` (global install)

Report which root was used at the top of the output.

## Step 2: Inventory All Surfaces

Run a file-system scan and count:

| Surface | Path | Count |
|---------|------|-------|
| Agents | `agents/*.md` | — |
| Skills | `skills/*/SKILL.md` | — |
| Commands | `commands/*.md` | — |
| Hooks | `hooks/hooks.json` entries | — |
| Rules | `rules/**/*.md` | — |
| MCP servers | `mcp-configs/mcp-servers.json` entries | — |

## Step 3: Cross-Reference

- Agents referenced in commands but not present → **MISSING**
- Skills referenced in commands but not present → **MISSING**
- Agents/skills installed but never referenced → **UNUSED** (low priority)
- Hook scripts that point to non-existent files → **BROKEN**

## Step 4: Rule Coverage Check

Detect languages used in the repo (by file extension) and verify each has a matching `rules/<lang>/` directory. Report languages without rules as **UNCOVERED**.

## Step 5: MCP Health

For each server in `mcp-configs/mcp-servers.json`:
- Verify the binary or npx package is resolvable
- Check that required environment variables are documented
- Flag servers with `npx` without a pinned version as **UNPINNED**

## Step 6: Output Report

```
ECC Analysis Report
===================
Root:     ~/.claude/
Harness:  claude

Surfaces
--------
  Agents:   42 installed, 3 missing, 1 unused
  Skills:   87 installed, 0 missing, 4 unused
  Commands: 38 installed
  Hooks:    12 active, 1 broken
  Rules:    9 languages covered, 2 uncovered (go, rust)
  MCP:      5 servers, 1 unpinned

Action Items (by priority)
--------------------------
  [CRITICAL] hooks/hooks.json → scripts/missing-hook.js not found
  [HIGH]     agents/planner.md missing — referenced by /plan command
  [MEDIUM]   rules/go/ missing — Go files detected in repo
  [LOW]      skills/old-skill/ installed but never referenced
```

## Integration

- Run `/ecc-tools doctor` to validate install-state drift after analysis
- Run `/ecc-tools repair` to fix broken or missing entries
- Run `/ecc-tools audit` for a full security + quality deep-dive

## Arguments

$ARGUMENTS:
- optional `--target` harness name
- optional `--json` flag
