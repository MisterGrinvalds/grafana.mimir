# External Integrations

**Analysis Date:** 2026-02-23

## APIs & External Services

**Prometheus Remote Write:**
- Prometheus Remote Write 1.0 and 2.0 protocol
- Implementation: `pkg/distributor/push.go`, `pkg/distributor/http_server.go`
- Auth: Via HTTP headers (X-Scope-OrgId for tenant)
- Endpoint: `/api/v1/push`

**OpenTelemetry (OTLP):**
- OTLP metrics ingestion (protobuf and JSON)
- Implementation: `pkg/distributor/otel.go`
- SDK: `go.opentelemetry.io/collector/pdata` v1.51.0
- Endpoint: `/otlp/v1/metrics`

**InfluxDB Line Protocol:**
- InfluxDB write compatibility
- Implementation: `pkg/distributor/influx.go`
- SDK: `github.com/influxdata/influxdb/v2` v2.8.0
- Endpoint: `/api/v1/push/influx/write`

**Prometheus Query API:**
- Full Prometheus HTTP API compatibility
- Implementation: `pkg/api/api.go`, `pkg/querier/`
- Endpoints: `/prometheus/api/v1/query`, `/prometheus/api/v1/query_range`, etc.

## Data Storage

**Object Storage (Blocks Storage):**
- Configuration: `pkg/storage/bucket/client.go`
- Client: `github.com/thanos-io/objstore`

Supported backends:
| Backend | Package | Config Prefix |
|---------|---------|---------------|
| AWS S3 | `pkg/storage/bucket/s3/` | `-blocks-storage.s3.*` |
| Google Cloud Storage | `pkg/storage/bucket/gcs/` | `-blocks-storage.gcs.*` |
| Azure Blob Storage | `pkg/storage/bucket/azure/` | `-blocks-storage.azure.*` |
| OpenStack Swift | `pkg/storage/bucket/swift/` | `-blocks-storage.swift.*` |
| Local Filesystem | `pkg/storage/bucket/filesystem/` | `-blocks-storage.filesystem.*` |

**Kafka (Ingest Storage):**
- Configuration: `pkg/storage/ingest/config.go`
- Client: `github.com/twmb/franz-go` v1.20.6
- Purpose: Alternative ingestion path via Kafka for durability
- Auth: SASL (PLAIN, SCRAM-SHA-256, SCRAM-SHA-512, OAUTHBEARER)
- Config prefix: `-ingest-storage.kafka.*`

**Caching:**
- Memcached: `github.com/grafana/gomemcache`
  - Used for: Index cache, chunks cache, query results cache
  - Configuration: `pkg/storage/tsdb/indexcache/remote.go`
- Redis: Supported via same caching interfaces
  - Configuration: `pkg/storage/tsdb/indexcache/config.go`

## Service Discovery & KV Store

**Memberlist (Gossip):**
- Implementation: `github.com/hashicorp/memberlist` (forked as `github.com/grafana/memberlist`)
- Purpose: Ring membership, KV store for distributed state
- Configuration: `pkg/mimir/mimir.go` -> `MemberlistKV`
- Default and recommended for production

**Consul:**
- Client: `github.com/hashicorp/consul/api` v1.33.2
- Purpose: Alternative KV store for ring membership
- Configuration: Via dskit ring config

**etcd:**
- Client: `go.etcd.io/etcd/client/v3` v3.6.8
- Purpose: Alternative KV store for ring membership
- Configuration: Via dskit ring config

## Secrets Management

**HashiCorp Vault:**
- Implementation: `pkg/vault/vault.go`
- Client: `github.com/hashicorp/vault/api` v1.22.0
- Purpose: Fetch secrets (TLS certificates, credentials)
- Auth methods: AppRole, Kubernetes, UserPass
- Configuration: `-vault.*` flags

## Alerting Integrations

**Notification Channels (via Alertmanager):**
- Implementation: `pkg/alertmanager/alertmanager.go`
- SDK: `github.com/prometheus/alertmanager` (forked)

Supported receivers:
- Discord
- Email (SMTP)
- Microsoft Teams (v1 and v2)
- OpsGenie
- PagerDuty
- Pushover
- Slack
- SNS (AWS)
- Telegram
- VictorOps
- Webex
- Webhook (generic HTTP)
- WeChat

**Grafana Alerting:**
- SDK: `github.com/grafana/alerting`
- Purpose: Extended receiver types compatible with Grafana Alerting

## Observability

**Prometheus Metrics:**
- Self-instrumentation with extensive metrics
- Prefix: `cortex_*` (for backwards compatibility)
- Registry: `github.com/prometheus/client_golang`
- Native histograms enabled by default (10% bucket growth)

**Distributed Tracing:**
- OpenTelemetry: `go.opentelemetry.io/otel` v1.40.0
- Jaeger (legacy): `github.com/uber/jaeger-client-go`
- Configuration via environment variables (OTEL_SERVICE_NAME, JAEGER_SERVICE_NAME)
- Implementation: `github.com/grafana/dskit/tracing`

**Profiling:**
- pprof endpoints built-in
- fgprof integration: `github.com/felixge/fgprof`
- Pyroscope integration: `github.com/grafana/pyroscope-go/godeltaprof`

## CI/CD & Deployment

**GitHub Actions:**
- Location: `.github/workflows/`
- Workflows: helm-ci, changelog-check, flaky-tests, deploy-pr-preview, etc.

**Container Registry:**
- Docker Hub: `grafana/mimir`, `grafana/mimirtool`, etc.
- Multi-arch builds: linux/amd64, linux/arm64

**Helm Charts:**
- Location: Separate repository (mimir-distributed)
- Tooling: `operations/mimir-mixin-tools/`

**Renovate:**
- Configuration: `renovate.json5`
- Automated dependency updates with Go module handling
- Branch prefixing: `deps-update/`

## Webhooks & Callbacks

**Incoming:**
- Alertmanager webhooks for alert receiving
- Remote write endpoints (Prometheus, OTLP, InfluxDB)

**Outgoing:**
- Alert notifications to configured receivers
- Webhook notifications to arbitrary HTTP endpoints
- Usage statistics reporting (optional, to `stats.grafana.org`)

## Environment Configuration

**Required for basic operation:**
- Object storage credentials (S3 keys, GCS service account, etc.)
- Block storage bucket name/path

**Required for features:**
- Kafka connection (if using ingest storage)
- Memcached/Redis addresses (if using caching)
- Alertmanager receiver credentials (SMTP, Slack tokens, etc.)
- Vault configuration (if using secrets management)

**Secrets (never commit):**
- AWS credentials / GCS service account JSON
- Kafka SASL credentials
- Alertmanager receiver tokens/passwords
- Vault tokens
- TLS private keys

---

*Integration audit: 2026-02-23*
