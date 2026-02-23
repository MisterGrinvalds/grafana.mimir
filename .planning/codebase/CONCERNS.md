# Codebase Concerns

**Analysis Date:** 2026-02-23

## Tech Debt

**Deprecated gRPC Dial API:**
- Issue: Multiple places use deprecated `grpc.Dial()` which will be removed in gRPC 2
- Files:
  - `pkg/blockbuilder/blockbuilder.go:129`
  - `pkg/usagetracker/usagetrackerclient/grpc_client_pool.go:73`
  - `pkg/storegateway/bucket_store_server_test.go:259`
  - `pkg/ruler/grpc_roundtripper.go:36`
  - `pkg/ruler/client_pool.go:85`
  - `pkg/ingester/client/buffering_client_test.go:43`
  - `pkg/ingester/client/client.go:54`
  - `pkg/ingester/client/mimir_util_test.go:34`
- Impact: Will break when upgrading to gRPC 2.x
- Fix approach: Replace `grpc.Dial()` with `grpc.NewClient()` (tracked via nolint comments noting "we'll address it before upgrading to gRPC 2")

**Deprecated Ruler Configuration:**
- Issue: Ruler has deprecated alertmanager configuration fields still in use
- Files:
  - `pkg/ruler/ruler.go:119` - `DeprecatedAlertmanagerURL`
  - `pkg/ruler/ruler.go:129` - `DeprecatedNotifier`
  - `pkg/ruler/rulespb/rules.pb.go:48` - deprecated `queryOffset` field
- Impact: Configuration confusion, dual-path maintenance
- Fix approach: Complete migration to new configuration structure (limits.ruler_alertmanager_client_config)

**Deprecated Block Labels:**
- Issue: Legacy block label constants remain in codebase
- Files:
  - `pkg/compactor/compactor.go:67-68` - `DeprecatedIngesterIDExternalLabel`, `DeprecatedTenantIDExternalLabel`
  - `pkg/storegateway/bucket_stores.go:46` - `DeprecatedTenantIDExternalLabel`
  - `pkg/compactor/block_upload.go:459` - checking deprecated labels
- Impact: Backward compatibility maintenance burden
- Fix approach: Eventually remove deprecated label support after sufficient migration period

**Store Gateway Ring Auto-Forget Deprecation:**
- Issue: `auto_forget_enabled` configuration is deprecated
- Files: `pkg/storegateway/gateway_ring.go:88`
- Impact: Confusing configuration options
- Fix approach: Remove deprecated field after deprecation period

**Incomplete TODOs in Active Development Areas:**
- Issue: Several TODOs indicate incomplete features or needed improvements
- Files:
  - `pkg/blockbuilder/tsdb.go:335` - Missing secondary hash function for limits
  - `pkg/blockbuilder/blockbuilder.go:265` - Can skip exemplar unmarshaling optimization
  - `pkg/blockbuilder/blockbuilder.go:501` - Missing block count metrics tracking
  - `pkg/blockbuilder/scheduler/scheduler.go:745` - Flush only if dirty
  - `pkg/blockbuilder/scheduler/scheduler.go:1014` - Missing error return for rejection reasons
  - `pkg/mimirpb/timeseries.go:276` - Check RW2 caching applicability
  - `pkg/distributor/validate.go:204-205` - Per-series/sample overrides efficiency
- Impact: Features incomplete or suboptimal
- Fix approach: Address TODOs during feature development

## Known Bugs

**No critical bugs identified from code analysis.**

The codebase uses extensive testing (test files present for all major components) and has error handling patterns in place.

## Security Considerations

**Unsafe Memory Management (CRITICAL):**
- Risk: Cross-tenant data leakage through `yoloString` optimization
- Files:
  - `pkg/mimirpb/timeseries.go:478` - `yoloString()` function
  - `pkg/mimirpb/timeseries.go:319` - `UnsafeMutableString` type alias
  - `pkg/mimirpb/timeseries.go:361-451` - Unmarshaling using yoloString
  - `pkg/mimirpb/mimir.pb.go:7557` - RW2 symbols use yoloString
  - `pkg/distributor/influxpush/parser.go:194-195` - yoloString in Influx parser
  - `pkg/storage/indexheader/index/symbols.go:292` - yoloString in index header
  - `pkg/storage/indexheader/index/postings.go:433` - yoloString in postings
- Current mitigation:
  - `strings.Clone()` calls in critical paths (see `pkg/distributor/distributor.go:1176`, `pkg/distributor/validate.go:182-184`)
  - Reference leak detection mechanism (`pkg/mimirpb/ref_leaks.go`)
  - Documentation in `CLAUDE.md` warning about the issue
- Recommendations:
  - Continue vigilant use of `strings.Clone()` when strings must outlive request buffers
  - Enable reference leak detection in testing/staging environments
  - Consider adding linter rules to catch unsafe string retention

**Reference Leak Detection Infrastructure:**
- Risk: Relies on sampling-based detection, may miss issues
- Files: `pkg/mimirpb/ref_leaks.go`
- Current mitigation: Configurable percentage sampling (0-100%)
- Recommendations: Run with higher sampling percentages during development

**Vault Authentication:**
- Risk: Uses `context.Background()` for authentication which ignores cancellation
- Files: `pkg/vault/vault.go:92`, `pkg/vault/vault.go:122`
- Current mitigation: Used only during initialization
- Recommendations: Consider propagating context from callers for timeout control

## Performance Bottlenecks

**Large Source Files (Complexity Indicators):**
- Problem: Several files have grown very large, indicating potential complexity issues
- Files:
  - `pkg/ingester/ingester.go` - 4547 lines
  - `pkg/distributor/distributor.go` - 3607 lines
  - `pkg/storegateway/bucket.go` - 2187 lines
- Cause: Accumulation of functionality over time
- Improvement path: Consider splitting into smaller, focused modules

**Validation Efficiency:**
- Problem: Per-request overhead in validation could be reduced
- Files: `pkg/distributor/validate.go:204-205` (noted in TODO)
- Cause: Each overrides call performs atomic pointer retrieval and map lookup
- Improvement path: Batch validation operations where possible

**Timeseries Pool Management:**
- Problem: Pool dirty object detection requires runtime check
- Files: `pkg/mimirpb/timeseries_pools.go:53`
- Cause: Pool returns dirty TimeSeries indicating improper ReuseTimeseries calls
- Improvement path: Add compile-time checks or linting rules

## Fragile Areas

**Unsafe Memory String Handling:**
- Files: All files using `yoloString`, `UnsafeMutableString`, or unmarshaling into pooled buffers
- Why fragile: Innocent-looking string operations can cause cross-tenant data leakage
- Safe modification:
  - Always use `strings.Clone()` for strings that escape request scope
  - Never retain references to `UnsafeMutableString` values beyond the handler function
  - Do not store these strings in error messages, logs, or metrics labels without cloning
- Test coverage: `pkg/mimirpb/ref_leaks_test.go` provides reference leak testing infrastructure

**Distributor Push Middleware Chain:**
- Files: `pkg/distributor/distributor.go:1140-1154`
- Why fragile: Order-dependent middleware chain; GEM integration point noted in TODO
- Safe modification: Understand full middleware chain before reordering
- Test coverage: `pkg/distributor/distributor_test.go` (9290 lines of tests)

**Ingester State Management:**
- Files: `pkg/ingester/ingester.go`
- Why fragile: Complex state transitions, ring membership, TSDB lifecycle
- Safe modification: Understand partition ring and instance ring interactions
- Test coverage: `pkg/ingester/ingester_test.go` (12354 lines of tests)

**Block Builder Scheduler:**
- Files: `pkg/blockbuilder/scheduler/scheduler.go`
- Why fragile: Job scheduling with offset tracking, partition coordination
- Safe modification: Understand job lifecycle and state persistence
- Test coverage: Gaps noted in TODOs

## Scaling Limits

**Memory Pressure from Buffer Pools:**
- Current capacity: Controlled by pool sizing
- Limit: Large requests can exhaust pool capacity
- Scaling path: Pool sizes are configurable; monitor via metrics

**Reference Leak Instrumentation:**
- Current capacity: Configurable via `max_inflight_instrumented_bytes`
- Limit: High instrumentation percentage increases memory overhead
- Scaling path: Reduce instrumentation percentage in production, increase sampling in test

## Dependencies at Risk

**gRPC v1 to v2 Migration:**
- Risk: Breaking API changes in gRPC 2.x
- Impact: All gRPC client code needs migration
- Migration plan: Replace `grpc.Dial()` with `grpc.NewClient()` per comments

**Go Version:**
- Risk: Module specifies `go 1.25.7` which is a future version
- Impact: Build compatibility with current Go toolchains
- Migration plan: Verify go.mod version requirement is correct

## Missing Critical Features

**Block Builder Metrics:**
- Problem: Block count tracking not implemented
- Files: `pkg/blockbuilder/blockbuilder.go:501`
- Blocks: Visibility into block builder operations

**Scheduler Error Detail:**
- Problem: Job rejection lacks detailed error reason
- Files: `pkg/blockbuilder/scheduler/scheduler.go:1014`
- Blocks: Debugging job scheduling issues

## Test Coverage Gaps

**Panic Paths Testing:**
- What's not tested: Many panic conditions in type assertions and histogram handling
- Files:
  - `pkg/mimirpb/compat.go:210,232,264,301,324` - Histogram type panics
  - `pkg/mimirpb/custom.go:71` - Unrecognized message type panic
  - `pkg/storage/indexheader/lazy_binary_reader.go:457-458` - Reader timeout panic
  - `pkg/blockbuilder/scheduler/scheduler.go:309` - Invalid spec panic
- Risk: Unexpected panics in production could crash processes
- Priority: Medium - most are defensive checks that should not be hit in normal operation

**Large Test Files Indicating Complex Testing Requirements:**
- What's not tested: Comprehensive integration between components
- Files:
  - `pkg/ingester/ingester_test.go` - 12354 lines
  - `pkg/distributor/distributor_test.go` - 9290 lines
- Risk: Complex test suites may have gaps or be difficult to maintain
- Priority: Low - extensive testing exists, but test organization could be improved

**Cost Attribution Active Series Recovery:**
- What's not tested: Complex state recovery from overflow
- Files: `pkg/costattribution/manager_test.go:289`
- Risk: Edge cases in overflow recovery might not be fully covered
- Priority: Medium

---

*Concerns audit: 2026-02-23*
