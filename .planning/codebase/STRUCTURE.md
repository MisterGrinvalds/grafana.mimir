# Codebase Structure

**Analysis Date:** 2026-02-23

## Directory Layout

```
mimir/
├── cmd/                    # Main entry points for binaries
│   ├── mimir/              # Main Mimir binary
│   ├── mimirtool/          # CLI tool for Mimir operations
│   ├── metaconvert/        # Block metadata conversion tool
│   └── query-tee/          # Query comparison proxy
├── pkg/                    # Core Go packages
│   ├── mimir/              # Root application wiring
│   ├── distributor/        # Write path entry point
│   ├── ingester/           # In-memory TSDB storage
│   ├── querier/            # Query execution
│   ├── frontend/           # Query-frontend optimization
│   ├── scheduler/          # Query-scheduler
│   ├── storegateway/       # Block storage queries
│   ├── compactor/          # Block compaction
│   ├── ruler/              # Rule evaluation
│   ├── alertmanager/       # Alert handling
│   ├── blockbuilder/       # Kafka-to-blocks (ingest storage)
│   ├── api/                # HTTP/gRPC API registration
│   ├── mimirpb/            # Protobuf definitions
│   ├── storage/            # Storage abstractions
│   ├── streamingpromql/    # MQE (Mimir Query Engine)
│   └── util/               # Shared utilities
├── integration/            # End-to-end integration tests
├── operations/             # Deployment configs and mixins
│   ├── helm/               # Helm charts
│   ├── mimir/              # Jsonnet deployment
│   └── mimir-mixin/        # Dashboards and alerts
├── tools/                  # Development and debugging tools
├── development/            # Local development setup
├── docs/                   # Documentation sources
└── vendor/                 # Vendored dependencies
```

## Directory Purposes

**`cmd/`:**
- Purpose: Binary entry points with main packages
- Contains: `main.go` files, minimal bootstrap logic
- Key files: `cmd/mimir/main.go` (primary binary)

**`pkg/mimir/`:**
- Purpose: Application root - config, modules, lifecycle
- Contains: Config struct, module manager setup, service orchestration
- Key files:
  - `mimir.go` - Root `Mimir` struct and `New()` constructor
  - `modules.go` - Module registration and dependencies
  - `runtime_config.go` - Runtime config loading

**`pkg/distributor/`:**
- Purpose: Write path - receives, validates, distributes samples
- Contains: Push handlers, HA tracking, Kafka writer, validation
- Key files:
  - `distributor.go` - Main Distributor struct
  - `push.go` - HTTP push handler
  - `ha_tracker.go` - HA replica deduplication

**`pkg/ingester/`:**
- Purpose: In-memory TSDB management, ring membership
- Contains: TSDB lifecycle, WAL, active series tracking, gRPC handlers
- Key files:
  - `ingester.go` - Main Ingester struct
  - `user_tsdb.go` - Per-tenant TSDB management
  - `activeseries/` - Active series tracking

**`pkg/querier/`:**
- Purpose: Query execution, queryable implementations
- Contains: PromQL engine config, queryables, workers
- Key files:
  - `querier.go` - Querier setup and config
  - `distributor_queryable.go` - Ingester queries
  - `blocks_store_queryable.go` - Store-gateway queries

**`pkg/frontend/`:**
- Purpose: Query-frontend with middleware chain
- Contains: Tripperware, query splitting, caching, sharding
- Key files:
  - `frontend.go` - Frontend initialization
  - `querymiddleware/` - Middleware implementations
  - `transport/handler.go` - HTTP handler

**`pkg/scheduler/`:**
- Purpose: Query scheduling and load distribution
- Contains: Queue management, querier connections
- Key files:
  - `scheduler.go` - Scheduler service
  - `queue/` - Request queue implementation

**`pkg/storegateway/`:**
- Purpose: Serves queries from object storage blocks
- Contains: Block syncing, bucket stores, index headers
- Key files:
  - `gateway.go` - StoreGateway struct
  - `bucket_stores.go` - Multi-tenant bucket stores

**`pkg/compactor/`:**
- Purpose: Block compaction in object storage
- Contains: Compaction jobs, cleanup, deletion
- Key files:
  - `compactor.go` - MultitenantCompactor struct
  - `split_merge_job.go` - Split/merge compaction

**`pkg/ruler/`:**
- Purpose: Recording and alerting rule evaluation
- Contains: Rule manager, Alertmanager client
- Key files:
  - `ruler.go` - Ruler struct
  - `manager.go` - Rule group manager

**`pkg/alertmanager/`:**
- Purpose: Multi-tenant Alertmanager
- Contains: Alert routing, notification, state sync
- Key files:
  - `multitenant.go` - MultitenantAlertmanager struct
  - `alertstore/` - Configuration storage

**`pkg/api/`:**
- Purpose: HTTP/gRPC API registration and routing
- Contains: Route handlers, middleware registration
- Key files:
  - `api.go` - API struct and route registration
  - `handlers.go` - Prometheus API handlers

**`pkg/mimirpb/`:**
- Purpose: Protobuf definitions for internal communication
- Contains: Generated .pb.go files, custom codecs
- Key files:
  - `mimir.proto` (source in vendor) -> `mimir.pb.go`
  - `custom_protobuf_codec.go` - Optimized codec

**`pkg/storage/`:**
- Purpose: Storage abstractions and implementations
- Contains: Bucket clients, TSDB utilities, ingest storage
- Key files:
  - `bucket/client.go` - Object storage client factory
  - `tsdb/config.go` - TSDB configuration
  - `ingest/` - Kafka-based ingest storage

**`pkg/streamingpromql/`:**
- Purpose: Mimir Query Engine (MQE) - streaming PromQL
- Contains: Query planning, operators, optimization
- Key files:
  - `engine.go` - MQE engine
  - `operators/` - PromQL operators
  - `planning/` - Query plan generation

**`pkg/util/`:**
- Purpose: Shared utilities across packages
- Contains: Validation, logging, metrics helpers
- Key files:
  - `validation/limits.go` - Per-tenant limits
  - `validation/validate.go` - Sample validation
  - `globalerror/` - Error catalog

**`integration/`:**
- Purpose: End-to-end integration tests
- Contains: Test scenarios using Docker containers
- Key files:
  - `e2emimir/` - Test helpers and fixtures
  - `*_test.go` - Integration test files

**`operations/`:**
- Purpose: Production deployment configurations
- Contains: Jsonnet, Helm, dashboards, alerts
- Key files:
  - `mimir/` - Jsonnet deployment manifests
  - `helm/` - Helm chart
  - `mimir-mixin/` - Grafana dashboards and Prometheus alerts

## Key File Locations

**Entry Points:**
- `cmd/mimir/main.go`: Main binary entry point
- `cmd/mimirtool/main.go`: CLI tool entry point

**Configuration:**
- `pkg/mimir/mimir.go`: Root `Config` struct with all component configs
- `pkg/util/validation/limits.go`: Per-tenant limits defaults
- `docs/configurations/`: Example config files

**Core Logic:**
- `pkg/mimir/modules.go`: Module registration and wiring
- `pkg/distributor/distributor.go`: Write path core
- `pkg/querier/querier.go`: Query path core
- `pkg/ingester/ingester.go`: Storage core

**Testing:**
- `integration/`: E2E tests with Docker
- `*_test.go`: Unit tests co-located with source

**Protobuf:**
- `pkg/mimirpb/`: Generated Go code
- `pkg/*/protobuf/`: Component-specific protos

## Naming Conventions

**Files:**
- Snake_case for Go files: `distributor.go`, `ha_tracker.go`
- `*_test.go` suffix for tests: `distributor_test.go`
- `*pb/` suffix for protobuf packages: `mimirpb/`, `schedulerpb/`

**Directories:**
- Lowercase, single words preferred: `distributor/`, `querier/`
- Compound words without separator: `storegateway/`, `blockbuilder/`

**Packages:**
- Match directory name: `package distributor`
- Internal subpackages: `distributor/distributorpb`

**Types:**
- PascalCase for exported: `Distributor`, `Config`
- Interfaces often suffixed: `RingClient`, `BlocksUploader`

**Constants:**
- camelCase for internal: `distributorRingKey`
- PascalCase for exported: `IngesterRingKey`

## Where to Add New Code

**New Feature (component):**
- Primary code: `pkg/<component>/`
- Tests: `pkg/<component>/*_test.go`
- Module registration: `pkg/mimir/modules.go`
- API endpoints: `pkg/api/api.go`

**New Component/Module:**
- Implementation: `pkg/<component>/<component>.go`
- Config: Add to `pkg/mimir/mimir.go` Config struct
- Module: Register in `pkg/mimir/modules.go`
- Integration test: `integration/<component>_test.go`

**New API Endpoint:**
- Handler: `pkg/api/api.go` or component-specific registration
- Protobuf (if gRPC): `pkg/<component>/<component>pb/`

**New Limit/Validation:**
- Limit definition: `pkg/util/validation/limits.go`
- Validation logic: `pkg/util/validation/validate.go`
- Error message: `pkg/util/globalerror/`

**Utilities:**
- Shared helpers: `pkg/util/`
- Component-specific: `pkg/<component>/`

**Configuration Options:**
- Add to relevant Config struct in component
- Register flags in `RegisterFlags()` method
- Update `docs/` if user-facing

## Special Directories

**`vendor/`:**
- Purpose: Vendored Go dependencies
- Generated: Yes (via `go mod vendor`)
- Committed: Yes

**`.git/`:**
- Purpose: Git version control
- Generated: Yes
- Committed: N/A (git internal)

**`development/`:**
- Purpose: Local development Docker Compose setups
- Generated: Partially (data directories)
- Committed: Config files yes, data no

**`operations/mimir-mixin-compiled/`:**
- Purpose: Pre-compiled dashboards/alerts
- Generated: Yes (from mixin)
- Committed: Yes

**`integration/e2e*`:**
- Purpose: Temporary test directories
- Generated: Yes (during test runs)
- Committed: No (gitignored)

---

*Structure analysis: 2026-02-23*
