# Architecture

**Analysis Date:** 2026-02-23

## Pattern Overview

**Overall:** Modular monolith with microservices deployment support

Grafana Mimir is a horizontally scalable, highly available, multi-tenant time series database forked from Cortex. It implements a modular architecture where components can run as a single process (monolithic mode), read-write deployment mode, or as separate microservices.

**Key Characteristics:**
- Module-based architecture using dskit's `modules.Manager` for dependency management
- Ring-based consistent hashing for data distribution and replication
- Multi-tenant isolation with per-tenant limits and configuration
- gRPC for inter-component communication, HTTP for external APIs
- Object storage (S3, GCS, Azure, Swift) as the primary durable storage backend

## Layers

**API Layer:**
- Purpose: HTTP/gRPC entry points for all external and internal requests
- Location: `pkg/api/`
- Contains: Route registration, middleware, authentication handlers
- Depends on: All component modules
- Used by: External clients (Prometheus remote write, PromQL queries, Alertmanager API)

**Distributor Layer:**
- Purpose: Receives incoming write requests, validates, and distributes to ingesters
- Location: `pkg/distributor/`
- Contains: Push handlers, HA deduplication, rate limiting, sharding logic
- Depends on: Ingester ring, validation/overrides, cost attribution
- Used by: API layer (remote write endpoints)

**Ingester Layer:**
- Purpose: In-memory time series storage with WAL, handles recent data writes and queries
- Location: `pkg/ingester/`
- Contains: TSDB management, active series tracking, ring membership
- Depends on: Ring, blocks storage config, validation/overrides
- Used by: Distributor (writes), Querier (recent data queries)

**Querier Layer:**
- Purpose: Executes PromQL queries by merging data from ingesters and store-gateways
- Location: `pkg/querier/`
- Contains: PromQL engine wrapper, queryable implementations, series merging
- Depends on: Distributor (ingester access), store-gateway client, validation/overrides
- Used by: Query-frontend, API layer

**Query-Frontend Layer:**
- Purpose: Query optimization, caching, splitting, and sharding
- Location: `pkg/frontend/`
- Contains: Tripperware middleware chain, query splitting, results caching
- Depends on: Query-scheduler discovery, codec, overrides
- Used by: External PromQL query clients

**Query-Scheduler Layer:**
- Purpose: Distributes query load across querier workers
- Location: `pkg/scheduler/`
- Contains: Queue management, tenant fairness, querier connection handling
- Depends on: Ring (optional), overrides
- Used by: Query-frontend, querier workers

**Store-Gateway Layer:**
- Purpose: Serves queries from long-term block storage
- Location: `pkg/storegateway/`
- Contains: Block syncing, index-header caching, bucket store
- Depends on: Ring, blocks storage, overrides
- Used by: Querier (historical data queries)

**Compactor Layer:**
- Purpose: Compacts and deduplicates blocks in object storage
- Location: `pkg/compactor/`
- Contains: Compaction planning, block deletion, tenant cleanup
- Depends on: Ring, blocks storage, overrides
- Used by: Background process (scheduled compaction)

**Ruler Layer:**
- Purpose: Evaluates recording and alerting rules
- Location: `pkg/ruler/`
- Contains: Rule group management, query execution, Alertmanager notification
- Depends on: Querier (for rule evaluation), ruler storage, distributor
- Used by: Background process (rule evaluation loop)

**Alertmanager Layer:**
- Purpose: Handles alert deduplication, grouping, routing, and notification
- Location: `pkg/alertmanager/`
- Contains: Multi-tenant alertmanager, state replication, UI
- Depends on: Ring, alertmanager storage, overrides
- Used by: Ruler (alert notifications), external alert API clients

## Data Flow

**Write Path (Remote Write):**

1. HTTP POST to `/api/v1/push` received by distributor
2. Distributor validates request against tenant limits
3. Distributor performs HA deduplication (if configured)
4. Series sharded across ingesters via consistent hash ring
5. Ingesters write to in-memory TSDB with WAL
6. Ingesters periodically flush blocks to object storage

**Ingest Storage Path (Kafka-based):**

1. HTTP POST to `/api/v1/push` received by distributor
2. Distributor writes to Kafka topic partitions
3. Ingester consumes from assigned Kafka partitions
4. Ingester writes to in-memory TSDB
5. Block-builder (optional) creates blocks from Kafka data

**Query Path (PromQL):**

1. HTTP GET/POST to `/prometheus/api/v1/query` received by query-frontend
2. Query-frontend applies middleware: splitting, sharding, caching
3. Request forwarded to query-scheduler
4. Query-scheduler assigns to available querier worker
5. Querier executes via PromQL engine against merged queryable
6. Queryable fans out to ingesters (recent) and store-gateways (historical)
7. Results merged and returned through the chain

**State Management:**

- **Ring state:** Memberlist gossip or etcd/consul for service discovery
- **Runtime config:** Optional file-based config reloading for per-tenant limits
- **Tenant limits:** `validation.Overrides` wrapping static + runtime config

## Key Abstractions

**Ring (`ring.Ring`):**
- Purpose: Consistent hashing for data distribution and service discovery
- Examples: `pkg/ingester/ring.go`, `pkg/storegateway/ring.go`, `pkg/distributor/distributor_ring.go`
- Pattern: Each stateful component maintains ring membership; clients query ring for instance selection

**Services (`services.Service`):**
- Purpose: Lifecycle management for all components
- Examples: All init* methods in `pkg/mimir/modules.go`
- Pattern: dskit services framework with state machine (New -> Starting -> Running -> Stopping -> Terminated)

**Queryable (`storage.Queryable`):**
- Purpose: Prometheus storage interface for query execution
- Examples: `pkg/querier/distributor_queryable.go`, `pkg/querier/blocks_store_queryable.go`
- Pattern: Composable queryables merged for unified query interface

**Limits/Overrides (`validation.Overrides`):**
- Purpose: Per-tenant configuration and rate limiting
- Examples: `pkg/util/validation/limits.go`, `pkg/util/validation/validate.go`
- Pattern: Static defaults + optional runtime config file for dynamic per-tenant overrides

**Module Manager (`modules.Manager`):**
- Purpose: Dependency injection and initialization ordering for components
- Examples: `pkg/mimir/modules.go` - `setupModuleManager()`
- Pattern: Modules registered with init functions; dependencies declared; manager initializes in correct order

## Entry Points

**Main Binary (`cmd/mimir/main.go`):**
- Location: `cmd/mimir/main.go`
- Triggers: CLI invocation
- Responsibilities: Config parsing, module initialization via `mimir.New()`, service lifecycle

**Mimir Struct (`pkg/mimir/mimir.go`):**
- Location: `pkg/mimir/mimir.go`
- Triggers: `mimir.New()` from main
- Responsibilities: Root config holder, module manager setup, service orchestration

**Module Initialization (`pkg/mimir/modules.go`):**
- Location: `pkg/mimir/modules.go`
- Triggers: `ModuleManager.InitModuleServices()` during startup
- Responsibilities: Per-component init functions, dependency wiring, API registration

**Distributor Push Handler:**
- Location: `pkg/distributor/push.go`, `pkg/distributor/distributor.go`
- Triggers: HTTP `/api/v1/push` (Prometheus remote write)
- Responsibilities: Request decompression, validation, ingester distribution

**Query Frontend Handler:**
- Location: `pkg/frontend/transport/handler.go`
- Triggers: HTTP `/prometheus/api/v1/query`, `/prometheus/api/v1/query_range`
- Responsibilities: Query parsing, middleware application, backend dispatch

## Error Handling

**Strategy:** Structured errors with global error catalog for user-facing messages

**Patterns:**
- `pkg/util/globalerror/` - Error IDs for documentation linking
- `github.com/pkg/errors` for error wrapping with context
- gRPC status codes for inter-component errors
- HTTP status codes with JSON error bodies for external APIs
- Per-tenant error responses with tenant ID context

## Cross-Cutting Concerns

**Logging:**
- go-kit/log throughout
- Rate-limited loggers for high-volume paths
- Span loggers for tracing context

**Validation:**
- Request validation in `pkg/util/validation/`
- Per-tenant limits via `validation.Overrides`
- Series/label validation before ingestion

**Authentication:**
- Tenant ID extraction via `X-Scope-OrgId` header
- `pkg/util/noauth/` for single-tenant/anonymous mode
- Middleware-based auth in `pkg/api/`

**Metrics:**
- Prometheus client_golang throughout
- `promauto.With(reg)` pattern for scoped registration
- Metrics namespaced with `cortex_` prefix

**Tracing:**
- OpenTelemetry support
- Span creation in hot paths
- Context propagation through gRPC/HTTP

---

*Architecture analysis: 2026-02-23*
