# Testing Patterns

**Analysis Date:** 2026-02-23

## Test Framework

**Runner:**
- Go standard `testing` package
- No external test runner

**Assertion Libraries:**
- Primary: `github.com/stretchr/testify/require` (fails immediately)
- Secondary: `github.com/stretchr/testify/assert` (continues on failure)
- Metrics: `github.com/prometheus/client_golang/prometheus/testutil`

**Run Commands:**
```bash
make test                                    # Run unit tests (in container)
go test ./pkg/...                            # Run unit tests locally
go test -v -tags=requires_docker ./integration/...  # Run integration tests
go test -run "^TestDistributor_Push$" ./pkg/distributor/...  # Single test
go test -timeout=20m -tags=requires_docker ./integration/...  # With timeout
```

## Test File Organization

**Location:**
- Co-located with source: `distributor.go` + `distributor_test.go`
- Same package for internal access
- Integration tests in separate `/integration/` directory

**Naming:**
- Unit tests: `<source>_test.go`
- Integration tests: `<component>_test.go` in `/integration/`
- Mock files: `*_mock.go` or `*_mock_test.go`

**Structure:**
```
pkg/distributor/
  distributor.go
  distributor_test.go
  distributor_ingest_storage_test.go
  validate.go
  validate_test.go

integration/
  distributor_test.go
  ingester_test.go
  querier_test.go
```

## Test Structure

**Suite Organization (table-driven tests):**
```go
func TestDistributor_Push(t *testing.T) {
    for name, tc := range map[string]struct {
        numIngesters   int
        happyIngesters int
        samples        int
        expectedError  string
    }{
        "A push to 3 happy ingesters should succeed": {
            numIngesters:   3,
            happyIngesters: 3,
            samples:        5,
        },
        "A push to 0 happy ingesters should fail": {
            numIngesters:   3,
            happyIngesters: 0,
            samples:        10,
            expectedError:  "failed pushing to ingester",
        },
    } {
        t.Run(name, func(t *testing.T) {
            // Test implementation
            if tc.expectedError != "" {
                require.ErrorContains(t, err, tc.expectedError)
            } else {
                require.NoError(t, err)
            }
        })
    }
}
```

**Config Setup Pattern:**
```go
func defaultIngesterTestConfig(t testing.TB) Config {
    t.Helper()

    consul, closer := consul.NewInMemoryClient(ring.GetCodec(), log.NewNopLogger(), nil)
    t.Cleanup(func() { assert.NoError(t, closer.Close()) })

    cfg := Config{}
    flagext.DefaultValues(&cfg)
    // Configure test-specific values
    return cfg
}

func defaultLimitsTestConfig() validation.Limits {
    limits := validation.Limits{}
    flagext.DefaultValues(&limits)
    return limits
}
```

**Cleanup Pattern:**
```go
t.Cleanup(func() {
    require.NoError(t, service.Stop())
})
```

## Mocking

**Framework:** `github.com/stretchr/testify/mock`

**Mock Definition Pattern:**
```go
// pkg/storage/bucket/client_mock.go
type ClientMock struct {
    mock.Mock
}

func (m *ClientMock) Upload(ctx context.Context, name string, r io.Reader) error {
    args := m.Called(ctx, name, r)
    return args.Error(0)
}

// Convenience method for common setup
func (m *ClientMock) MockUpload(name string, err error) {
    m.On("Upload", mock.Anything, name, mock.Anything).Return(err)
}
```

**Mock Usage:**
```go
func TestSomething(t *testing.T) {
    mockBucket := &bucket.ClientMock{}
    mockBucket.MockUpload("file.txt", nil)

    // Use mockBucket in test

    mockBucket.AssertExpectations(t)
}
```

**What to Mock:**
- External services (object storage, Kafka, Consul)
- gRPC clients/servers
- Time-dependent operations (use `github.com/grafana/dskit/mtime`)
- Ring membership

**What NOT to Mock:**
- Internal business logic
- Validation functions
- Config parsing

## Fixtures and Factories

**Test Data Generators:**
```go
// pkg/util/test/histogram.go
func GenerateTestHistogram(i int) *histogram.Histogram {
    return tsdbutil.GenerateTestHistogram(int64(i))
}

func GenerateTestFloatHistogram(i int) *histogram.FloatHistogram {
    return tsdbutil.GenerateTestFloatHistogram(int64(i))
}
```

**Usage:**
```go
import util_test "github.com/grafana/mimir/pkg/util/test"

hist := util_test.GenerateTestHistogram(1)
```

**Location:**
- Shared test utilities: `pkg/util/test/`
- Package-specific helpers: within `*_test.go` files

## Coverage

**Requirements:** No enforced coverage threshold

**View Coverage:**
```bash
go test -coverprofile=coverage.out ./pkg/...
go tool cover -html=coverage.out
```

## Test Types

**Unit Tests:**
- Located in `pkg/` alongside source
- Test individual functions/methods
- Use mocks for external dependencies
- Run with: `go test ./pkg/...`

**Integration Tests:**
- Located in `/integration/`
- Require Docker (use `requires_docker` build tag)
- Test full component interaction
- Run Mimir in Docker containers
- Based on `github.com/grafana/e2e` framework

**Integration Test Build Tag:**
```go
// SPDX-License-Identifier: AGPL-3.0-only
//go:build requires_docker

package integration
```

## Integration Test Framework

**Framework:** `github.com/grafana/e2e`

**Service Creation:**
```go
// integration/e2emimir/services.go
func NewDistributor(name string, consulAddress string, flags map[string]string, options ...Option) *MimirService {
    return newMimirServiceFromOptions(
        name,
        map[string]string{
            "-target":                    "distributor",
            "-log.level":                 "warn",
            "-auth.multitenancy-enabled": "true",
            // ...more flags
        },
        flags,
        options...,
    )
}
```

**Test Pattern:**
```go
func TestDistributor(t *testing.T) {
    s, err := e2e.NewScenario(networkName)
    require.NoError(t, err)
    defer s.Close()

    consul := e2edb.NewConsul()
    require.NoError(t, s.StartAndWaitReady(consul))

    distributor := e2emimir.NewDistributor("distributor", consul.NetworkHTTPEndpoint(), nil)
    require.NoError(t, s.StartAndWaitReady(distributor))

    // Test assertions
}
```

**Environment Variables:**
- `MIMIR_IMAGE` - Docker image for Mimir (default: `grafana/mimir:latest`)
- `MIMIRTOOL_IMAGE` - Docker image for mimirtool
- `MIMIR_CHECKOUT_DIR` - Repository checkout path
- `E2E_TEMP_DIR` - Temp directory for test files
- `E2E_NETWORK_NAME` - Docker network name (default: `e2e-mimir-test`)

## Common Patterns

**Async Testing:**
```go
// Use dskit test.Poll for polling
import "github.com/grafana/dskit/test"

test.Poll(t, 100*time.Millisecond, expectedValue, func() interface{} {
    return actualValue()
})
```

**Error Testing:**
```go
// Check error contains expected message
require.ErrorContains(t, err, "expected message")

// Check specific error type
require.ErrorIs(t, err, expectedError)

// Check no error
require.NoError(t, err)
```

**Metric Testing:**
```go
import "github.com/prometheus/client_golang/prometheus/testutil"

// Compare full metric output
require.NoError(t, testutil.GatherAndCompare(reg, strings.NewReader(`
    # HELP cortex_distributor_received_samples_total The total number of received samples.
    # TYPE cortex_distributor_received_samples_total counter
    cortex_distributor_received_samples_total{user="user"} 5
`), "cortex_distributor_received_samples_total"))

// Get single metric value
value := testutil.ToFloat64(metric)
require.Equal(t, float64(5), value)
```

**Helper Functions:**
```go
func verifyUtilizationLimitedRequestsMetric(t *testing.T, reg *prometheus.Registry) {
    t.Helper()  // Mark as helper for better error reporting

    const expMetrics = `
        # HELP cortex_ingester_utilization_limited_read_requests_total Total number...
        # TYPE cortex_ingester_utilization_limited_read_requests_total counter
        cortex_ingester_utilization_limited_read_requests_total{reason="cpu"} 1
    `
    assert.NoError(t, testutil.GatherAndCompare(reg, strings.NewReader(expMetrics),
        "cortex_ingester_utilization_limited_read_requests_total"))
}
```

**Context with Tenant ID:**
```go
import "github.com/grafana/dskit/user"

ctx := user.InjectOrgID(context.Background(), "test-tenant")
```

**Time Control:**
```go
import "github.com/grafana/dskit/mtime"

now := time.Now()
mtime.NowForce(now)
t.Cleanup(mtime.NowReset)
```

## Test Helpers Location

**Shared Utilities:**
- `pkg/util/test/histogram.go` - Histogram generators
- `pkg/util/validation/limits_mock.go` - Mock limits
- `pkg/storage/bucket/client_mock.go` - Mock bucket client

**Integration Test Helpers:**
- `integration/e2emimir/services.go` - Service factories
- `integration/e2emimir/client.go` - HTTP/gRPC clients
- `integration/asserts.go` - Common assertions

---

*Testing analysis: 2026-02-23*
