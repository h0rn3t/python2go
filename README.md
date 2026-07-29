# py2go

Cursor / Claude Code skill: migrate Python projects to idiomatic Go (Fiber, pgx, GORM, slog by default).

Canonical path: [`.claude/skills/py2go/`](.claude/skills/py2go/)  
Cursor symlink: [`.cursor/skills/py2go`](.cursor/skills/py2go) → same files.

## Install (recommended) — `npx skills`

Uses the [skills](https://github.com/vercel-labs/skills) CLI. Works for Claude Code, Cursor, and many other agents.

```bash
# List what this repo publishes
npx skills add h0rn3t/python2go -l

# Project install (Claude Code + Cursor)
npx skills add h0rn3t/python2go -s py2go -a claude-code -a cursor -y

# Global install (all your projects)
npx skills add h0rn3t/python2go -s py2go -g -a claude-code -a cursor -y

# Local path (this checkout, before/without GitHub push)
npx skills add /absolute/path/to/python2go -s py2go -a claude-code -a cursor -y

# Install for every detected agent
npx skills add h0rn3t/python2go -s py2go --all
```

Useful flags: `-g` global, `-a` agents, `-s` skill name, `-y` skip prompts, `--copy` copy instead of symlink.

Verify:

```bash
npx skills ls -a claude-code -a cursor
```

Update later:

```bash
npx skills update py2go
```

## Install — Claude Code (manual)

### Project

```bash
mkdir -p /path/to/your-project/.claude/skills
cp -R .claude/skills/py2go /path/to/your-project/.claude/skills/
```

Invoke:

```text
/py2go path/to/python-project
/py2go . --type http --strict-tdd
```

### Personal

```bash
mkdir -p ~/.claude/skills
cp -R .claude/skills/py2go ~/.claude/skills/
# or: ln -s "$(pwd)/.claude/skills/py2go" ~/.claude/skills/py2go
```

## Install — Cursor (manual)

### Project

```bash
mkdir -p /path/to/your-project/.cursor/skills
ln -s /absolute/path/to/python2go/.claude/skills/py2go \
  /path/to/your-project/.cursor/skills/py2go
# or cp -R instead of ln -s
```

### Personal

```bash
mkdir -p ~/.cursor/skills
ln -s /absolute/path/to/python2go/.claude/skills/py2go ~/.cursor/skills/py2go
```

In chat: ask to run **py2go** on a Python path.

## Dual layout in this repo

```bash
# canonical
.claude/skills/py2go/
# Cursor
.cursor/skills/py2go -> ../../.claude/skills/py2go
```

`npx skills add` discovers the skill from `.claude/skills/` automatically.

## What it produces

| Artifact | Purpose |
|---|---|
| `CLAUDE.md` | Agent rules (Claude Code) |
| `AGENTS.md` | Same rules (Cursor / others) |
| `MIGRATION.md` | Ordered module checklist |
| `notes/` | Discover synthesis |

## Defaults

HTTP **Fiber**, DB **GORM** + **pgx**, logger **slog**. Escape hatches go in `CLAUDE.md` / `AGENTS.md`. Tuned for Cursor with Grok 4.5 (plan) and Composer 2.5 (execute). Refs: [translation-rules](.claude/skills/py2go/references/translation-rules.md), [go126-baseline](.claude/skills/py2go/references/go126-baseline.md), [artifacts](.claude/skills/py2go/references/artifacts.md).

