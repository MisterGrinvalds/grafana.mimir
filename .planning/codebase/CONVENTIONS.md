# Coding Conventions

**Analysis Date:** 2026-02-23

## Naming Patterns

**Files:**
- Source files: `snake_case.go` (e.g., `distributor.go`, `ingester_test.go`)
- Test files: `<source>_test.go` co-located with source
- Mock files: `*_mock.go` or `*_mock_test.go`
- Protocol buffers: `*.proto` with generated `*.pb.go`

**Functions:**
- Public: `PascalCase` (e.g., `NewDistributor`, `Push`)
- Private: `camelCase` (e.g., `validateLabels`, `pushToIngesters`)
- Test functions: `Test<Component>_<Method>` (e.g., `TestDistributor_Push`)
- Test helper functions: `camelCase` with `t.Helper()` call (e.g., `defaultIngesterTestConfig`)

**Variables:**
- Local: `camelCase`
- Package-level: `camelCase` (private) or `PascalCase` (exported)
- Constants: `PascalCase` for exported, `camelCase` for private
- Error variables: `err<Description>` (e.g., `errInvalidTenantShardSize`)

**Types:**
- Structs: `PascalCase` (e.g., `Distributor`, `Config`)
- Interfaces: `PascalCase`, often ending with `-er` suffix (e.g., `Pusher`, `Limiter`)
- Error types: implement `error` interface, often with `ID` type for user-facing errors

## Code Style

**Formatting:**
- Tool: `gofmt` via golangci-lint
- Indentation: tabs
- Line length: no hard limit, but keep reasonable

**Linting:**
- Primary tool: `golangci-lint` configured in `.golangci.yml`
- Key linters enabled:
  - `errorlint` - error comparison checks
  - `forbidigo` - forbidden API patterns (e.g., no `CloseSend` on gRPC streams)
  - `gocritic` - dupImport check
  - `gosec` - security checks
  - `revive` - code style
  - `staticcheck` - static analysis
  - `misspell` - spelling in docs

**Blocked Packages (enforced via faillint):**
- Use `github.com/stretchr/testify/assert` not `github.com/bmizerany/assert`
- Use `context` not `golang.org/x/net/context`
- Use `go.uber.org/atomic` not `sync/atomic`
- Use `github.com/grafana/regexp` not `regexp`
- Use `github.com/go-kit/log` not `github.com/go-kit/kit/log`
- Use `errors` not `github.com/pkg/errors` (in migrated packages)

## Import Organization

**Order (enforced by gci formatter):**
1. Standard library imports
2. Third-party imports
3. Internal Mimir imports (`github.com/grafana/mimir/...`)

**Example from `pkg/distributor/distributor.go`:**
```go
import (
    "context"
    "flag"
    "fmt"

    "github.com/go-kit/log"
    "github.com/grafana/dskit/concurrency"
    "github.com/prometheus/client_golang/prometheus"

    "github.com/grafana/mimir/pkg/ingester/client"
    "github.com/grafana/mimir/pkg/mimirpb"
)
```

**Path Aliases:**
- Used sparingly for clarity: `ring_client "github.com/grafana/dskit/ring/client"`
- Common pattern: `util_test "github.com/grafana/mimir/pkg/util/test"`

## Error Handling

**Patterns:**
- Use `errors.New()` for simple errors
- Use `fmt.Errorf()` with `%w` for wrapping (when needed)
- Use `github.com/pkg/errors` in legacy code, `errors` in new code
- Never use `errors.Cause()` - behavior differs between pkg/errors and stdlib

**User-Facing Errors:**
- Define in `pkg/util/globalerror/user.go`
- Use immutable error IDs (never rename after release)
- Format with `ID.Message()` or `ID.MessageWithPerTenantLimitConfig()`
- Example: `globalerror.MaxLabelNamesPerSeries.Message("...")`

**Error Classification:**
```go
// pkg/util/globalerror/user.go
type ID string

const (
    MissingMetricName ID = "missing-metric-name"
    MaxLabelNamesPerSeries ID = "max-label-names-per-series"
    // ... more IDs
)

func (id ID) Message(msg string) string {
    return fmt.Sprintf("%s%s: %s", errPrefix, id, msg)
}
```

## Logging

**Framework:** `github.com/go-kit/log`

**Patterns:**
- Create logger in component constructor
- Use structured logging with key-value pairs
- Log levels via `level.Debug()`, `level.Info()`, `level.Warn()`, `level.Error()`

```go
import "github.com/go-kit/log/level"

level.Error(d.log).Log("msg", "failed to push", "err", err, "user", userID)
```

## Comments

**When to Comment:**
- Public APIs: document purpose, parameters, return values
- Complex algorithms: explain the "why"
- Non-obvious behavior: document edge cases

**Style:**
- Comment sentences begin with the name of the thing being described
- Comment sentences end with a period
- Example: `// NewDistributor creates a new distributor instance.`

## Function Design

**Size:** Keep functions focused on single responsibility; extract helpers for complex logic

**Parameters:**
- Config structs for multiple options
- Accept interfaces, return concrete types
- Context as first parameter when needed

**Return Values:**
- Return errors as last value
- Use named returns sparingly, only when it improves clarity

## Module Design

**Exports:**
- Export only what's needed by other packages
- Use internal packages for implementation details

**Barrel Files:**
- Not commonly used; direct imports preferred

## Prometheus Metrics

**Registration Pattern:**
```go
// Never use global variables for metrics
// Create and register with promauto.With(reg)
// Take registerer as constructor parameter

func NewDistributor(reg prometheus.Registerer, ...) *Distributor {
    return &Distributor{
        receivedSamples: promauto.With(reg).NewCounterVec(prometheus.CounterOpts{
            Name: "cortex_distributor_received_samples_total",
            Help: "The total number of received samples.",
        }, []string{"user", "type"}),
    }
}
```

**Testing Metrics:**
```go
// Use testutil.GatherAndCompare() for metric assertions
require.NoError(t, testutil.GatherAndCompare(reg, strings.NewReader(expectedMetrics), "metric_name"))
```

## Configuration Conventions

**Config File Options:**
- Lowercase with underscore separation: `memcached_client`

**CLI Flags:**
- Lowercase with dash separation: `-memcached-client`
- Always prefixed with single dash in documentation

**Flag Registration:**
```go
func (c *Config) RegisterFlags(f *flag.FlagSet) {
    f.IntVar(&c.ReplicationFactor, "distributor.replication-factor", 3, "...")
}
```

## Memory Safety

**Critical Warning - Unsafe String Handling:**
- Buffer pools are shared between requests
- `yoloString` / `unsafeMutableString` types indicate unsafe memory references
- Strings from `PreallocWriteRequest` must not outlive the request handler
- Use `strings.Clone()` for strings that must persist beyond handler scope
- `slices.Clone()` does NOT deep-copy string contents

**Safe Patterns:**
```go
// BAD: string reference may become invalid
errorMsg := timeseries.Labels[0].Name

// GOOD: deep copy the string
errorMsg := strings.Clone(timeseries.Labels[0].Name)
```

---

*Convention analysis: 2026-02-23*
