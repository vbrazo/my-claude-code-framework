---
name: golang-idioms
description: Writing Go the way Go wants — error wrapping, small interfaces, safe concurrency, table tests, and module hygiene
---

# Idiomatic Go

## Handling errors

```go
// Return errors, never panic in library code
func LoadConfig(path string) (Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return Config{}, fmt.Errorf("reading config %s: %w", path, err)
    }

    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return Config{}, fmt.Errorf("parsing config: %w", err)
    }

    return cfg, nil
}
```

Guidelines:
- Wrap as you go, adding context with `fmt.Errorf("context: %w", err)`
- The `%w` verb is what lets callers reach back in with `errors.Is` and `errors.As`
- Deal with an error once — don't both log it and return it up the stack
- Give expected failure conditions named sentinel errors

```go
var (
    ErrNotFound    = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
)

func GetUser(id string) (User, error) {
    user, ok := store[id]
    if !ok {
        return User{}, fmt.Errorf("user %s: %w", id, ErrNotFound)
    }
    return user, nil
}

// Caller
user, err := GetUser(id)
if errors.Is(err, ErrNotFound) {
    http.Error(w, "user not found", http.StatusNotFound)
    return
}
```

## Designing interfaces

```go
// Keep interfaces small (1-3 methods)
type Reader interface {
    Read(p []byte) (n int, err error)
}

type UserStore interface {
    GetUser(ctx context.Context, id string) (User, error)
    CreateUser(ctx context.Context, u User) error
}

// Accept interfaces, return structs
func NewService(store UserStore, logger *slog.Logger) *Service {
    return &Service{store: store, logger: logger}
}
```

Guidelines:
- Declare an interface next to its consumer, not next to the type that satisfies it
- Favor small, composable interfaces over one sprawling one
- Lean on the standard library's `io.Reader`, `io.Writer`, `fmt.Stringer`
- A single-method interface takes the method's name plus an `er` suffix

## Goroutines and channels

### A worker pool
```go
func process(ctx context.Context, jobs <-chan Job, workers int) <-chan Result {
    results := make(chan Result, workers)
    var wg sync.WaitGroup

    for range workers {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                select {
                case <-ctx.Done():
                    return
                case results <- job.Execute():
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

### Fan-out, fan-in
```go
func fanOut[T, R any](ctx context.Context, items []T, fn func(T) R, concurrency int) []R {
    sem := make(chan struct{}, concurrency)
    results := make([]R, len(items))
    var wg sync.WaitGroup

    for i, item := range items {
        wg.Add(1)
        sem <- struct{}{}
        go func() {
            defer func() { <-sem; wg.Done() }()
            results[i] = fn(item)
        }()
    }

    wg.Wait()
    return results
}
```

Guidelines:
- Thread `context.Context` through as the first parameter
- Make sure every goroutine has an exit — context cancellation or a closed channel
- Wait on completion with a `sync.WaitGroup`
- Buffer channels when producer and consumer move at different speeds
- Never launch a goroutine until you know exactly how it ends

## Propagating context

```go
func (s *Service) HandleRequest(ctx context.Context, req Request) (Response, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    user, err := s.store.GetUser(ctx, req.UserID)
    if err != nil {
        return Response{}, fmt.Errorf("getting user: %w", err)
    }

    ctx = context.WithValue(ctx, userKey, user)
    return s.processRequest(ctx, req)
}
```

Guidelines:
- Any function that touches I/O takes context as its first parameter
- Bound every external call with `context.WithTimeout` or `context.WithDeadline`
- `defer cancel()` the moment you create a cancellable context
- Reach for `context.WithValue` only for request-scoped data like trace IDs or auth info
- Never stash a context inside a struct

## Table-driven tests

```go
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name  string
        email string
        want  bool
    }{
        {"valid email", "user@example.com", true},
        {"missing @", "userexample.com", false},
        {"empty string", "", false},
        {"multiple @", "user@@example.com", false},
        {"valid with subdomain", "user@mail.example.com", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := ValidateEmail(tt.email)
            if got != tt.want {
                t.Errorf("ValidateEmail(%q) = %v, want %v", tt.email, got, tt.want)
            }
        })
    }
}
```

### Test helpers
```go
func newTestServer(t *testing.T) *httptest.Server {
    t.Helper()
    handler := setupRoutes()
    srv := httptest.NewServer(handler)
    t.Cleanup(srv.Close)
    return srv
}

func assertEqual[T comparable](t *testing.T, got, want T) {
    t.Helper()
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}
```

Call `t.Helper()` in every test utility so failures point at the caller. Prefer `t.Cleanup()` over `defer` for tearing down test resources, and keep fixtures under a `testdata/` directory.

## Managing modules

```
go.mod structure:
module github.com/org/project

go 1.23

require (
    github.com/lib/pq v1.10.9
    golang.org/x/sync v0.7.0
)
```

Commands:
```bash
go mod tidy          # remove unused, add missing
go mod verify        # verify checksums
go list -m -u all    # check for updates
go get -u ./...      # update all dependencies
go mod vendor        # vendor dependencies (optional)
```

Run `go mod tidy` before each commit, pin major versions, and read the changelog before bumping anything.

## Making the zero value useful

Shape your types so the zero value is already usable:

```go
// sync.Mutex zero value is an unlocked mutex (ready to use)
var mu sync.Mutex

// bytes.Buffer zero value is an empty buffer (ready to use)
var buf bytes.Buffer
buf.WriteString("hello")

// Custom types: make zero value meaningful
type Server struct {
    Addr    string        // defaults to ""
    Handler http.Handler  // defaults to nil
    Timeout time.Duration // defaults to 0 (no timeout)
}

func (s *Server) ListenAndServe() error {
    addr := s.Addr
    if addr == "" {
        addr = ":8080" // useful default
    }
    handler := s.Handler
    if handler == nil {
        handler = http.DefaultServeMux
    }
    // ...
}
```

Guidelines:
- Reach for a meaningful zero value before reaching for a constructor
- Use a pointer receiver when the method mutates the receiver
- Use a value receiver when the method only reads
- Don't export fields callers shouldn't set directly — gate them behind a constructor

## Structured logging

```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

logger.Info("request handled",
    slog.String("method", r.Method),
    slog.String("path", r.URL.Path),
    slog.Int("status", status),
    slog.Duration("latency", time.Since(start)),
)
```

Reach for `log/slog` from the standard library (Go 1.21+). Log structured fields rather than interpolated strings, and carry request ID, user ID, and operation name on every entry.

## What to avoid

- Handing back `interface{}` / `any` where a concrete type would do
- Hiding non-trivial setup in `init()`, which makes tests awkward
- Swallowing errors with a bare `_` and no explanation
- Spawning goroutines with no plan for their lifecycle
- Holding a lock over too much code and creating mutex contention
- Reaching for channels where a plain mutex would guard shared state more simply
- Naked returns in anything longer than a few lines
