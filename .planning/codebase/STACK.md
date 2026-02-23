# Technology Stack

**Analysis Date:** 2026-02-23

## Languages

**Primary:**
- Go 1.25.7 - Main application language for all components

**Secondary:**
- Jsonnet - Used for Kubernetes manifests and mixin configuration (`operations/mimir-mixin/`, `operations/mimir/`)
- Protocol Buffers (proto3) - gRPC service definitions and internal data serialization (`pkg/mimirpb/`, `pkg/storegateway/storepb/`)
- JavaScript/Node.js - Tooling for documentation screenshots (`operations/mimir-mixin-tools/screenshots/`)

## Runtime

**Environment:**
- Go runtime (self-contained binary)
- Linux containers (Docker/Kubernetes deployment)

**Package Manager:**
- Go modules
- Lockfile: `go.sum` (present)
- Vendor directory: `vendor/` (present, committed)

## Frameworks

**Core:**
- `github.com/grafana/dskit` v0.0.0-20260218 - Distributed systems toolkit (service lifecycle, ring, memberlist, gRPC utilities)
- `github.com/prometheus/prometheus` v1.99.0 (forked as `mimir-prometheus`) - PromQL engine, TSDB, remote write/read
- `github.com/prometheus/alertmanager` v0.31.0 (forked as `prometheus-alertmanager`) - Alerting functionality
- `github.com/gorilla/mux` v1.8.1 - HTTP routing

**Testing:**
- `github.com/stretchr/testify` v1.11.1 - Assertions and mocking
- `github.com/grafana/e2e` v0.1.2 - End-to-end integration testing framework
- `go.uber.org/goleak` v1.3.0 - Goroutine leak detection

**Build/Dev:**
- Docker/Docker Compose - Local development and CI
- Make - Build automation (`Makefile`)
- golangci-lint v2.8.0 - Linting with custom rules (`.golangci.yml`)
- goimports - Code formatting with `-local github.com/grafana/mimir`
- Renovate - Automated dependency updates (`renovate.json5`)

## Key Dependencies

**Critical:**
- `github.com/thanos-io/objstore` v0.0.0-20250813 - Object storage abstraction (S3, GCS, Azure, Swift)
- `google.golang.org/grpc` v1.79.1 - Inter-component RPC
- `github.com/gogo/protobuf` v1.3.2 - Protocol Buffers with gogofast optimizations
- `github.com/twmb/franz-go` v1.20.6 - Kafka client for ingest storage

**Infrastructure:**
- `github.com/hashicorp/memberlist` v0.3.1 (forked) - Gossip protocol for service discovery
- `github.com/grafana/gomemcache` v0.0.0-20251127 - Memcached client for caching
- `go.etcd.io/etcd/client/v3` v3.6.8 - etcd client for KV store
- `github.com/hashicorp/consul/api` v1.33.2 - Consul client for service discovery

**Observability:**
- `github.com/prometheus/client_golang` v1.23.3 - Prometheus metrics
- `go.opentelemetry.io/otel` v1.40.0 - OpenTelemetry tracing
- `github.com/go-kit/log` v0.2.1 - Structured logging

**Compression:**
- `github.com/klauspost/compress` v1.18.4 - High-performance compression (zstd, gzip, snappy)
- `github.com/golang/snappy` v1.0.0 - Snappy compression
- `github.com/pierrec/lz4/v4` v4.1.25 - LZ4 compression

## Configuration

**Environment:**
- YAML config files with `-config.file` flag
- CLI flags for all options
- Environment variable expansion with `${VAR}` or `${VAR:default}` syntax
- Runtime config file for dynamic per-tenant overrides

**Key Config Files:**
- `cmd/mimir/main.go` - Configuration loading and flag registration
- `pkg/mimir/mimir.go` - Root `Config` struct with all component configs

**Build:**
- `Makefile` - Primary build orchestration
- `mimir-build-image/Dockerfile` - Docker build image with all dev dependencies
- `.golangci.yml` - Linter configuration

## Platform Requirements

**Development:**
- Go 1.25.7+
- Docker (for build image and integration tests)
- Make
- gnu-sed on macOS (`brew install gnu-sed`)

**Production:**
- Linux (amd64 or arm64)
- Container runtime (Docker/containerd/Kubernetes)
- Object storage (S3, GCS, Azure Blob, Swift, or local filesystem)
- Optional: Kafka cluster for ingest storage
- Optional: Memcached/Redis for caching
- Optional: Consul/etcd for service discovery (or memberlist for gossip)

## Binaries Produced

| Binary | Location | Purpose |
|--------|----------|---------|
| `mimir` | `cmd/mimir/` | Main Mimir server (all components) |
| `mimirtool` | `cmd/mimirtool/` | CLI for managing Mimir (rules, alertmanager configs, etc.) |
| `query-tee` | `cmd/query-tee/` | Query comparison proxy for testing |
| `metaconvert` | `cmd/metaconvert/` | Metadata conversion utility |

## Docker Images

- `grafana/mimir` - Main production image
- `grafana/mimir-build-image` - Build toolchain image
- `grafana/mimirtool` - CLI tool image
- `grafana/query-tee` - Query tee image

---

*Stack analysis: 2026-02-23*
