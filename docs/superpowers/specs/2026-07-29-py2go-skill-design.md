# py2go skill redesign (0.2.0)

Date: 2026-07-29

## Goals

- Universal Cursor skill for Python → Go migrations
- Opinionated defaults + documented escape hatches
- Lean: only `SKILL.md` + `references/translation-rules.md`
- English
- Stdlib-first translation rules
- HTTP default: **Fiber** (not Gin)
- DB access default: **GORM** (sqlc/Ent/sqlx as escape hatch)

## Non-goals

- Full playbook / strangler / spec-first reference files
- vanducng ecosystem (`gostack`, `vd:*`)
- Hard flag-locked stack

## Files

- `.claude/skills/py2go/` — canonical (Claude Code `/py2go`)
- `.cursor/skills/py2go` → symlink to `.claude/skills/py2go` (Cursor)
- `SKILL.md` — orchestrator (7 phases, playbooks, guardrails) + CC frontmatter (`user-invocable`, `argument-hint`, `allowed-tools`)
- `references/translation-rules.md` — Python→Go map

## Artifacts produced by the skill

- `CLAUDE.md` + `AGENTS.md` — same stack/escape/public-API rules (keep in sync)
- `MIGRATION.md` — ordered checkbox map
