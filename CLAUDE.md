# CLAUDE.md — AI Assistant Guide for claude-code Repository

This file provides context for AI assistants (including Claude Code itself) working in this repository. It describes what the repo is, how it's organized, and the conventions to follow.

---

## What This Repository Is

This is the **official public repository for Claude Code** — Anthropic's agentic coding tool that runs in your terminal. The repository serves several purposes:

1. **Issue tracker** — Bug reports, feature requests, and model-behavior issues from users
2. **Documentation hub** — Settings examples, hook examples, and plugin documentation
3. **Plugin library** — Official Claude Code plugins published under `plugins/`
4. **Automation scripts** — GitHub Actions workflows and TypeScript/Bash scripts for issue management

> The Claude Code binary itself is **closed source** and distributed via npm/curl/Homebrew/winget. This repo does **not** contain the application source code.

---

## Repository Structure

```
claude-code/
├── .claude/                    # Claude Code project configuration
│   └── commands/               # Custom slash commands for this repo
│       ├── commit-push-pr.md   # /commit-push-pr — commit, push, create PR
│       ├── dedupe.md           # /dedupe — find duplicate GitHub issues
│       └── triage-issue.md     # /triage-issue — analyze and label issues
├── .claude-plugin/
│   └── marketplace.json        # Plugin marketplace manifest for this repo
├── .devcontainer/              # VS Code Dev Container configuration
│   ├── Dockerfile
│   ├── devcontainer.json
│   └── init-firewall.sh
├── .github/
│   ├── ISSUE_TEMPLATE/         # Structured issue templates (bug, feature, docs, model behavior)
│   └── workflows/              # GitHub Actions CI/CD workflows
├── examples/
│   ├── hooks/                  # Example hook scripts (Python)
│   └── settings/               # Example settings JSON files for org deployments
├── plugins/                    # Official Claude Code plugins
│   ├── agent-sdk-dev/
│   ├── claude-opus-4-5-migration/
│   ├── code-review/
│   ├── commit-commands/
│   ├── explanatory-output-style/
│   ├── feature-dev/
│   ├── frontend-design/
│   ├── hookify/
│   ├── learning-output-style/
│   ├── plugin-dev/
│   ├── pr-review-toolkit/
│   └── ralph-wiggum/
├── scripts/                    # Automation scripts (TypeScript + Bash)
│   ├── auto-close-duplicates.ts
│   ├── backfill-duplicate-comments.ts
│   ├── comment-on-duplicates.sh
│   ├── edit-issue-labels.sh
│   ├── gh.sh                   # Restricted gh CLI wrapper (read-only subset)
│   ├── issue-lifecycle.ts      # Shared lifecycle label definitions
│   ├── lifecycle-comment.ts
│   └── sweep.ts                # Stale issue sweep script
├── CHANGELOG.md                # Claude Code release notes
├── LICENSE.md
├── README.md
└── SECURITY.md
```

---

## Plugin Structure

Every plugin under `plugins/` follows this standard layout:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata (name, version, author, description)
├── commands/                # Slash commands (.md files with YAML frontmatter)
├── agents/                  # Specialized agent definitions
├── skills/                  # Agent skills (auto-invoked capabilities)
├── hooks/                   # Hook scripts (PreToolUse, PostToolUse, Stop, etc.)
├── .mcp.json                # MCP server configuration (optional)
└── README.md                # Plugin documentation
```

### Slash Command Format

Slash command files use YAML frontmatter followed by the prompt body:

```markdown
---
allowed-tools: Bash(git add:*), Bash(git commit:*)
description: Short description shown in the command picker
---

## Context
- Current git status: !`git status`

## Your task
Instructions for Claude...
```

- `allowed-tools` restricts which tools the command can use (security sandbox)
- Dynamic context can be injected with the `!` prefix on inline shell commands

---

## GitHub Automation Workflows

### Issue Lifecycle (`.github/workflows/`)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `claude-issue-triage.yml` | Issue opened / comment created | Runs `/triage-issue` via Claude Code Action to apply labels |
| `claude-dedupe-issues.yml` | Issue opened | Runs `/dedupe` to find and comment on duplicate issues |
| `auto-close-duplicates.yml` | Scheduled | Closes issues flagged as duplicates after 3 days of no objection |
| `sweep.yml` | Scheduled (10:00, 22:00 UTC) | Marks stale issues and closes expired lifecycle-labeled issues |
| `issue-lifecycle-comment.yml` | Label added | Posts nudge comments when lifecycle labels are applied |
| `claude.yml` | `@claude` mentions | Responds to `@claude` mentions in issues/PRs/comments |
| `lock-closed-issues.yml` | Issue/PR closed | Locks closed items after a period |
| `non-write-users-check.yml` | PR modifying `.github/` | Warns when `allowed_non_write_users` is added to workflows |

### Issue Lifecycle Labels

Defined in `scripts/issue-lifecycle.ts` — this is the **single source of truth**:

| Label | Timeout | Auto-close Reason |
|-------|---------|-------------------|
| `invalid` | 3 days | Not about Claude Code |
| `needs-repro` | 7 days | Missing reproduction steps |
| `needs-info` | 7 days | Missing environment/version info |
| `stale` | 14 days | Inactive |
| `autoclose` | 14 days | Inactive |

Issues with ≥10 upvotes (`STALE_UPVOTE_THRESHOLD`) are exempt from stale/autoclose.

### Automation Scripts

Scripts run with **Bun** (not Node.js):

```bash
bun run scripts/sweep.ts        # Enforce lifecycle timeouts
bun run scripts/auto-close-duplicates.ts
```

Required environment variables:
- `GITHUB_TOKEN` — for all GitHub API calls
- `GITHUB_REPOSITORY_OWNER` / `GITHUB_REPOSITORY_NAME` — for sweep.ts
- `GH_REPO` or `GITHUB_REPOSITORY` — for `scripts/gh.sh`

### `scripts/gh.sh` — Restricted GitHub CLI Wrapper

This wrapper limits Claude Code's access to a safe subset of `gh` commands:

**Allowed commands:**
- `./scripts/gh.sh issue view <number> [--comments]`
- `./scripts/gh.sh issue list [--state open] [--limit N]`
- `./scripts/gh.sh search issues "query" [--limit N]`
- `./scripts/gh.sh label list [--limit N]`

**Allowed flags only:** `--comments`, `--state`, `--limit`, `--label`

> Always use `./scripts/gh.sh` instead of raw `gh` in automation contexts — it enforces repo scoping and prevents unintended operations.

---

## Claude Code Action Integration

The repository uses `anthropics/claude-code-action@v1` in several workflows. Key patterns:

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    github_token: ${{ secrets.GITHUB_TOKEN }}
    prompt: "/triage-issue REPO: ${{ github.repository }} ISSUE_NUMBER: ${{ github.event.issue.number }} EVENT: ${{ github.event_name }}"
    claude_args: "--model claude-opus-4-6"
    allowed_non_write_users: "*"   # ⚠️ Security-sensitive — use with care
```

**Security note:** Adding `allowed_non_write_users` triggers an AppSec review comment. Do not add new permissions to workflows using this setting without careful review.

---

## Dev Container

The `.devcontainer/` configuration provides a sandboxed development environment:

- **Base image:** Node.js-based with zsh, Claude Code, git-delta
- **VS Code extensions:** Claude Code, ESLint, Prettier, GitLens
- **Network:** Firewall initialized via `init-firewall.sh` (requires `NET_ADMIN`, `NET_RAW` capabilities)
- **Volumes:** Persistent bash history and Claude config across rebuilds
- **Remote user:** `node`
- **Workspace:** Mounted at `/workspace`

---

## Available Plugins

| Plugin | Key Commands/Skills | Purpose |
|--------|---------------------|---------|
| `agent-sdk-dev` | `/new-sdk-app` | Set up Claude Agent SDK projects |
| `claude-opus-4-5-migration` | skill: `claude-opus-4-5-migration` | Migrate model strings from older to Opus 4.5 |
| `code-review` | `/code-review` | Automated PR review with 5 parallel agents |
| `commit-commands` | `/commit`, `/commit-push-pr`, `/clean_gone` | Git workflow automation |
| `explanatory-output-style` | Hook: SessionStart | Educational insights about implementation choices |
| `feature-dev` | `/feature-dev` | 7-phase guided feature development workflow |
| `frontend-design` | skill: `frontend-design` | Production-grade, distinctive UI generation |
| `hookify` | `/hookify`, `/hookify:list`, `/hookify:configure` | Create and manage custom Claude Code hooks |
| `learning-output-style` | Hook: SessionStart | Interactive learning mode |
| `plugin-dev` | `/plugin-dev:create-plugin` | 8-phase plugin creation workflow |
| `pr-review-toolkit` | `/pr-review-toolkit:review-pr` | Specialized PR review agents |
| `ralph-wiggum` | `/ralph-loop`, `/cancel-ralph` | Autonomous iterative development loops |
| `security-guidance` | Hook: PreToolUse | Security warnings on file edits |

---

## Settings Examples (`examples/settings/`)

Three reference settings files for enterprise/org deployments:

| File | Description |
|------|-------------|
| `settings-lax.json` | Disables `--dangerously-skip-permissions` and plugin marketplaces |
| `settings-strict.json` | Full lockdown: blocks user hooks, denies web tools, bash requires approval |
| `settings-bash-sandbox.json` | Bash must run inside sandbox; user permission rules blocked |

These are applied via the Claude Code settings hierarchy (enterprise > project > user).

---

## Hook Examples (`examples/hooks/`)

`bash_command_validator_example.py` — A `PreToolUse` hook that:
- Intercepts Bash tool calls
- Validates commands against configurable regex rules
- Redirects `grep` to `rg` (ripgrep) and `find -name` to ripgrep equivalents
- Returns exit code `0` (allow), `1` (show stderr to user), or `2` (block and show to Claude)

---

## Key Conventions

### When Working on Issues
- Use `./scripts/gh.sh` (not raw `gh`) for all GitHub API reads
- Never post comments on issues from triage workflows — only modify labels
- Apply lifecycle labels conservatively; false positives are worse than missing labels
- Only apply `needs-repro`/`needs-info` to **bug** issues, never to questions or enhancements

### When Working on Plugins
- Every plugin needs `.claude-plugin/plugin.json` and a `README.md`
- Slash commands must have `allowed-tools` frontmatter to restrict tool access
- Hook exit codes: `0` = allow, `1` = show stderr to user only, `2` = block + show to Claude
- Rules files for the hookify plugin go in `.claude/hookify.<name>.local.md`

### When Modifying Workflows
- Never add `allowed_non_write_users: "*"` without AppSec review
- Scripts run with Bun — not Node.js or npx
- Always pin action versions (e.g., `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`)

### Git Workflow
- Default branch is `main`; active development happens on `master` and feature branches
- Branch names for AI-assisted work follow the pattern: `claude/<task-id>`
- Use `git push -u origin <branch-name>` when pushing new branches

### Code Style
- TypeScript scripts use Bun-compatible syntax (top-level `await`, `#!/usr/bin/env bun`)
- Bash scripts use `set -euo pipefail` and quote all variables
- No build step required — scripts run directly via Bun

---

## Current Version

The latest Claude Code release is documented in `CHANGELOG.md`. As of the most recent entry: **v2.1.63**, which introduced `/simplify` and `/batch` bundled commands, HTTP hooks, and various memory leak fixes.

---

## Reporting Issues

- Use `/bug` inside Claude Code to file issues directly
- File GitHub issues at: https://github.com/anthropics/claude-code/issues
- Use the structured issue templates in `.github/ISSUE_TEMPLATE/`
- Join the community at: https://anthropic.com/discord
