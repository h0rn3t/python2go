# Layered layout (http | worker)

**When:** project type is `http` or `worker`.  
**MUST-read** during **design** and **scaffold**. For `cli` | `tui` | `pipeline` | `lib`, use idiomatic Go (`cmd/`, `internal/<pkg>/`); do **not** invent these layered folders.

## Tree

Always create all six packages (stubs OK):

```text
cmd/<app>/main.go          # wire only
internal/
  handler/                 # Fiber / asynq (adapter entry)
  service/                 # business logic
  repository/              # DB, queues, external IO
  model/                   # domain types, domain errors
  schema/                  # DTOs (request/response, wire)
  config/                  # config load
```

## Dependencies (strict)

```text
handler → service → repository
handler ↔ schema          # boundary only (parse / validate / map)
service ↔ model
repository ↔ model
```

**Forbidden:** `handler` → `repository`. No import cycles.

Prefer mapping `schema` → `model` in `handler`. `service` / `repository` speak `model`, not wire DTOs.

## Mixed binaries

HTTP/worker + CLI: shared logic in layered `internal/`. API/worker `cmd/` wires `handler`. CLI `cmd/` may call `service` directly (thin main).

## Scaffold checklist

- [ ] six `internal/*` packages exist
- [ ] `cmd/<app>` only wires deps
- [ ] `AGENTS.md` Architecture = `layered`
- [ ] smoke builds a trivial path through handler → service (repository stub OK)

## Translate mapping

| Python | Go package |
|---|---|
| routes / views / tasks entry | `handler` |
| services / use-cases / business rules | `service` |
| ORM / DAOs / DB access / queue producers | `repository` |
| domain entities / domain errors | `model` |
| Pydantic / serializers / request-response models | `schema` |
| settings | `config` |
