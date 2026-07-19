---
name: go
description: Use when writing or reviewing Go code — language idioms, error handling, concurrency, HTTP clients/servers, testing conventions, and project layout.
---

Apply Go language standards and idioms to the current task.

**Writing code** — follow all standards in this document; silently verify against the §18 checklist before returning output.

**Code review** — §18 defines WHAT to check; output format and severity levels are defined by the loaded review skill.

These standards are general Go idioms drawn from Effective Go, the Go Code Review Comments wiki, official Go blog posts, `go vet`/`gofmt` defaults, and the Google and Uber Go style guides. They apply to any Go codebase — a library, a CLI, a service, a script — regardless of architecture or framework.

---

## 1. Language Features (Go 1.22+)

Prefer modern standard-library features over hand-rolled or third-party equivalents:

| Feature | When to use |
|---------|-------------|
| `log/slog` | Standard structured logger — instead of `log` or third-party loggers |
| `slices` package | `slices.Contains`, `slices.Sort`, `slices.Index`, `slices.Equal` |
| `maps` package | `maps.Keys`, `maps.Values`, `maps.Clone` |
| `min()`, `max()` builtins | Instead of manual if-comparisons |
| `clear()` builtin | Empty a map or zero a slice |
| Range over int (1.22+) | `for i := range n` instead of a C-style counting loop |
| Range over func (1.23+) | `for v := range iterFunc` custom iterators |
| Generics | Type-safe collections and utility functions |
| `errors.Is` / `errors.As` | Always for error inspection — never `==` on wrapped errors |
| `context.WithCancelCause` | More informative cancellation |

```go
// Use min/max instead of an if
timeout := min(requestTimeout, globalMaxTimeout)

// slices instead of a manual loop
if slices.Contains(allowedRoles, role) {
    // ...
}

// Generics for type-safe utility functions
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}
```

Do not add generics or abstractions speculatively — reach for them when concrete types genuinely repeat (§16, YAGNI).

---

## 2. Project Layout

Go does not prescribe a strong project layout. The only conventions the toolchain and community actually enforce are:

- `cmd/<name>/main.go` — one directory per binary; `main` packages do wiring and startup only.
- `internal/` — packages here cannot be imported by other modules; put anything not meant as public API here.
- `pkg/` — optional; only if you deliberately export reusable packages. Many projects don't need it.
- Package name = directory name; keep packages cohesive around a purpose.

Beyond that, let the design of the program drive the structure. Do **not** impose a mandated layered architecture. Avoid catch-all packages named `util`, `helpers`, `common`, `misc`, `base`, or `shared` — name packages for what they provide.

---

## 3. Functional Options Pattern

For constructors with many optional parameters, prefer functional options over long parameter lists or a config struct the caller must fully populate:

```go
type Server struct {
    addr    string
    timeout time.Duration
    maxConns int
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func WithMaxConns(n int) Option {
    return func(s *Server) { s.maxConns = n }
}

func NewServer(opts ...Option) *Server {
    s := &Server{
        addr:     ":8080",
        timeout:  30 * time.Second,
        maxConns: 100,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

Use it when it clarifies call sites; a plain struct or a few parameters is fine when there are only one or two knobs (KISS).

---

## 4. Dependency Injection

Go has no built-in DI container, and the standard-library idiom is **explicit constructor injection**: a type declares its dependencies as constructor parameters (often interfaces), and the caller wires concrete implementations together, typically in `main`.

```go
type Store interface {
    Get(ctx context.Context, id string) ([]byte, error)
}

type Service struct {
    store Store
    log   *slog.Logger
}

func NewService(store Store, log *slog.Logger) *Service {
    return &Service{store: store, log: log}
}

// main.go — manual wiring
func main() {
    store := postgres.New(dsn)
    log := slog.Default()
    svc := NewService(store, log)
    // ...
}
```

For small to medium programs, manual wiring in `main` is idiomatic and usually sufficient — it stays simple, typo-safe, and debuggable at compile time. As the dependency graph grows, two community tools formalize this without changing the underlying idiom:

- **[Uber fx](https://github.com/uber-go/fx)** — reflection-based DI at runtime; constructors are registered with `fx.Provide`, and `fx.Invoke`/lifecycle hooks (`OnStart`/`OnStop`) build and tear down the graph. Trades some compile-time safety for less wiring boilerplate in large graphs.
- **[Google Wire](https://github.com/google/wire)** — compile-time code generation; produces plain constructor-calling Go code, so wiring errors surface at build time rather than at startup.

Guidelines that apply regardless of whether wiring is manual, fx, or wire:

- Keep wiring code separate from business logic — a dedicated `main`/wiring function, not scattered `New*` calls inside handlers or use cases.
- Depend on interfaces at the consumer, not concrete types (§6), so the same constructor works with a real implementation or a test fake.
- If a component starts background work (a goroutine, a connection pool), give it an explicit start/stop lifecycle (§12) so DI teardown — whether manual `defer`, an `fx.Lifecycle` hook, or a generated cleanup func — actually runs it.
- Avoid package-level singletons and `init()`-time wiring (§17 anti-patterns) — they bypass DI entirely and make tests hard to isolate.

---

## 5. Naming Conventions

### Packages

- Short, lowercase, single word, no underscores or `mixedCaps`: `bytes`, `http`, `strconv`.
- Package name = directory name.
- Don't stutter: the package qualifies the identifier, so `widget.New`, not `widget.NewWidget`; `widget.Service`, not `widget.WidgetService`.

### Identifiers

| Kind | Style | Examples |
|------|-------|----------|
| Exported types/funcs | `MixedCaps` | `Fetcher`, `ReadAll` |
| Unexported | `mixedCaps` | `fetcher`, `readAll` |
| Interfaces (single-method) | `-er` suffix | `Reader`, `Writer`, `Notifier` |
| Acronyms | Consistent case | `userID`, `httpClient`, `URL`, `ServeHTTP` |
| Constants | `MixedCaps` (not `SCREAMING_SNAKE`) | `MaxRetries`, `defaultTimeout` |
| Sentinel errors | `Err` prefix | `ErrNotFound`, `ErrClosed` |
| Error types | `Error` suffix | `ValidationError`, `ParseError` |

```go
// Correct
var ErrNotFound = errors.New("not found")
const MaxRetries = 3
type ParseError struct{ /* ... */ }

// Incorrect
var notFound = errors.New("not found") // sentinel needs Err prefix
const MAX_RETRIES = 3                  // not Go casing
```

- Getters have no `Get` prefix: `w.Owner()`, not `w.GetOwner()`. Setters do use `Set`: `w.SetOwner(...)`.
- Constructors are named `New` (single type in package) or `NewT` (multiple types).
- Keep names short in small scopes (`i`, `r`, `buf`) and more descriptive as scope widens.
- Receiver names are short (1–2 chars), consistent across all methods of a type, and never `self`/`this`.

```go
func New(log *slog.Logger) *Service { /* ... */ }

func (s *Service) Owner() string { return s.owner } // getter, no Get
```

---

## 6. Types and Interfaces

- Define named types over bare primitives when it prevents mixing up arguments or documents intent:

```go
// Bad: easy to transpose the two strings
func Transfer(from, to string, amount int) error

// Good: the compiler catches mistakes
type AccountID string
type Cents int64

func Transfer(from, to AccountID, amount Cents) error
```

- **Accept interfaces, return concrete types.** Define an interface where it is *consumed*, not where it is implemented, and keep it small — `io.Reader` (one method) is the ideal; a 20-method interface is a smell.
- Don't pass a pointer to an interface; pass the interface value.
- Use `any` (alias for `interface{}`) only when the type genuinely varies; prefer concrete types or generics.

```go
// A small consumer-side interface
type Store interface {
    Get(ctx context.Context, id string) ([]byte, error)
}

// Generic helper — panics turn a (value, error) pair into value
func Must[T any](v T, err error) T {
    if err != nil {
        panic(err)
    }
    return v
}
```

---

## 7. Error Handling

### Rules

- Never ignore an error: `v, _ := fn()` is forbidden except for functions whose error is documented as always-nil or genuinely irrelevant (and even then, prefer to be explicit).
- Handle an error **once**: either log it or return it, not both. Logging and returning produces duplicate log lines up the stack.
- Return early; keep the happy path un-indented on the left.
- Add context when propagating, and wrap with `%w` so the chain stays inspectable:

```go
// Incorrect: ignored error
v, _ := parse(data)

// Incorrect: handled twice — logged AND returned, duplicate log lines up the stack
v, err := parse(data)
if err != nil {
    log.Error("failed", "err", err)
    return err
}

// Correct: handled once, with context
v, err := parse(data)
if err != nil {
    return fmt.Errorf("load config: parse: %w", err)
}
```

- Error strings are lowercase and without trailing punctuation (they get wrapped): `"open file: permission denied"`, not `"Open file: permission denied."`.
- Use `%w` to wrap (preserves the chain for `errors.Is`/`errors.As`); use `%v` only when you deliberately want to *hide* the underlying error from callers.

### Sentinel errors and error types

```go
// Sentinel errors — for callers that need to match on a specific condition
var (
    ErrNotFound = errors.New("not found")
    ErrClosed   = errors.New("already closed")
)

// Error type — when callers need structured data about the failure
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s: %s", e.Field, e.Message)
}
```

Inspect with `errors.Is` (sentinel) and `errors.As` (type), never `==` or a type assertion, because the error may be wrapped:

```go
if errors.Is(err, ErrNotFound) {
    // handle missing
}

var ve *ValidationError
if errors.As(err, &ve) {
    // use ve.Field, ve.Message
}
```

If a custom error type needs custom matching semantics (e.g. two values are "equal" when a status field matches), implement `Is(target error) bool` on it so `errors.Is` works as intended:

```go
func (e *StatusError) Is(target error) bool {
    t, ok := target.(*StatusError)
    return ok && e.Code == t.Code
}
```

### `defer` for cleanup

Place cleanup in a `defer` immediately after acquiring the resource, so it runs on every return path:

```go
func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("open %s: %w", path, err)
    }
    defer f.Close() // runs on every return below

    // ...
    return nil
}
```

For writable files where a failed `Close` matters, capture the error: `defer func() { err = errors.Join(err, f.Close()) }()` in a named-return function.

### Retries for transient failures

Retry only transient/network errors with exponential backoff, honor context cancellation, and never retry non-retryable (e.g. client 4xx) errors:

```go
func IsRetryable(err error) bool {
    var netErr net.Error
    return errors.As(err, &netErr) && netErr.Timeout()
}

func Retry(ctx context.Context, attempts int, base time.Duration, fn func() error) error {
    var err error
    for attempt := range attempts {
        if err = ctx.Err(); err != nil {
            return err
        }
        if err = fn(); err == nil {
            return nil
        }
        if !IsRetryable(err) {
            return err // don't retry non-transient errors
        }
        delay := base << attempt // exponential backoff
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(delay):
        }
    }
    return fmt.Errorf("after %d attempts: %w", attempts, err)
}
```

- Do not `panic` for expected/recoverable errors — return an `error`. Reserve `panic` for truly unrecoverable programmer errors, and recover only at process boundaries.

---

## 8. Logging

**Standard:** `log/slog`. Avoid `fmt.Println`/`log.Printf` for application logging.

```go
func setupLogger(env string) *slog.Logger {
    var h slog.Handler
    if env == "production" {
        h = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelInfo})
    } else {
        h = slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{Level: slog.LevelDebug})
    }
    return slog.New(h)
}
```

### Rules

- Pass the logger explicitly (via constructor or `context`) rather than relying on a package-global.
- Prefer the `*Context` methods so handlers can extract values (e.g. a trace ID) from the context:

```go
log.InfoContext(ctx, "request handled",
    slog.String("method", r.Method),
    slog.Int("status", status),
    slog.Duration("elapsed", elapsed),
)
```

- Attach shared attributes once with `Logger.With`:

```go
log = log.With(slog.String("component", "fetcher"), slog.String("version", version))
```

### Levels

| Level | When |
|-------|------|
| `Debug` | Developer detail, off in production |
| `Info`  | Normal, noteworthy events |
| `Warn`  | Recoverable/expected problems (retry, fallback) |
| `Error` | Failures needing attention |

---

## 9. HTTP Clients and Servers

Framework choice (stdlib `net/http`, chi, echo, gin, …) is up to the project — these rules apply regardless.

- **Separation of concerns:** keep request handlers thin. Decode input, call into business logic, encode the response. Business logic does not belong inline in a transport handler.
- **Always set server timeouts.** A server without them is vulnerable to slow-client (Slowloris) resource exhaustion:

```go
srv := &http.Server{
    Addr:              addr,
    Handler:           handler,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       15 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       120 * time.Second,
}
```

- **Always set a client timeout** (the zero-value `http.Client` waits forever), and use request-scoped context:

```go
client := &http.Client{Timeout: 10 * time.Second}

req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
    return err
}
resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close() // always, to free the connection back to the pool
```

- Graceful shutdown: run `ListenAndServe` in a goroutine, and on signal call `srv.Shutdown(ctx)` with a bounded timeout so in-flight requests drain:

```go
func run(ctx context.Context, srv *http.Server) error {
    errCh := make(chan error, 1)
    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            errCh <- err
        }
    }()

    select {
    case err := <-errCh:
        return err
    case <-ctx.Done():
        shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        return srv.Shutdown(shutdownCtx)
    }
}
```

---

## 10. CLI Programs

`flag` (stdlib) covers most needs; reach for a library like `cobra` only for large multi-command tools. Parse flags in `main`, wire graceful shutdown with `signal.NotifyContext`, and exit non-zero on failure:

```go
func main() {
    verbose := flag.Bool("verbose", false, "enable verbose output")
    flag.Parse()

    ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    if err := run(ctx, *verbose); err != nil {
        fmt.Fprintln(os.Stderr, "error:", err)
        os.Exit(1)
    }
}
```

- Keep `main` small: parse, wire, call `run(ctx, ...) error`. Put testable logic in `run`.
- Write diagnostics to `stderr` and program output to `stdout`.
- Exit with a non-zero code on error (`os.Exit(1)`); note `os.Exit` skips deferred calls, so only call it from `main`.

---

## 11. Testing

### Table-driven tests

The standard Go pattern. Add a case by adding a table row:

```go
func TestClamp(t *testing.T) {
    t.Parallel()

    tests := []struct {
        name          string
        value, lo, hi int
        want          int
    }{
        {"below range", -1, 0, 10, 0},
        {"within range", 5, 0, 10, 5},
        {"above range", 42, 0, 10, 10},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()

            got := Clamp(tt.value, tt.lo, tt.hi)
            if got != tt.want {
                t.Errorf("Clamp(%d, %d, %d) = %d, want %d",
                    tt.value, tt.lo, tt.hi, got, tt.want)
            }
        })
    }
}
```

### Rules

- `t.Parallel()` in tests and subtests that are safe to run concurrently.
- `t.Helper()` in assertion/setup helpers so failures report the caller's line.
- `t.Fatal`/`t.Fatalf` when the test cannot meaningfully continue (e.g. setup failed); `t.Error`/`t.Errorf` to report a failure but keep checking.
- Name tests `Test<Type>_<Method>` or `Test<Func>`; use descriptive subtest names.
- Mock at **interfaces**, not concrete structs — define the small interface at the consumer and pass a fake in tests. Prefer hand-written fakes; generators like `mockery` are fine for large surfaces.
- Compare complex values with `reflect.DeepEqual` or `go-cmp`'s `cmp.Diff` (reporting the diff), not field by field.
- Use `httptest` for HTTP handlers/clients and `t.TempDir()` for filesystem tests (auto-cleaned).
- Tag slow tests needing external resources with a build constraint and run them separately:

```go
//go:build integration

package store_test
```

- Run with `-race` (`go test -race ./...`) so data races surface before production.

### Principles

- New logic is covered by tests. Tests should fail when the behavior they cover changes — a test that stays green after a logic change verifies nothing.
- Test observable behavior (input → output, resulting state), not implementation details.
- Cover edge cases: nil, empty slice/map, zero and boundary values, a canceled context.
- Don't test the standard library; test your own logic. Keep each test focused and non-redundant.

---

## 12. Concurrency

### Goroutine discipline

- Every goroutine needs a clear lifecycle: **whoever starts a goroutine is responsible for its termination and for observing its error.** Never "fire and forget."
- Pass `context.Context` as the first parameter to any function doing I/O or blocking work, and select on `ctx.Done()` in loops and channel operations so work is cancelable.
- Background goroutines that must finish before exit (flushing, draining a queue) need a way to wait — a `sync.WaitGroup` drained during graceful shutdown — so work isn't lost when the process stops.

```go
// Fire-and-forget — the error is lost and the goroutine may outlive its caller
go doWork(item)

// Tracked with errgroup — errors propagate, and Wait blocks until done
g, ctx := errgroup.WithContext(ctx)
g.Go(func() error {
    return doWork(ctx, item)
})
if err := g.Wait(); err != nil {
    return fmt.Errorf("work: %w", err)
}
```

Graceful drain of a tracked background goroutine:

```go
type Worker struct{ wg sync.WaitGroup }

func (w *Worker) Start(ctx context.Context, item Item) {
    w.wg.Add(1)
    go func() {
        defer w.wg.Done()
        w.process(ctx, item)
    }()
}

func (w *Worker) Shutdown(ctx context.Context) error {
    done := make(chan struct{})
    go func() { w.wg.Wait(); close(done) }()
    select {
    case <-done:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}
```

### Bounded worker pool

Bound fan-out; unbounded goroutine creation exhausts resources:

```go
func processAll(ctx context.Context, items []Item, workers int) error {
    g, ctx := errgroup.WithContext(ctx)
    ch := make(chan Item)

    g.Go(func() error {
        defer close(ch)
        for _, it := range items {
            select {
            case ch <- it:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })

    for range workers {
        g.Go(func() error {
            for it := range ch {
                if err := process(ctx, it); err != nil {
                    return err
                }
            }
            return nil
        })
    }

    return g.Wait()
}
```

### Channel and sync rules

- Only the **sender** closes a channel; closing from the receiver (or double-closing) panics.
- Never send/receive without a `select` guarded by `ctx.Done()` (or a timeout) when a block could be permanent.
- Protect shared mutable state with a mutex, or don't share it (communicate via channels). Use `sync.RWMutex` for read-heavy data.
- `sync.Once` for one-time initialization; `sync.Pool` to reuse allocations on hot paths.

```go
var (
    once sync.Once
    inst *Service
)

func Instance() *Service {
    once.Do(func() { inst = newService() })
    return inst
}
```

---

## 13. Documentation

Doc comments precede the declaration and begin with the name being documented:

```go
// Package widget provides primitives for assembling and validating widgets.
package widget

// Widget is a single assembled unit.
type Widget struct {
    ID   string
    Size int
}

// Resize sets the widget's size. It returns ErrInvalidSize if size is negative.
func (w *Widget) Resize(size int) error { /* ... */ }
```

- Every exported package, type, function, method, and var/const that isn't self-evident gets a doc comment.
- Each package has a package comment (one file, often `doc.go` for larger packages).
- Comments are complete sentences in English.
- `//nolint:<linter> // <reason>` and `//go:generate` directives must carry a reason/purpose.

---

## 14. Formatting, Vetting, and Linting

- `gofmt` (or `goimports`, which also groups/prunes imports) is non-negotiable — all code is formatted. Never hand-format.
- `go vet ./...` must be clean; it catches real bugs (bad `Printf` verbs, lock copies, unreachable code).
- `golangci-lint` is the community-standard aggregator. A reasonable default set:

```yaml
# .golangci.yml
linters:
  enable:
    - errcheck      # unchecked errors
    - govet         # go vet
    - staticcheck   # comprehensive static analysis
    - ineffassign   # ineffectual assignments
    - unused        # unused code
    - gofmt         # formatting
    - goimports     # import grouping
    - revive        # style rules
    - bodyclose     # unclosed http.Response.Body
    - noctx         # HTTP requests without context
```

Wire these into CI so `gofmt`/`go vet`/`golangci-lint`/`go test -race` all run on every change.

---

## 15. Modules and Dependencies

| Command | When |
|---------|------|
| `go mod tidy` | After adding/removing imports; before committing |
| `go get pkg@version` | Add or update a dependency |
| `go mod verify` | Verify module integrity |
| `go list -m all` | List the full dependency graph |

- Always commit `go.sum`.
- Run `go mod tidy` so the module graph stays minimal and accurate.
- Don't add a dependency for something the standard library already does well. Every dependency is surface area to maintain and audit.
- Use `go mod vendor` only when you need reproducible builds without network access.

---

## 16. Principles

**SOLID, applied to Go:**
- **S** — one package/type, one responsibility.
- **O** — extend via new types satisfying an interface rather than editing existing ones.
- **L** — interface implementations must be interchangeable (a fake and the real thing behave the same).
- **I** — small interfaces; `io.Reader` (one method) over a wide interface.
- **D** — depend on interfaces defined at the consumer, not concrete implementations.

**DRY** — factor genuinely shared logic into a function/package; but a little duplication is cheaper than the wrong abstraction.

**KISS** — clear code over clever code. Go optimizes for readers.

**YAGNI** — don't add interfaces, generics, or configuration for hypothetical future needs.

---

## 17. Anti-Patterns

| Anti-pattern | Correct approach |
|--------------|------------------|
| `v, _ := fn()` discarding an error | Handle the error |
| Logging **and** returning the same error | Handle once — log or return |
| `panic()` for expected/recoverable errors | Return an `error` |
| Global mutable state | Pass dependencies explicitly |
| Goroutine with no lifecycle | `errgroup` / `sync.WaitGroup` / done channel |
| `interface{}` / `any` overuse | Concrete types or generics |
| `init()` doing heavy or side-effecting work | Explicit setup in `main`/constructor |
| HTTP server/client without timeouts | Set Read/Write/Idle (server) and `Timeout` (client) |
| Pointer to an interface | Pass the interface value |
| `time.Sleep` in a retry loop | `select` on `time.After` + `ctx.Done()` |
| `fmt.Errorf("...: %v", err)` when you meant to wrap | `%w` to preserve the chain |
| Package named `util`/`helpers`/`common` | A name describing what it provides |
| Overly long functions | Decompose into focused functions |
| `//nolint` with no reason | `//nolint:errcheck // reason: ...` |

---

## 18. Checklists

### When writing code

Silently verify before returning code:

- [ ] Every `error` is checked — no `v, _ := fn()` discarding an error
- [ ] Errors are either logged or returned, not both; wrapped with `%w` when propagated
- [ ] Early return for invalid/boundary cases — happy path un-nested on the left
- [ ] `context.Context` is the first parameter of every function doing I/O; passed through, not `context.Background()` deep in logic
- [ ] A goroutine is started only with a way to observe completion/errors (`errgroup`/`WaitGroup`/done channel)
- [ ] Channel ops and blocking waits use `select` with `ctx.Done()` or a timeout — no unconditional blocking
- [ ] Shared mutable state is protected by a mutex (or not shared)
- [ ] `defer f.Close()` / `defer rows.Close()` / `defer resp.Body.Close()` immediately after a successful open
- [ ] `defer cancel()` after every `context.WithCancel`/`WithTimeout`
- [ ] `http.Server` has Read/Write/Idle timeouts; `http.Client` has a `Timeout`
- [ ] Network calls retry only transient errors, with backoff and context cancellation — not 4xx/client errors
- [ ] Graceful shutdown via `signal.NotifyContext`, draining in-flight work with a bounded timeout
- [ ] `log/slog` with `*Context` methods — not `fmt.Println`/`log.Printf`
- [ ] Exported types, funcs, and methods have godoc comments (English, starting with the name)
- [ ] New logic is covered by a table-driven test with `t.Parallel()`/`t.Helper()` where appropriate
- [ ] No leftover debug output (`fmt.Println`, `spew.Dump`) outside tests; no `TODO`/`FIXME` without a linked issue
- [ ] Long-lived processes don't grow maps/slices/caches unbounded — bound or evict; caches have a TTL/max size
- [ ] Fan-out is bounded by a worker limit or semaphore
- [ ] `gofmt`/`goimports` clean, `go vet ./...` clean, `golangci-lint run ./...` clean, `go test -race ./...` clean
- [ ] `go mod tidy` — module graph is minimal; no dependency added for something stdlib handles
- [ ] Naming follows Go conventions (MixedCaps, `Err`-prefixed sentinels, `Error`-suffixed types, no `Get` on getters, consistent acronym case)

### When reviewing code

**🔴 Critical — blockers:**
- [ ] `v, _ := fn()` where `_` is an `error` → ignored error
- [ ] Goroutine launched with no lifecycle (`errgroup`/`WaitGroup`/done channel) → goroutine leak
- [ ] Shared mutable map/slice without synchronization → data race, undefined behavior
- [ ] `close(ch)` by a receiver, or double close → panic; only the sender closes
- [ ] `ch <- v` / `<-ch` with no `select`/timeout guard → permanent block / deadlock under load
- [ ] Goroutine with no `ctx.Done()`/done channel → leak under load
- [ ] `http.Server` without `ReadTimeout`/`WriteTimeout` → Slowloris / resource exhaustion
- [ ] `http.Client` with no `Timeout` → hangs forever on a stalled peer
- [ ] `resp.Body` not closed via `defer resp.Body.Close()` → connection/pool leak
- [ ] Missing `defer f.Close()` after `os.Open` → fd leak; in a loop → fd exhaustion
- [ ] `rows`/`stmt` not closed, or transaction without `Rollback` on the error path → resource leak / locks
- [ ] SQL built by string concatenation of input → SQL injection; use parameter placeholders
- [ ] `math/rand` for tokens/secrets/security-sensitive values → use `crypto/rand`
- [ ] Secrets hardcoded in source → environment/secret manager
- [ ] `panic()` in normal control flow for an expected error → return an `error`
- [ ] `sql.DB` without `SetMaxOpenConns`/`SetMaxIdleConns`/`SetConnMaxLifetime` → connection exhaustion
- [ ] New logic with no test coverage

**🟡 Warnings:**
- [ ] `context.Context` not the first parameter of an I/O function → convention violated
- [ ] `context.Background()` used deep in business logic (not `main`/tests) → not cancelable
- [ ] Missing `defer cancel()` after `WithTimeout`/`WithCancel` → context leak
- [ ] Error logged **and** returned → handled twice
- [ ] `fmt.Errorf("...: %v", err)` where the chain matters → use `%w`
- [ ] `==` or a type assertion to compare errors instead of `errors.Is`/`errors.As` → breaks on wrapped errors
- [ ] Custom error type needing value-based matching without an `Is(target error) bool` method → `errors.Is` silently fails
- [ ] Retry without a retryable check (retries 4xx) or without `ctx.Done()` in the backoff → not cancelable
- [ ] Missing graceful shutdown via `signal.NotifyContext`
- [ ] `log.Printf`/`fmt.Println` instead of `slog` → unstructured logging
- [ ] Packages named `util`/`helpers`/`common`/`misc` → use a descriptive name
- [ ] Fan-out with no bound (semaphore/worker limit) → unbounded goroutines
- [ ] Unbounded map/slice/cache in a long-lived process; cache without TTL/max size → memory growth / OOM
- [ ] Temp files/dirs not cleaned up (prefer `t.TempDir()` in tests, `defer os.Remove` otherwise)
- [ ] `fmt.Println`/`spew.Dump` left outside tests, or `TODO`/`FIXME` without a linked issue
- [ ] Tests are superficial, verify implementation details instead of behavior, or duplicate each other
- [ ] Tests still pass after the covered behavior changes → they verify nothing
- [ ] `go test -race` not run → data races undetected
- [ ] Business logic living in a transport handler → separation of concerns
- [ ] API/config/env change without corresponding doc update

**🔵 Suggestions:**
- [ ] Interface with too many methods → split into smaller ones
- [ ] Interface defined in the implementing package rather than the consumer → Go idiom reversed
- [ ] Missing `t.Parallel()` in independent tests/subtests
- [ ] Sentinel errors without `Err` prefix; error types without `Error` suffix
- [ ] Inconsistent acronym case: `userId` → `userID`, `Url` → `URL`
- [ ] `interface{}`/`any` where a concrete type or generic would do
- [ ] Getter named `GetX` → drop the `Get`
- [ ] Stuttering identifiers: `widget.WidgetService` → `widget.Service`
- [ ] Long parameter list of options → functional options pattern
