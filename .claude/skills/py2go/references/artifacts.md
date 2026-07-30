# Artifact templates

Fill these during **design**. Keep `AGENTS.md` and `CLAUDE.md` identical in substance (copy or symlink). Do not invent extra sections.

## `AGENTS.md` / `CLAUDE.md`

```
# Go rewrite — agent rules

## Stack
- Go: 1.26+
- Project type: <cli|tui|http|pipeline|worker|lib>
- Defaults: <from playbook>
- Escape hatches (only if used): <none | list + why>

## Public API
- Exported: pkg/... | none (all internal/)
- Decision: <one sentence>

## Architecture
- Layout: layered | idiomatic
- If layered (http|worker): handler → service → repository; model + schema + config always present
- If idiomatic (cli|tui|pipeline|lib): no forced layered folders

## Non-negotiables
- Idiomatic Go, not Python-shaped
- No lib/pq; no golang/mock; no panic for business errors
- Parity fixtures: <path to real production-shaped data>
- Stdlib-first; third-party only from defaults or recorded hatch

## Validate command
go fix ./... && go build ./... && go vet ./... && go test -race ./...
```

## `MIGRATION.md`

```
# Migration map

Options: type=<...> strangler=<yes|no> spec-first=<path|no> strict-tdd=<yes|no>

## Status
- [ ] discover
- [ ] design (AGENTS.md + CLAUDE.md + this file)
- [ ] scaffold (go 1.26, smoke green)
- [ ] translate (modules below)
- [ ] validate (real fixtures)
- [ ] cutover
- [ ] cleanup

## Modules (order)
- [ ] <py path> → <go path> — success: go test -race ./path/...
- [ ] …

## Cutover
- Mode: hard | strangler
- Ramp plan (if strangler): …
```
