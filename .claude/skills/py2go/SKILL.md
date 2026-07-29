---
name: py2go
description: >-
  Migrate Python projects to idiomatic Go end-to-end. Six project-type playbooks
  (CLI, TUI, HTTP, pipeline, worker, library) with opinionated defaults (Fiber,
  pgx, GORM, slog) plus documented escape hatches. Module-by-module rewrite with
  golden-file parity tests; optional strangler cutover or OpenAPI/proto-first
  regeneration. Use when porting Python to Go, scaffolding a Go rewrite, or
  generating CLAUDE.md/AGENTS.md/MIGRATION.md for an AI-driven migration.
  Works in Claude Code (/py2go) and Cursor.
user-invocable: true
argument-hint: "[python-project-path] [--type cli|tui|http|pipeline|worker|lib|auto] [--strangler] [--spec-first FILE] [--strict-tdd] [--resume] [--dry-run]"
license: MIT
compatibility: Claude Code, Cursor, and similar AI coding agents.
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Bash(python:*) Bash(python3:*) Agent AskUserQuestion
metadata:
  author: python2go-fork
  version: "0.2.1"
  inspired-by: "https://github.com/vanducng/skills/tree/main/skills/py2go"
  upstream: "https://winder.ai/python-to-go-migration-with-claude-code/"
---

# py2go

End-to-end Python → Go migration. Discover → design → scaffold → translate → validate → cutover → cleanup.

**Hosts:** Claude Code (project skill under `.claude/skills/py2go`, invoke `/py2go`) and Cursor (same files via `.cursor/skills/py2go` symlink). One source of truth.

Opinionated **defaults** with **escape hatches** recorded in `CLAUDE.md` / `AGENTS.md`. Prefer idiomatic Go over Python-shaped Go. Reject AST transpilers (Grumpy, etc.). Prefer stdlib; reach for third-party libs only when the default table says so or the escape hatch is documented.

### Claude Code invocation

```
/py2go path/to/python-project
/py2go path/to/python-project --type http --strict-tdd
/py2go . --strangler --resume
```

Treat `$ARGUMENTS` / the rest of the user message as the soft options listed below. If no path is given, use the current workspace.

## When to load which reference

| Task | Open |
|---|---|
| Python construct → Go idiom | [references/translation-rules.md](references/translation-rules.md) |

Strangler / spec-first: short notes under Cutover and Design below — no separate playbooks in this lean skill.

## Soft options (say them in chat)

- `--type cli|tui|http|pipeline|worker|lib|auto` — force project type
- `--strangler` — gradual cutover instead of hard cut
- `--spec-first FILE` — drive from OpenAPI/proto instead of Python source shape
- `--strict-tdd` — Go test file must precede impl in each translate commit
- `--resume` / `--dry-run` — continue from `MIGRATION.md` / plan only

## The 7 phases

```
discover → design → scaffold → translate → validate → cutover → cleanup
```

### 1. Discover

Two-pass. First pass: answer each prompt separately (do not merge early). Second pass: synthesize into `notes/`.

1. Entry points (`__main__`, console_scripts, ASGI/WSGI, Celery apps)
2. Dependency surface (`requirements*.txt` / `pyproject.toml` — classify: keep / replace / drop)
3. Project type signals (see playbooks table)
4. IO boundaries (DB, queues, HTTP clients, files, env)
5. Concurrency model (async, threads, processes)
6. Test & fixture assets worth keeping as golden inputs
7. Load-bearing NumPy/SciPy? → **STOP** (gRPC-wrap Python; do not translate)

### 2. Design

Emit (same content in both agent rule files — do not drift):

- **`CLAUDE.md`** — for Claude Code: stack defaults, escape hatches, public-API decision (`pkg/` vs `internal/`), translation non-negotiables
- **`AGENTS.md`** — same body for Cursor / other agents (copy or symlink to `CLAUDE.md`)
- **`MIGRATION.md`** — ordered file/module map with checkboxes

If only one of `CLAUDE.md` / `AGENTS.md` already exists, update that file and mirror to the other. Prefer keeping both in sync.

**Spec-first:** if an OpenAPI/proto file is the source of truth, generate server stubs first (`oapi-codegen` / `buf`), then fill handlers to match Python behavior — `MIGRATION.md` tracks handlers, not Python files 1:1.

### 3. Scaffold

`go mod init`, layout, `golangci-lint`, CI, Makefile with `build` / `test` / `lint` / `smoke`. Smoke must compile and run a trivial path.

### 4. Translate

Per module in `MIGRATION.md` order:

1. Read Python source + existing tests/fixtures
2. Write Go test first (behavior / golden), then impl
3. `go build && go vet && go test -race ./...` + lint
4. Check off `MIGRATION.md`; commit that module

Under `--strict-tdd`: test file mtime must precede impl in the commit.

### 5. Validate

Golden-file / parity checks on **real** production-shaped fixtures (path recorded in `CLAUDE.md` / `AGENTS.md`). Synthetic-only is not enough to mark validate done. Integration tests for IO boundaries. Dead-code sweep of unused Python ports.

### 6. Cutover

- **Default:** hard cutover once parity passes
- **`--strangler`:** put a gateway/proxy in front; shadow traffic to Go; compare responses; ramp %; keep Python as fallback until stable. Document the ramp plan in `MIGRATION.md`

### 7. Cleanup

Orphan Python packages, unused Go deps (`go mod tidy`), remove dual-run scaffolding, short retro in `notes/`.

## Project-type playbooks

| Type | Detected from | Default stack | Escape hatch |
|---|---|---|---|
| CLI | Click/Typer, `console_scripts` | Cobra + Viper + lipgloss | `flag` + env only for tiny tools |
| TUI | Textual / Rich.live / prompt_toolkit | Bubble Tea + Bubbles + Lipgloss | — (Elm rewrite expected) |
| HTTP | FastAPI/Flask/Django/Starlette | **Fiber** + validator | chi / Echo / Gin / `net/http` if team already chose |
| Pipeline | pandas/polars/Airflow/Prefect/Dagster | streaming `[]T` + channels | qframe; **STOP** if NumPy/SciPy load-bearing |
| Worker | Celery/RQ/Dramatiq/Arq/Taskiq | asynq (Redis) or river (Postgres) | watermill / Temporal when event/workflow heavy |
| Library | `__init__.py` exports | `pkg/` + forced public-API decision | — |

## Hard guardrails

1. **No 1:1 syntax port** — Go must not mirror Python signatures/structure across >50% of lines.
2. **No `lib/pq`** — use `pgx/v5`.
3. **No `golang/mock`** — use `go.uber.org/mock`.
4. **GORM default** — sqlc/Ent/sqlx only if recorded as escape hatch in `CLAUDE.md` / `AGENTS.md`.
5. **No `panic` for business errors** — return `error`.
6. **No blind `internal/`** — design must state public API explicitly.
7. **No synthetic-only validation** — real-data fixture path required.
8. **NumPy/SciPy load-bearing** — STOP; recommend gRPC wrap of Python.
9. **Stdlib-first** — see translation rules; optional libs only when noted.

## Defaults and escape hatches

| Concern | Default | Escape hatch (record in CLAUDE.md / AGENTS.md) |
|---|---|---|
| HTTP framework | Fiber (`gofiber/fiber/v2`) | chi, Echo, Gin, stdlib `net/http` |
| Postgres driver | pgx/v5 | — (lib/pq forbidden) |
| DB access | GORM | sqlc / Ent / sqlx |
| Migrations | golang-migrate | Atlas |
| Config | Viper (with Cobra) | `envconfig` / plain `os.Getenv` for simple services |
| Logger | `log/slog` | zap/zerolog only if existing house standard |
| Validation | go-playground/validator | ozzo-validation for complex rules |
| Mocking | go.uber.org/mock | — |
| OpenAPI codegen | oapi-codegen | — |
| Queue | asynq or river | watermill / Temporal |
| Cache | ristretto or map+mutex | redis client; optional loader libs |
| Auth | golang-jwt/v5 + argon2 | paseto / scs sessions |
| Observability | OpenTelemetry + Prometheus | — |
| Build/release | goreleaser + ko | classic Dockerfile multi-stage |

## Adding a playbook later

1. Add detect signal + row to the playbooks table
2. Add diverging rows to [translation-rules.md](references/translation-rules.md)
3. Keep this skill lean — prefer table rows over new reference files
