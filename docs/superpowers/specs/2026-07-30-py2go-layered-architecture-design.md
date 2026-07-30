# py2go layered architecture — design

Date: 2026-07-30  
Status: implemented  
Scope: update py2go skill so HTTP/worker Go rewrites use classic layered layout

## Goal

Agents scaffolding or translating Python → Go via py2go must produce a **layered** `internal/` layout for `http` and `worker` project types, and keep **idiomatic Go** layout for other types.

## Decisions (locked)

| Decision | Choice |
|---|---|
| Layer style | Classic global 3-layer packages (not feature-sliced, not hexagonal) |
| Domain package name | `model/` (not `domain/`) |
| DTO package name | `schema/` |
| When layered applies | `http` and `worker` only |
| Other types | Idiomatic Go (`cmd/`, `internal/<pkg>/`, no forced layers) |
| Empty packages | Always create all six layered packages (stubs OK) |
| Dependencies | Strict: `handler → service → repository` |
| Forbidden | `handler → repository` |
| Schema boundary | `schema` used at handler edge; service/repository use `model` |
| Encoding in skill | Approach 1: new `references/layered.md` + hooks in SKILL.md / artifacts.md / README |

## Layout (http | worker)

```text
cmd/<app>/main.go          # wire only
internal/
  handler/                 # Fiber / asynq (or other adapter) entry
  service/                 # business logic
  repository/              # DB, queues, external IO
  model/                   # domain types, domain errors
  schema/                  # DTOs (request/response, wire formats)
  config/                  # config load
```

### Dependency rules

- `handler` → `service` → `repository`
- `handler` may import `schema` (parse/validate/map at boundary)
- `service` and `repository` import `model`
- `service` must not depend on `schema` except rare edge mapping (prefer map in handler)
- `handler` must not import `repository`
- No import cycles

### Mixed binaries

If the project has both an HTTP/worker binary and a CLI:

- Shared logic lives in layered `internal/` packages
- API/worker `cmd/` wires handlers
- CLI `cmd/` may call `service` directly (thin main), without inventing a full CLI handler package unless needed

## Idiomatic path (cli | tui | pipeline | lib)

No forced `handler/service/repository/schema` tree. Use normal Go module layout (`cmd/`, `internal/…`, optional `pkg/` for public API). Public API decision still recorded in `AGENTS.md`.

## Skill file changes

### New: `references/layered.md`

MUST-read during **design** and **scaffold** when project type is `http` or `worker`. Contents:

- Package tree above
- Dependency rules and forbidden imports
- Stub policy (all packages always present)
- Short note: other types → idiomatic (pointer only; do not duplicate playbooks)

### Edit: `SKILL.md`

1. MUST-read map: add row for design/scaffold (http|worker) → `references/layered.md`
2. Playbooks table: http and worker default stack cells note **layered layout**
3. Hard guardrails: add rule — for `http|worker`, enforce layered packages + strict deps; for other types do not invent layered folders

### Edit: `references/artifacts.md`

Add to `AGENTS.md` / `CLAUDE.md` template:

```text
## Architecture
- Layout: layered | idiomatic
- If layered: handler → service → repository; model + schema + config always present
```

### Edit: `README.md`

One-line note under Defaults: HTTP/worker use layered `internal/` (`handler`/`service`/`repository`/`model`/`schema`/`config`).

### Out of scope

- No change to Fiber/GORM/pgx defaults
- No hexagonal ports/adapters
- No feature-sliced packages as default
- No automated import linter in the skill (document rules; agents follow them)
- No commit of `.superpowers/brainstorm/` session artifacts

## Success criteria

1. Agent running design for an HTTP Python app writes `Architecture: layered` in AGENTS.md and scaffolds the six packages
2. Agent running design for a Click CLI does **not** create empty handler/repository/schema trees
3. Translate phase for HTTP maps Python routes → `handler`, domain → `model`/`service`, persistence → `repository`, Pydantic wire types → `schema`

## Open follow-ups (not this change)

- Optional `depguard`/golangci rule to enforce import direction (escape hatch later)
