---
name: py2go
description: >-
  Use when porting Python to Go, scaffolding a Go rewrite, migrating
  FastAPI/Flask/Django/Click/Typer/Celery/RQ to Go, or generating
  AGENTS.md/CLAUDE.md/MIGRATION.md for a Python→Go migration. Covers CLI,
  TUI, HTTP, pipeline, worker, and library rewrites with Fiber/pgx/GORM/slog
  defaults, golden parity tests, optional strangler or OpenAPI/proto-first.
compatibility: Cursor (Grok 4.5, Composer 2.5), Claude Code, similar agents.
metadata:
  author: python2go-fork
  version: "0.3.0"
  inspired-by: "https://github.com/vanducng/skills/tree/main/skills/py2go"
  upstream: "https://winder.ai/python-to-go-migration-with-claude-code/"
---

# py2go

Python → idiomatic Go. Phases: `discover → design → scaffold → translate → validate → cutover → cleanup`.

Reject AST transpilers. Prefer idiomatic Go over Python-shaped Go. Prefer stdlib; third-party only from the defaults table or a recorded escape hatch. **Target Go 1.26+.**

Canonical files live under `.claude/skills/py2go/`; Cursor uses `.cursor/skills/py2go` (symlink). Keep `AGENTS.md` ↔ `CLAUDE.md` in sync.

## Cursor models (Grok 4.5 / Composer 2.5)

| Role | Model bias | Do |
|---|---|---|
| Plan | Grok 4.5 (or any strong planner) | discover + design only; lock stack/API in `AGENTS.md` before code |
| Execute | Composer 2.5 | one `MIGRATION.md` module per turn; end every turn on a green check |
| Solo | either | still one phase gate at a time; never skip validate success conditions |

**Always encode a verifiable success condition** before editing (Composer was trained on test loops). Example: `go test -race ./internal/foo/...` green + checkbox ticked.

**Cadence:** one short status line before tools; no essays. Do not spawn subagents to “verify yourself.” Commit only if the user asks.

**Soft options** (say in chat): `--type cli|tui|http|pipeline|worker|lib|auto`, `--strangler`, `--spec-first FILE`, `--strict-tdd`, `--resume`, `--dry-run`. No path → current workspace. Claude Code: `/py2go …` with the same options.

## MUST-read map

| When | Read |
|---|---|
| Any Python→Go construct while translating | [references/translation-rules.md](references/translation-rules.md) |
| Scaffold / translate / validate | [references/go126-baseline.md](references/go126-baseline.md) |
| Design (emit artifacts) | [references/artifacts.md](references/artifacts.md) |

Do not proceed past the gate without reading the file. Soft “optional” reads are forbidden for these three.

## Phase gates

Finish a phase only when its **Done when** holds. Then stop or ask before the next phase unless the user said “full migration.”

### 1. Discover

Two-pass. Pass 1: answer each item separately. Pass 2: synthesize into `notes/`.

1. Entry points (`__main__`, console_scripts, ASGI/WSGI, Celery apps)
2. Deps (`requirements*.txt` / `pyproject.toml` → keep / replace / drop)
3. Project type (playbooks table)
4. IO boundaries (DB, queues, HTTP clients, files, env)
5. Concurrency (async / threads / processes)
6. Golden fixtures worth keeping
7. Load-bearing NumPy/SciPy? → **STOP** (gRPC-wrap Python; do not translate)

**Done when:** `notes/` lists type, IO map, dep decisions, and STOP/go.

### 2. Design

Emit from [references/artifacts.md](references/artifacts.md):

- `AGENTS.md` + `CLAUDE.md` (same body)
- `MIGRATION.md` (ordered modules + checkboxes)

Spec-first: generate stubs (`oapi-codegen` / `buf`) first; `MIGRATION.md` tracks handlers, not Python files 1:1.

**Done when:** both agent files + `MIGRATION.md` exist; public API (`pkg/` vs `internal/`) decided in one sentence.

### 3. Scaffold

`go mod init` → `go mod edit -go=1.26` (init may default older). Layout, `golangci-lint` (enable `modernize` if available), CI, Makefile: `build` / `test` / `lint` / `smoke`. Smoke must compile and run a trivial path.

**Done when:** `go 1.26` in `go.mod` and smoke green.

### 4. Translate

Per module in `MIGRATION.md` order:

1. Read Python + fixtures
2. **MUST** read translation-rules + go126-baseline
3. Test first (behavior/golden), then impl — `--strict-tdd`: test mtime before impl in the commit
4. Success condition: `go fix ./... && go build && go vet && go test -race ./...` (+ lint)
5. Tick checkbox; commit only if asked

**Done when:** all translate checkboxes ticked with green tests.

### 5. Validate

Parity on **real** production-shaped fixtures (path in `AGENTS.md`). Synthetic-only ≠ done. Integration tests for IO. Dead-code sweep. Re-run `go fix ./...`; reject `interface{}`, `math/rand` v1, `sort.Slice`, pre-`AsType` dances, etc.

**Done when:** real-fixture path recorded and parity/integration green.

### 6. Cutover

- Default: hard cut after parity
- `--strangler`: gateway → shadow Go → compare → ramp %; Python fallback until stable; ramp plan in `MIGRATION.md`

**Done when:** traffic plan executed or hard-cut deployed (as requested).

### 7. Cleanup

Orphan Python, `go mod tidy`, remove dual-run scaffolding, short retro in `notes/`.

**Done when:** tidy clean; dual-run gone.

## Playbooks

| Type | Detected from | Default stack | Escape hatch |
|---|---|---|---|
| CLI | Click/Typer, `console_scripts` | Cobra + Viper + lipgloss | `flag` + env for tiny tools |
| TUI | Textual / Rich.live / prompt_toolkit | Bubble Tea + Bubbles + Lipgloss | — (Elm rewrite) |
| HTTP | FastAPI/Flask/Django/Starlette | **Fiber** + validator | chi / Echo / Gin / `net/http` if team chose |
| Pipeline | pandas/polars/Airflow/Prefect/Dagster | streaming `[]T` + channels | qframe; **STOP** if NumPy/SciPy load-bearing |
| Worker | Celery/RQ/Dramatiq/Arq/Taskiq | asynq (Redis) or river (Postgres) | watermill / Temporal if event/workflow heavy |
| Library | `__init__.py` exports | `pkg/` + forced public-API decision | — |

## Hard guardrails

1. No 1:1 syntax port — Go must not mirror Python structure across >50% of lines.
2. No `lib/pq` — `pgx/v5`.
3. No `golang/mock` — `go.uber.org/mock`.
4. GORM default — sqlc/Ent/sqlx only as recorded hatch in `AGENTS.md`.
5. No `panic` for business errors — return `error`.
6. No blind `internal/` — design states public API explicitly.
7. No synthetic-only validation — real-data fixture path required.
8. NumPy/SciPy load-bearing — STOP; gRPC-wrap Python.
9. Stdlib-first — see translation-rules.
10. Go 1.26+ idioms — see go126-baseline; no older equivalents by habit.

## Defaults / escape hatches

Record every hatch in `AGENTS.md` / `CLAUDE.md`.

| Concern | Default | Escape hatch |
|---|---|---|
| HTTP | Fiber (`gofiber/fiber/v2`) | chi, Echo, Gin, `net/http` |
| Postgres | pgx/v5 | — (lib/pq forbidden) |
| DB access | GORM | sqlc / Ent / sqlx |
| Migrations | golang-migrate | Atlas |
| Config | Viper (with Cobra) | `envconfig` / `os.Getenv` |
| Logger | `log/slog` | zap/zerolog if house standard |
| Validation | go-playground/validator | ozzo-validation |
| Mocking | go.uber.org/mock | — |
| OpenAPI | oapi-codegen | — |
| Queue | asynq or river | watermill / Temporal |
| Cache | ristretto or map+mutex | redis client |
| Auth | golang-jwt/v5 + argon2 | paseto / scs |
| Observability | OpenTelemetry + Prometheus | — |
| Build/release | goreleaser + ko | multi-stage Dockerfile |

## Extending

1. Add detect signal + playbook row
2. Add rows to [translation-rules.md](references/translation-rules.md)
3. Keep SKILL.md lean — table rows over new files
