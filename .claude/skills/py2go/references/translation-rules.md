# Python → Go Translation Rules

**Stdlib-first.** Defaults below are the canonical translation. Optional third-party libs appear only as escape hatches — record them in `CLAUDE.md` / `AGENTS.md` when used. Prefer idiomatic Go over Python-shaped Go. **Target Go 1.26+** — apply [go126-baseline.md](go126-baseline.md) whenever it fits (`new(expr)`, `errors.AsType`, `slices`/`maps`, `range n`, `sync.WaitGroup.Go`, etc.).

## How to read this file

| Column | Meaning |
|---|---|
| Python | Source construct |
| Go | Canonical translation (use this unless escape hatch is recorded) |
| Notes | Stdlib vs optional import path / when to deviate |

Library names are real Go import paths.

## Language constructs

| Python | Go | Notes |
|---|---|---|
| `class Foo(Base):` (inheritance) | `type Foo struct{}` + interface; embed `*Base` only when truly is-a | stdlib |
| `class Foo:` (data + methods) | `type Foo struct{}` + methods or free funcs; interface + concrete impl | stdlib |
| `@dataclass` | `type Foo struct{}` + `func NewFoo(...) (*Foo, error)` | stdlib |
| `@dataclass` with defaults | constructor + functional options; non-zero field pointers via `new(expr)` | stdlib — Go 1.26+ `new(30)` not a `ptr` helper |
| `Enum` | `type X int; const ( A X = iota; ... )` + `String()`; Marshal if serialized | stdlib |
| `Optional[T]` (nullable field) | `*T` (init with `new(expr)` when non-zero) | stdlib; avoid Option monads by default |
| `Optional[T]` (return value) | `(T, error)` for failure; `(T, bool)` for lookup | stdlib |
| `Union[A, B]` | interface + type switch | stdlib; dual-success cases are rare — redesign |
| `dict[str, Any]` (at boundary) | `map[string]any` at IO; decode to concrete struct ASAP | stdlib — `any`, not `interface{}` |
| list comprehension `[f(x) for x in xs]` | `for` + `append`; or `slices` helpers when clearer | stdlib; optional `samber/lo` only for 2+ chained transforms |
| list comprehension w/ filter | `for` + `if` + `append` | stdlib |
| `sum(xs)` | `for _, v := range xs { sum += v }` | stdlib |
| `sorted(xs)` | `slices.Sort(xs)` | stdlib — not `sort.Slice` |
| `xs.sort(key=fn)` | `slices.SortFunc(xs, less)` | stdlib |
| `x in xs` | `slices.Contains(xs, x)` | stdlib — **not** `lo.Contains` |
| `set(xs)` (dedup) | loop into `map[T]struct{}` then keys; or sort+compact | stdlib |
| `set` semantics | `map[T]struct{}` | stdlib |
| `groupby(xs, key=fn)` | `map[K][]V` built in a loop | stdlib |
| `zip(a, b)` | indexed `for` over `min(len(a), len(b))` | stdlib — builtin `min` |
| `chunk(xs, n)` | slice windows in a loop | stdlib |
| `range(n)` | `for i := range n` | stdlib — Go 1.22+ |

## Concurrency

| Python | Go | Notes |
|---|---|---|
| `async def f()` | `func f(ctx context.Context) (T, error)` | stdlib — pass `context.Context` |
| `await f()` | direct call; concurrency via goroutine + channel | stdlib |
| `asyncio.gather(*tasks)` | `errgroup.Group` + `g.Go` + `g.Wait`; simple fan-out: `sync.WaitGroup.Go` | `golang.org/x/sync/errgroup` / stdlib 1.25+ |
| bounded gather | `errgroup` + `semaphore.Weighted` | `golang.org/x/sync/semaphore` |
| `asyncio.Queue` | `chan T` (buffered); writer closes | stdlib |
| `asyncio.Lock` / `Semaphore` | `sync.Mutex` / `semaphore.Weighted` | stdlib / x/sync |
| `asyncio.Event` | `chan struct{}` close-to-signal | stdlib |
| `ThreadPoolExecutor` | goroutines + errgroup; optional `panjf2000/ants/v2` | stdlib first |
| reactive streams | channels + errgroup | avoid reactive libs by default |

## Errors

| Python | Go | Notes |
|---|---|---|
| `raise ValueError(msg)` | `return fmt.Errorf("...")` or `fmt.Errorf("%w: ...", err)` | stdlib |
| `try / except ValueError` | `if err != nil { ... }` immediately | stdlib |
| `try / except / finally` | `defer cleanup()` + `if err != nil` | stdlib |
| `raise X from Y` | `fmt.Errorf("...: %w", y)` | stdlib |
| custom exception class | `var ErrX = errors.New("...")` + `errors.Is`; typed: `errors.AsType[*MyErr](err)` | stdlib — Go 1.26+ Prefer `AsType` over `var x; errors.As` |
| `logger.error("..." + str(id))` | `slog.Error("op failed", "user_id", id)` | low-cardinality attrs |
| panic recovery in goroutine | `defer func() { if r := recover(); r != nil { ... } }()` at entry | stdlib |
| `assert x, msg` | `if !x { return fmt.Errorf("...") }` | never assert-libs in prod |

## Logging

| Python | Go | Notes |
|---|---|---|
| `logging.getLogger(__name__)` | `slog.Default()` or package logger | stdlib |
| `logger.info("...")` | `slog.Info("...")` | stdlib |
| `logger.info("...", extra={"k": v})` | `slog.Info("...", "k", v)` | stdlib |
| `loguru.logger.bind(user=u)` | `logger.With("user", u)` | stdlib |
| multi-handler / Sentry / sampling | optional slog middleware libs | only if needed; record escape |
| HTTP request logging | Fiber middleware logging request/status/latency | Fiber default; or custom slog middleware |

## HTTP server

| Python | Go | Notes |
|---|---|---|
| `FastAPI()` | `fiber.New()` | **default**; escape: chi / Echo / Gin / `net/http` |
| `@app.get("/path")` | `app.Get("/path", handler)` | Fiber |
| Pydantic `BaseModel` body | struct + `json` tags + `go-playground/validator` | parse/bind in handler |
| Dependency Injection (Depends) | constructor injection at composition root | stdlib; see DI section |
| Middleware | `app.Use(...)` | Fiber |
| CORS | `fiber/middleware/cors` | Fiber contrib |
| Sessions | store in cookie/header via middleware or `alexedwards/scs` | escape hatch |
| Static files | `app.Static("/static", "./assets")` | Fiber |
| File upload | `c.FormFile` / `c.SaveFile` | Fiber |
| WebSocket | `coder/websocket` or `gorilla/websocket` | — |
| Rate limit | Fiber limiter middleware or `ulule/limiter` | — |
| Health check | simple `app.Get("/healthz", ...)` | — |
| OpenAPI codegen | `oapi-codegen` (adapt to Fiber handlers) | or hand-written routes |

## Backend frameworks (Django/Flask)

| Python | Go | Notes |
|---|---|---|
| Django (full stack) | Fiber + GORM + redesign — no "Go Django" | admin/ORM/templates have no peer |
| Django ORM | GORM (default); Ent for huge schemas; sqlc for SQL-first | — |
| DRF serializers | struct + tags + validator + JSON | — |
| Django admin | strip or rebuild separately | missing by default |
| Flask route | `app.Get(...)` (Fiber) | — |
| Flask-SQLAlchemy | GORM (default) or sqlx | — |
| Jinja2 | `html/template` or `a-h/templ` | stdlib first |

## Database access

| Python | Go | Notes |
|---|---|---|
| `psycopg*` / `asyncpg` | `github.com/jackc/pgx/v5` | required default |
| `sqlite3` | `modernc.org/sqlite` (no cgo) or `mattn/go-sqlite3` | — |
| MySQL | `github.com/go-sql-driver/mysql` | — |
| `pymongo` | `go.mongodb.org/mongo-driver/v2` | — |
| ClickHouse | `github.com/ClickHouse/clickhouse-go/v2` | — |
| SQLAlchemy | **GORM** (`gorm.io/gorm` + `gorm.io/driver/postgres`) | escape: sqlc / Ent / sqlx / Bun (record in CLAUDE.md / AGENTS.md) |
| Alembic | `golang-migrate/migrate` | escape: Atlas |
| connection pool | `pgxpool.New(...)` | — |

## Configuration

| Python | Go | Notes |
|---|---|---|
| `BaseSettings` / dynaconf | Viper (with Cobra CLI) | escape: `caarlos0/env` / plain env for simple services |
| `os.getenv` | `os.Getenv` or Viper | stdlib fine for few keys |
| `python-dotenv` | `joho/godotenv` if needed | — |
| struct binding | `viper.Unmarshal` or env tags | — |

## Validation

| Python | Go | Notes |
|---|---|---|
| Pydantic field validators | `go-playground/validator` struct tags | — |
| `@validator` decorator | custom validate funcs | — |
| complex conditional rules | `go-ozzo/ozzo-validation` | escape when tags get clunky |

## Auth & crypto

| Python | Go | Notes |
|---|---|---|
| `pyjwt` | `github.com/golang-jwt/jwt/v5` | — |
| JWT alternative | `aidanwoods.dev/go-paseto` | when you control both sides |
| `bcrypt` / passlib | `golang.org/x/crypto/bcrypt` | — |
| argon2 | `golang.org/x/crypto/argon2` (Argon2id) | default for new projects |
| OAuth2 client | `golang.org/x/oauth2` | — |
| sessions | `alexedwards/scs` or `gorilla/sessions` | — |
| TOTP | `pquerna/otp` | — |
| WebAuthn | `go-webauthn/webauthn` | — |
| RBAC | `casbin/casbin` | — |
| CSRF | middleware / `gorilla/csrf` | — |

## Background jobs & scheduling

| Python | Go | Notes |
|---|---|---|
| Celery task | `hibiken/asynq` (Redis) | default |
| Celery without Redis | `riverqueue/river` (Postgres) | — |
| Celery + Kafka | `ThreeDotsLabs/watermill` | escape |
| APScheduler cron | `robfig/cron/v3` | — |
| `schedule` | `go-co-op/gocron` | — |
| durable workflows | `temporalio/sdk-go` | high-stakes only |
| worker pool | errgroup + semaphore | optional ants |
| retry/backoff | `cenkalti/backoff/v4` or `avast/retry-go/v4` | — |
| distributed lock | `bsm/redislock` or PG advisory locks | — |

## HTTP client

| Python | Go | Notes |
|---|---|---|
| `requests` | `net/http.Client` **with timeout** | stdlib |
| `requests.Session` | `*http.Client` + RoundTripper | stdlib |
| `httpx` async | `net/http` + errgroup fan-out | stdlib |
| retry + backoff | `hashicorp/go-retryablehttp` | — |
| circuit breaker | `sony/gobreaker` | — |
| full resilience stack | resty / heimdall | escape |
| webhook signing | `crypto/hmac` + `sha256` | stdlib |

## Caching

| Python | Go | Notes |
|---|---|---|
| `functools.lru_cache` | map + `sync.Mutex` or `dgraph-io/ristretto` | stdlib / ristretto; avoid fancy loader libs by default |
| read-through + singleflight | `golang.org/x/sync/singleflight` + your loader | stdlib |
| `redis-py` | `redis/go-redis/v9` | — |
| Redis perf-critical | `redis/rueidis` | caveat: watch p99 |
| Memcached | `bradfitz/gomemcache` | — |

## Dependency injection

| Python | Go | Notes |
|---|---|---|
| FastAPI `Depends` | constructors at `main` / `cmd` composition root | stdlib default |
| `dependency-injector` | manual wiring | escape: `google/wire`, `uber-go/fx`, or `samber/do` v2 if graph >20 |

## Data pipelines (fragile)

| Python | Go | Notes |
|---|---|---|
| `pandas.DataFrame` | streaming `[]Struct` + channels — **redesign** | no pandas peer; qframe closest |
| load-bearing `numpy` | **STOP** — gRPC-wrap Python | do not translate |
| basic linalg | `gonum.org/v1/gonum` | — |
| CSV | `encoding/csv` (+ gocsv if needed) | stdlib |
| Parquet | `parquet-go/parquet-go` | — |
| Excel | `xuri/excelize/v2` | — |
| Kafka | `segmentio/kafka-go` or `twmb/franz-go` | — |
| NATS | `nats-io/nats.go` | — |
| S3 | `aws-sdk-go-v2/service/s3` | — |
| DuckDB | `marcboeker/go-duckdb` | local analytics |
| Airflow task | redesign DAG; goroutines or Temporal | no Airflow peer |

## CLI

| Python | Go | Notes |
|---|---|---|
| Click / Typer | `spf13/cobra` + `spf13/viper` | escape: `flag` for tiny tools |
| rich console | `charmbracelet/lipgloss` | — |
| rich table | `jedib0t/go-pretty/v6` | — |
| rich markdown | `charmbracelet/glamour` | — |
| tqdm | `schollz/progressbar/v3` | — |
| prompt_toolkit | `charmbracelet/huh` | — |
| spinner | `briandowns/spinner` | — |

## TUI

| Python | Go | Notes |
|---|---|---|
| Textual | `charmbracelet/bubbletea` | Elm rewrite, not 1:1 |
| Textual widgets | `charmbracelet/bubbles` | — |
| styling | `charmbracelet/lipgloss` | — |
| forms | `charmbracelet/huh` | — |
| SSH TUI | `charmbracelet/wish` | — |

## Encoding / serialization

| Python | Go | Notes |
|---|---|---|
| `json` | `encoding/json` | stdlib |
| JSON hot path | goccy/sonic after bench | escape |
| yaml | `gopkg.in/yaml.v3` | — |
| toml | `pelletier/go-toml/v2` | — |
| protobuf | `google.golang.org/protobuf` + buf | — |
| msgpack | `vmihailenco/msgpack/v5` | — |

## Testing

| Python | Go | Notes |
|---|---|---|
| pytest | `testing` + table-driven + `stretchr/testify/require` | prefer `require` |
| `@parametrize` | `for _, tc := range tests` | stdlib |
| `unittest.mock` | `go.uber.org/mock` | not archived golang/mock |
| mock gen | mockery or mockgen | — |
| httpx MockTransport | `net/http/httptest` + `httptest.Server` | stdlib |
| pytest-asyncio | goroutines + testcontainers-go | — |
| property-based | `testing.F` fuzz or gopter | stdlib first |
| benchmark | `BenchmarkX` + benchstat | stdlib |

## Observability

| Python | Go | Notes |
|---|---|---|
| OpenTelemetry | `go.opentelemetry.io/otel` + Fiber middleware | — |
| prometheus_client | `prometheus/client_golang` | — |
| Sentry | `getsentry/sentry-go` | optional slog bridge |
| profiling | `net/http/pprof` | stdlib |

## File / time / misc

| Python | Go | Notes |
|---|---|---|
| `pathlib.Path` | `path/filepath` + `os` | stdlib |
| `datetime` | `time` | reject carbon clones |
| `uuid` | `google/uuid` | — |
| watchdog | `fsnotify/fsnotify` | — |
| smtplib | `wneessen/go-mail` | — |
| `subprocess.run` | `os/exec` | stdlib |

## Container / build / release

| Python | Go | Notes |
|---|---|---|
| multi-stage Dockerfile | goreleaser + ko (or classic Dockerfile) | — |
| container base | `gcr.io/distroless/static-debian12` | — |
| lockfile | `go.mod` + `go.sum` | stdlib |
| lint | golangci-lint | — |
| format | gofumpt + goimports | — |

## Forbidden defaults

Must be an explicit escape hatch in `CLAUDE.md` / `AGENTS.md`:

- ❌ `lib/pq` — use pgx
- ❌ `golang/mock` — use `go.uber.org/mock`
- ❌ raw `database/sql` sprawl without a chosen access layer — GORM is default; sqlc/Ent/sqlx only as recorded escape
- ❌ bare `net/http.Get` with no timeout
- ❌ `panic` for business errors
- ❌ AST transpilers (Grumpy, etc.)
- ❌ `samber/lo` for ops covered by stdlib (`Contains`, `Sort`, `Keys`, simple map/filter)
- ❌ optional monadic/DI/reactive libs as the first choice
