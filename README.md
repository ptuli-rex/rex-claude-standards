# Rex Claude Standards

Shared Claude Code conventions for the Rex tech team. Provides Zoho workflow rules and project references that every team member's Claude Code picks up automatically.

## Quick Start

### 1. Install (one-time)

```bash
git clone git@github.com:ptuli-rex/rex-claude-standards.git ~/.claude/rex-standards
cd ~/.claude/rex-standards && ./setup
```

### 2. Init a project (once per repo)

```bash
cd ~/projects/getdone/gd-api
~/.claude/rex-standards/bin/rex-team-init
```

This adds shared rules to your project's CLAUDE.md and nudges you to set up a release command if one doesn't exist.

### 3. Set up release command (optional, per workspace)

```bash
~/.claude/rex-standards/bin/rex-setup-release
```

Interactive script that creates a `/prod-release` command tailored to how your specific project deploys.

## What's Included

| Rule | Description |
|------|-------------|
| `rules/zoho-workflow.md` | Zoho task lifecycle, status IDs, task requirement for features |
| `rules/project-ids.md` | Zoho portal and project ID reference |

## How It Works

- `setup` symlinks `rules/` into `~/.claude/rules/rex/`
- `rex-team-init` adds `@import` lines to your project's CLAUDE.md pointing to the symlinked rules
- Claude Code loads these rules at session start — every team member gets the same conventions
- `rex-setup-release` creates a workspace-level `/prod-release` command with your deploy flow

## Updating

```bash
cd ~/.claude/rex-standards && git pull
```

Rules are symlinked, so pulling updates the standards for all projects immediately.

## Projects Covered

GetDone, ProofUp, BillRoute, IDCore, PayUp, JobCall, TenantPort, CredBuild, Platform
