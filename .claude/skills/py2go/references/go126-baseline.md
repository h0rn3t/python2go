# Go 1.26+ baseline

Use the modern form **whenever it applies**. Do not invent uses just to showcase a feature. Skip experimental `GOEXPERIMENT=…` APIs unless the user opts in.

| Prefer | Over | Since |
|---|---|---|
| `any` | `interface{}` | 1.18 |
| generics | `any` + type-assert soup | 1.18 |
| `errors.Is` / `errors.As` / `errors.Join` | `== err`, multierr libs | 1.13+ / 1.20 |
| `min` / `max` / `clear` | hand-rolled | 1.21 |
| `slices` / `maps` / `cmp` (`Sort`, `Contains`, `Clone`, `Or`, …) | `sort.Slice`, manual loops | 1.21 |
| `log/slog` | `log`, zap/zerolog (unless escape hatch) | 1.21 |
| `sync.OnceValue` / `OnceFunc` | custom sync.Once wrappers | 1.21 |
| `for i := range n` | `for i := 0; i < n; i++` | 1.22 |
| no loop-var shadow copies | `x := x` in range loops | 1.22 |
| `math/rand/v2` | `math/rand` + `rand.Seed` | 1.22 |
| `database/sql.Null[T]` | hand-rolled null wrappers | 1.22 |
| `reflect.TypeFor[T]()` | `reflect.TypeOf((*T)(nil)).Elem()` | 1.22 |
| `iter` / `range` over funcs; `slices.Collect` / `maps.Keys` seq forms | awkward intermediate slices when an iterator is clearer | 1.23 |
| `omitzero` JSON tag | `omitempty` for zero structs/values | 1.24 |
| `os.Root` for user-supplied paths | open-at-path without root jail | 1.24 |
| `t.Context()` / `b.Loop()` | `context.Background()` in tests; old bench timers | 1.24 |
| `runtime.AddCleanup` | `runtime.SetFinalizer` | 1.24 |
| `tool` directives in `go.mod` | stray tool deps in `require` | 1.24 |
| `strings.SplitSeq` / `FieldsSeq` / `Lines` | split + range when streaming | 1.24 |
| `sync.WaitGroup.Go` | `wg.Add` + `go func` + `Done` for simple cases | 1.25 |
| `testing/synctest` for concurrent unit tests | time-sleep flake harnesses | 1.25 |
| `new(expr)` for non-zero pointers | `ptr := v; &ptr` helpers | 1.26 |
| `errors.AsType[T](err)` | `var x T; errors.As(err, &x)` | 1.26 |
| `slog.NewMultiHandler` | third-party slog fan-out | 1.26 |
| `httputil.ReverseProxy{Rewrite: …}` | `Director`-based proxies | 1.26 |
| self-referential generic constraints when they simplify builders/trees | awkward workarounds | 1.26 |

After each translate module and at validate: `go fix ./...`. Keep `AGENTS.md` / `CLAUDE.md` stating `go 1.26+`.
