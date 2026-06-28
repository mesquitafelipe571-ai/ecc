---
description: Full security and quality audit of the ECC install — agent prompts, hook scripts, MCP configs, supply chain, and rule completeness.
argument-hint: "[--min-severity low|medium|high|critical] [--surface agents|hooks|mcp|rules|all] [--json]"
---

# ECC Tools — Audit

Run a comprehensive security and quality audit across all ECC surfaces. Covers agent prompt injection risks, hook script safety, MCP supply-chain exposure, and rule completeness. Produces a prioritized remediation checklist.

## What This Command Does

1. **Agent prompt audit** — detect injection vectors, missing defenses, overly broad tool access
2. **Hook script audit** — flag unsafe shell patterns, secrets in scripts, missing input validation
3. **MCP supply-chain audit** — unpinned packages, unauthenticated transports, excessive permissions
4. **Rule completeness audit** — languages used but not covered, stale rule content
5. **Secret scan** — hardcoded credentials, tokens, or keys anywhere in ECC config files
6. **Output prioritized remediation plan**

## Usage

```
/ecc-tools audit [--min-severity low|medium|high|critical] [--surface agents|hooks|mcp|rules|all] [--json]
```

| Flag | Default | Description |
|------|---------|-------------|
| `--min-severity` | `medium` | Minimum severity level to report |
| `--surface` | `all` | Limit audit to one surface type |
| `--json` | false | Emit machine-readable JSON |

## Step 1: Agent Prompt Audit

For each agent in `agents/*.md`, check:

| Check | Severity | What to Look For |
|-------|----------|-----------------|
| No prompt-injection defense | HIGH | Agent processes external content without validation |
| Overly broad tool access | MEDIUM | `tools: all` without justification |
| Missing output sanitization | MEDIUM | Agent outputs user-supplied content verbatim |
| No identity/persona guardrails | LOW | Agent can be convinced to change role |
| Hardcoded secrets in prompts | CRITICAL | API keys, tokens, passwords in frontmatter or body |

## Step 2: Hook Script Audit

For each script referenced in `hooks/hooks.json`:

| Check | Severity | What to Look For |
|-------|----------|-----------------|
| Unquoted variable expansions | HIGH | `$VAR` without quotes → injection risk |
| `eval` or dynamic execution | HIGH | `eval "$input"`, `$(...)` with user data |
| Secrets in environment setup | CRITICAL | Hardcoded keys in hook scripts |
| Missing file-existence checks | MEDIUM | Script fails silently if file not found |
| Unrestricted `rm -rf` patterns | HIGH | Destructive commands without guards |
| No error handling | LOW | Script exits 0 on failure masking errors |

## Step 3: MCP Supply-Chain Audit

For each server in `mcp-configs/mcp-servers.json`:

| Check | Severity | What to Look For |
|-------|----------|-----------------|
| Unpinned `npx` package | HIGH | `npx some-package` without `@version` |
| HTTP transport (not HTTPS) | HIGH | Unencrypted MCP server connections |
| `filesystem` scope too broad | MEDIUM | Access to `/` or `~` instead of project root |
| Unauthenticated remote server | MEDIUM | No API key or auth token configured |
| Deprecated or archived package | MEDIUM | Package no longer maintained |

## Step 4: Rule Completeness Audit

- Detect all languages used in the repo (file extensions)
- Compare against `rules/` directories
- Flag missing languages as **UNCOVERED**
- Check that existing rules were updated in the last 6 months (stale if not)

## Step 5: Secret Scan

Scan all ECC config files (`.md`, `.json`, `.yaml`, `.js`) for patterns matching:
- `sk-...` (OpenAI keys)
- `ghp_...` (GitHub tokens)
- `AKIA...` (AWS keys)
- Any `password =` / `secret =` / `token =` assignments
- Base64-encoded strings > 40 chars in config context

**If secrets are found**: STOP and report immediately. Do not continue audit. Instruct the user to rotate the exposed credential before proceeding.

## Step 6: Output Report

```
ECC Security & Quality Audit
==============================
Surfaces scanned: agents (42), hooks (12), mcp (5), rules (9)
Findings: 2 CRITICAL, 3 HIGH, 5 MEDIUM, 8 LOW

CRITICAL
────────
  [hooks/scripts/deploy.sh:14] Hardcoded AWS_SECRET_KEY detected
  [agents/custom-agent.md:3]   API token in frontmatter

HIGH
────
  [hooks/hooks.json → pre-bash.sh] Unquoted $BASH_COMMAND expansion
  [mcp-configs/mcp-servers.json]   npx @modelcontextprotocol/server-filesystem (unpinned)
  [agents/data-agent.md]           No prompt-injection defense for external content

Remediation Order
─────────────────
  1. Rotate exposed AWS key immediately
  2. Remove token from agent frontmatter, use env var
  3. Quote $BASH_COMMAND in pre-bash.sh
  4. Pin filesystem MCP server to @1.2.3
  5. Add untrusted-input guard to data-agent.md
```

## Integration

- Run `/ecc-tools analyze` first to understand the install surface
- Run `/ecc-tools repair` after fixing issues to validate corrections
- Use `agents/security-reviewer` for deeper per-file review
- Use `/security-scan` for project code (not ECC config) security scanning

## Arguments

$ARGUMENTS:
- optional `--min-severity` level
- optional `--surface` filter
- optional `--json` flag
