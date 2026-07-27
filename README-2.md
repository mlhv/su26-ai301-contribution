# Contribution #2: Switch in_queue and processing internal spans to logs

**Contribution Number:** 2
**Student:** Minh Le
**Issue:** https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/1193
**Status:** Phase 4 [In Progress]

---

## Why I Chose This Issue

The issue interests me as it will involve learning about auto-instrumentation and I've been hearing a lot about eBPF technology and wanted to learn more about it. I have already had some exposure to the open-telemetry ecosystem so this will be a nice issue to work on.

Another reason that I chose this issue is the codebase is mainly written in C and I wanted to learn more about memory management/low-level debugging.

---

## Understanding the Issue

### Problem Description

OBI's traces exporter creates two full internal child spans, `"in queue"` and `"processing"`, for every request where there's an observable gap between the request arriving and its handler actually starting. These are complete sibling `ptrace.Span` records — each with its own `spanId`, `parentSpanId`, `traceId`, and timestamps — not lightweight annotations. For every request with queue time, this doubles the number of OTLP spans sent for that request (parent + 2 sub-spans instead of just the parent).

### Expected Behavior

Per the original issue, this queue/processing timing information should be represented in a less verbose way than two full sibling spans. The issue itself proposed span events as the fix ("perhaps span events are better choice for this information"). However, the OTel Span Event API (`AddEvent`/`RecordException`) has since been deprecated in favor of the Logs API (https://opentelemetry.io/blog/2026/deprecating-span-events/) — the underlying OTLP data model and the ability to view events on a span timeline aren't deprecated, only the API, but new work is steered toward logs. So the actual expected behavior, updated from the original issue, is: this data should become a log record, not a span event.

### Current Behavior

Every request with an observable queue gap (`RequestStart < Start < End`) produces exactly 3 sibling spans: `"in queue"` (`RequestStart` → `Start`), `"processing"` (`Start` → `End`), and the parent request span (e.g. `"GET /create-trace"`), all sharing one `traceId` and parented under the top span. Confirmed two ways: the existing `TestGenerateTraces` unit test, and a live docker-compose + real Jaeger repro querying `http://localhost:16686/api/traces` after sending requests through the actual eBPF pipeline.

### Affected Components

- `pkg/export/otel/tracesgen/tracesgen.go` — `createSubSpans`, `generateTracesWithAttributes` (span creation to be gated/suppressed by a new toggle)
- `pkg/export/otel/traces.go` — `TracesReceiver`, `processSpans` (production wiring for the traces signal)
- `pkg/export/otel/otelcfg/config_traces.go` — `TracesConfig` (existing pattern to mirror for the new `LogsConfig`)
- `pkg/export/otel/otelcfg/config_logs.go` — new `LogsConfig` (to be created)
- `pkg/export/otel/logsgen/logsgen.go` — new package building `plog.Logs` records (to be created)
- `pkg/export/otel/logs.go` — new `LogsReceiver`/`getLogsExporter` (to be created)
- `pkg/obi/config.go` — `Config` struct: new `Logs` field and `QueueProcessingAsLogsEnabled()` toggle helper
- `pkg/appolly/instrumenter.go` — swarm pipeline graph wiring, new `OTELLogsReceiver` node
- `internal/test/integration/` — integration test suite (real-Jaeger repro today; toggle-on variant to be added)
- `schemas/obi/groups/traces.yaml` — weaver semantic-convention registry (new `queue.duration`/`processing.duration` attributes)

---

## Reproduction Process

### Environment Setup

Development environment: an OrbStack Ubuntu 26.04 (arm64) VM on Apple Silicon, matching the host architecture with no emulation. The repo fork is cloned onto the VM's native Linux disk rather than the virtiofs-shared `/Users` mount, since eBPF/BTF work needs real Linux filesystem semantics. Toolchain: Go 1.25.11 (installed directly from go.dev — apt's `golang-go` is too old for this repo's `go.mod`), `clang`, `clang-format`, `clang-tidy`, `make`, `docker.io`. Confirmed BTF support via `/sys/kernel/btf/vmlinux` (present). `make dev` (generate + compile-for-coverage) runs successfully. Also had to install `docker-compose-v2` via apt to get the `docker compose` subcommand working, since it wasn't present by default in this image.

### Steps to Reproduce

1. Run the existing unit test to confirm today's span-based behavior at the assertion level:
   `go test ./pkg/export/otel/... -run TestGenerateTraces -v`
2. Since the test only asserts pass/fail, add a temporary throwaway test that dumps the actual OTLP JSON span payload (via `ptrace.JSONMarshaler`) for a span with an observable queue gap, to see the real shape.
3. Bring up the full integration docker-compose stack for a live, real-eBPF repro:
   `OTEL_EBPF_EXECUTABLE_PATH="(pingclient|testserver)" docker compose -f internal/test/integration/docker-compose.yml up -d --build`
4. Send requests through an endpoint that introduces an observable queue gap: `curl "http://localhost:8080/create-trace?delay=10ms&status=200"`
5. Query Jaeger's API directly: `curl "http://localhost:16686/api/traces?service=testserver&operation=GET%20%2Fcreate-trace"`
6. Tear down: `docker compose -f internal/test/integration/docker-compose.yml down -v`

### Reproduction Evidence

- **Unit-level:** The throwaway JSON dump showed exactly the shape described above — `"in queue"` (kind 1/INTERNAL, `startTimeUnixNano`/`endTimeUnixNano` spanning `RequestStart`→`Start`), `"processing"` (kind 1/INTERNAL, `Start`→`End`), and the parent (kind 2/SERVER, `RequestStart`→`End`), all sharing one `traceId`, both children's `parentSpanId` pointing at the parent's `spanId`.
- **Live/integration-level:** Querying Jaeger after sending real requests through the actual eBPF pipeline returned traces with exactly 3 spans each — e.g. trace `cdb4faf1...` with spans `in queue` → `07d2dccc...` (parent), `processing` → `07d2dccc...` (parent), `GET /create-trace` (parent, no parent ref) — repeated consistently across multiple requests.
- **Feature branch:** https://github.com/mlhv/opentelemetry-ebpf-instrumentation/tree/queue-processing-spans-to-logs (implementation not yet started as of this writing; branch created off an up-to-date `upstream/main`, ready for task-by-task work).

---

## Solution Approach

### Analysis

OBI already has a full traces export pipeline (`pkg/export/otel/traces.go`, `tracesgen/`) but no logs export pipeline exists anywhere in `pkg/export/otel/` today — only `metric` and `traces`/`tracesgen`. So this is "add a new signal," not a small API swap. The good news: the same collector exporter factories OBI already uses for traces (`otlpexporter`, `otlphttpexporter`, `debugexporter`) already support `CreateLogs` out of the box — the same OTLP exporter binary handles all signals — so a minimal logs pipeline is largely mechanical duplication of the existing traces pattern, not new architecture.

For the opt-in toggle, this repo has two existing conventions: a bitmask `Features` pattern (`pkg/export/feature.go`) used for selecting *which metric families* get computed, and a simple per-feature `Enabled bool` pattern (`NodeJSConfig.Enabled`, `JavaConfig.Enabled`, `JVMRuntimeMetricsConfig.Enabled`) used for opt-in instrumentation behaviors. The latter fits this feature much better.

A real correctness question surfaced during design: since the new logs pipeline and the existing traces pipeline would independently read the same spans off the same queue, they need to agree on the same parent span ID for the log record to actually correlate with the trace a backend receives. Investigation into the eBPF layer (`bpf/generictracer/protocol_http.h` and equivalents) showed that a span ID is generated unconditionally for essentially every event regardless of incoming trace-context propagation, and a Go-side safety net (`pkg/ebpf/common/pids.go`'s `normalizeTraceContext`) backfills any exception — both of which run once, upstream of where the pipeline fans out to multiple independent subscribers. So the two receivers naturally agree without needing new coordination machinery.

### Proposed Solution

Add a plain config bool, `LogsConfig.QueueProcessingLogs`, gated by a derived helper `Config.QueueProcessingAsLogsEnabled()` that only returns true when a logs endpoint is *also* configured — so a half-configured toggle can never silently drop the queue/processing timing data. When enabled, `createSubSpans` stops firing entirely (replace, not additive — matching how the other opt-in toggles in this codebase work), and a new `LogsReceiver`/`logsgen` pipeline emits one combined log record per request (only when there was an observable queue gap), with both durations (`queue.duration`, `processing.duration`, in seconds) as attributes, correlated to the parent span via the log record's native `trace_id`/`span_id` fields.

### Implementation Plan

Using the UMPIRE framework:

**Understand:** OBI needs a brand-new logs export signal so the queue/processing timing information can be represented as a single log record instead of two extra full OTLP spans — following OTel's own guidance to target the Logs API instead of the now-deprecated span-events API for this kind of migration.

**Match:** The existing traces signal (`otelcfg.TracesConfig`, `tracesgen/tracesgen.go`, `traces.go`) is the pattern to mirror almost 1:1 — same collector exporter factories, same batch/queue/retry configuration knobs, same "new swarm node subscribing to the existing span queue" wiring pattern already used for the metrics sub-pipeline (`setupMetricsSubPipeline`, which subscribes to the same `exportableSpans` queue as `OTELTracesReceiver` today).

**Plan:**
1. Thread a `suppressQueueProcessingSpans` bool through `tracesgen.go`/`traces.go`/`instrumenter.go` with zero behavior change yet (hardcoded `false`), with a new test proving the toggle collapses 3 spans down to 1 when set.
2. Add `otelcfg.LogsConfig`, mirroring `TracesConfig`'s endpoint/protocol/batch/queue/retry/backoff fields, plus the new `QueueProcessingLogs` toggle field.
3. Add the `logsgen` package's `GenerateLogs` function, building `plog.Logs` records by reusing the existing, already-exported `tracesgen.GroupSpans` (for filtering/sampling) and a newly-exported `tracesgen.SpanStartTime` (for detecting the observable gap) — no reimplementation of that logic.
4. Add `otel.LogsReceiver`/`getLogsExporter` in a new `logs.go`, reusing several already-existing package-private helpers from `traces.go` directly (`convertHeaders`, `emptyHost`, `udsHost`, `udsMiddleware`, `getTraceSettings`) since both files live in the same `otel` package.
5. Wire everything into `pkg/obi/config.go` (new `Config.Logs` field, `QueueProcessingAsLogsEnabled()` helper, queue-config validation) and `pkg/appolly/instrumenter.go` (new `OTELLogsReceiver` swarm node; flip the Task 1 suppression flag from hardcoded `false` to the real derived value).
6. Extend the integration test suite with a toggle-on variant (verified via OBI's own debug-protocol log output, since Jaeger doesn't expose a generic log-query API) and register the two new attributes in the `schemas/obi` weaver semantic-convention registry.

**Implement:** Not yet started. Branch `queue-processing-spans-to-logs` created off an up-to-date `upstream/main` and pushed to my fork: https://github.com/mlhv/opentelemetry-ebpf-instrumentation/tree/queue-processing-spans-to-logs

**Review:**
- [ ] Toggle defaults to off — zero behavior change for existing users who don't opt in
- [ ] Toggle only takes effect when a logs endpoint is actually configured — no silent data loss from a half-configured toggle
- [ ] Follows existing OBI patterns (mirrors `TracesConfig`/`traces.go`'s structure; reuses `tracesgen.GroupSpans`/`TraceAppResourceAttrs`/`AttrsToMap` instead of reimplementing them, per this repo's CONTRIBUTING.md guidance against duplicating existing functionality)
- [ ] Unit tests for `logsgen.GenerateLogs`, the new span-suppression gating in `tracesgen`, and `LogsConfig`
- [ ] Integration test confirming both toggle-off (unchanged, still 3 spans) and toggle-on (1 log record, 0 sub-spans) behavior
- [ ] New attributes (`queue.duration`, `processing.duration`) registered in the `schemas/obi` weaver registry
- [ ] No unrelated or opportunistic refactoring bundled into the change
- [ ] `make fmt` / `make lint` clean before opening the PR

**Evaluate:** (Planned; not yet executed, since implementation hasn't started)
1. Run the full unit test suite: `go test ./pkg/export/otel/... ./pkg/obi/... ./pkg/appolly/... -v`
2. Re-run the toggle-off integration path to confirm zero regression — still exactly 3 spans (`"in queue"`, `"processing"`, parent) per request with observable queue time.
3. Run the new toggle-on integration test and confirm exactly 1 log record (`event.name: "request.queue_processing"`, with `queue.duration`/`processing.duration` attributes) and 0 `"in queue"`/`"processing"` spans per request.
4. Manually inspect the debug-protocol log output captured from OBI's own container logs to sanity-check the actual OTLP log payload shape.

---

## Testing Strategy

### Unit Tests

- [x] Test Case 1: Span-suppression toggle collapses the sub-spans — `TestGenerateTraces/test_with_subtraces_-_queue_processing_suppressed` (`pkg/export/otel/traces_test.go`). With `suppressQueueProcessingSpans=true`, a request with an observable queue gap now emits exactly 1 span (the parent) instead of 3, and the parent takes the real eBPF-assigned `SpanID` directly rather than a freshly generated random one.
- [x] Test Case 2: `LogsConfig.Enabled()` covers all four ways the logs signal turns on — `TestLogsConfig_Enabled` (`pkg/export/otel/otelcfg/config_logs_test.go`): disabled by default, enabled via a plain endpoint, enabled via `ProtocolDebug`, enabled via an injected `LogsConsumer`. Mirrors the same four cases already covered for `TracesConfig`/`MetricsConfig`.
- [x] Test Case 3: Logs endpoint resolution — `TestHTTPLogsEndpoint` / `TestHTTPLogsEndpoint_CommonOnly`: a signal-specific `LogsEndpoint` takes precedence over `CommonEndpoint`, and a common endpoint alone still resolves correctly with `/v1/logs` appended.
- [x] Test Case 4: Queue-config validation — `TestLogsConfig_NormalizeQueueConfig`: defaults `QueueSize` to `4 * BatchMaxSize` when unset, and rejects a `QueueSize` smaller than `2 * BatchMaxSize` (the same guard `TracesConfig` has — an under-sized queue silently drops every batch with `"element size too large"`, a failure mode that isn't obvious from the config alone).
- [x] Test Case 5: `logsgen.GenerateLogs` emits exactly one combined log record when there's an observable queue gap — `TestGenerateLogs_ObservableGap`: asserts `EventName == "request.queue_processing"`, the log record's `trace_id`/`span_id` match the parent span's (the entire correlation mechanism this approach depends on), and `queue.duration`/`processing.duration` are correct to within 10ms.
- [x] Test Case 6: `logsgen.GenerateLogs` emits zero log records when there's no observable gap — `TestGenerateLogs_NoObservableGap`: confirms "replace, not additive" doesn't invent timing data for requests that never queued.
- [x] Test Case 7: `getLogsExporter` builds a working debug-protocol exporter that starts and shuts down cleanly — `TestGetLogsExporter_Debug`.
- [x] Test Case 8: `getLogsExporter` rejects an invalid protocol string — `TestGetLogsExporter_InvalidProtocol`.

All 8 cases (15 sub-tests) pass:
```
go test ./pkg/export/otel/... -run 'TestGenerateTraces|TestGetLogsExporter' -v
go test ./pkg/export/otel/logsgen/... -v
go test ./pkg/export/otel/otelcfg/... -run 'TestLogsConfig|TestHTTPLogsEndpoint' -v
```
`go build ./...` is also clean with all of Tasks 1–4's new/modified files in the tree — no compile-time drift between the new `logs`/`logsgen`/`config_logs` packages and the existing `otel`/`otelcfg` code they extend.

### Integration Tests

- [x] Toggle-on variant confirming 0 "in queue"/"processing" spans + 1 log record through the real eBPF pipeline — `TestSuite_QueueProcessingLogs` (`internal/test/integration/logs_test.go`), a new top-level test with its own `docker.ComposeSuite` and its own config (`configs/obi-config-logs.yml`, `queue_processing_logs: true`, `protocol: debug`), so it doesn't perturb the existing `TestSuite_Go` suite/config at all. Polls the debug-protocol exporter's rendered output (`require.EventuallyWithT`, since the span→log pipeline is asynchronous) for `EventName: request.queue_processing`, `queue.duration: Double(...)`, `processing.duration: Double(...)`.
- [x] Toggle-off regression check (still exactly 3 spans, unchanged) — re-ran the pre-existing `TestSuite_Go/go-old-supported/HTTP_traces` subtest against the same branch; confirmed it still passes unmodified, proving the toggle-off default path is untouched by any of this work.

Run: `go clean -testcache && go test -p 1 -v -timeout 30m -run TestSuite_QueueProcessingLogs ./internal/test/integration`, then `-run TestSuite_Go/go-old-supported/HTTP_traces` for the regression check. Both green.

### Manual Testing

Jaeger (used for the toggle-off repro earlier in this doc) can't be used to manually inspect the *toggle-on* behavior end-to-end: it's a traces-only backend with no logs ingestion path at all, so a log record sent to it wouldn't be stored or queryable, let alone visibly correlated to its trace. Span events used to render inline on a span's own timeline because that data was physically part of the span record itself, stored and rendered by Jaeger directly — a separate log record that merely *references* a span via `trace_id`/`span_id` needs a backend that ingests and correlates both signals to show that relationship at all.

Set up a disposable `docker-compose` stack (outside the repo, in a scratch directory — not part of this branch) reusing the already-built `hatest-testserver`/`hatest-obi` images, swapping Jaeger+otelcol for a single `grafana/otel-lgtm` container (Tempo for traces, Loki for logs, Grafana UI, all pre-wired with native OTLP ingestion on one endpoint). The demo OBI config deliberately does **not** set `otel_logs_export.endpoint` or `otel_traces_export.endpoint` explicitly — it relies on the shared `OTEL_EXPORTER_OTLP_ENDPOINT` env var populating both `TracesConfig.CommonEndpoint` and `LogsConfig.CommonEndpoint` (both fields carry the identical env tag), confirmed via OBI's own debug logs showing both `/v1/traces` and `/v1/logs` HTTP requests firing to the same endpoint from one config line. This is the exact same shared-endpoint mechanism responsible for the Critical finding under Approach decisions below, deliberately exercised here as a positive case.

Sent real requests through the live eBPF pipeline (`curl "http://localhost:8080/create-trace?delay=300ms&status=200"`) and verified the result two ways, not just by eyeballing the UI:

- **Tempo's trace query API** (`/api/traces/<traceID>`) returned exactly **1 span** per request (`GET /create-trace`, no `"in queue"`/`"processing"` children) — suppression confirmed directly from the backend's own data, not inferred from the UI.
- **Loki's range-query API** (`/loki/api/v1/query_range`) returned the `request.queue_processing` log line for the same request, with `queue_duration`/`processing_duration` auto-promoted to queryable labels and, critically, a **`trace_id` that matched the Tempo trace exactly** — cross-referencing the two APIs by ID is what actually proves the correlation is real, rather than two unrelated pieces of telemetry that happen to both exist. `processing_duration` (`0.303157506`) matched the requested `delay=300ms` almost exactly; `queue_duration` (tens to low-hundreds of microseconds) reflected real, unforced Go-runtime scheduling jitter rather than anything deliberately induced.
- Also confirmed the log record's `Body` is genuinely empty (`""` in Loki's raw response) — all of the data lives in structured `Attributes` and first-class fields (`EventName`, `TraceID`, `SpanID`), consistent with the "events as logs" model this whole feature targets rather than the traditional free-text log-message convention.

Used Grafana's Explore UI (split view: Tempo pane + Loki pane, linked by the same trace ID) to capture the actual PR screenshots, only after the API-level cross-check above already confirmed the underlying data was correct.

---

## Implementation Notes

### Task 1 — Span-suppression toggle (committed: `12773ed6 feat(traces): add span-suppression toggle`)

Threaded a `suppressQueueProcessingSpans bool` parameter through the existing traces call chain end-to-end: `tracesgen.generateTracesWithAttributes` → `tracesgen.GenerateTracesWithSelectedResourceAttributes` → `otel.tracesOTELReceiver` (`makeTracesReceiver`/`processSpans`) → `otel.TracesReceiver` → `pkg/appolly/instrumenter.go`'s `newGraphBuilder`. Hardcoded to `false` at the `instrumenter.go` call site for this task, exactly as planned — zero behavior change until Task 5 flips it to a real derived value.

Renamed the unexported `spanStartTime` to the exported `SpanStartTime` in `tracesgen.go` so `logsgen` (a separate package, added in Task 3) could reuse the exact same "was there an observable queue gap" calculation instead of reimplementing it.

One deliberate behavior decision: when sub-spans are suppressed, the parent span now takes the real eBPF-assigned `SpanID` directly (when valid) rather than generating a fresh random one — matching how the code already behaves for spans that never had sub-spans in the first place, so there's exactly one code path for "no sub-spans," not two.

### Task 2 — `otelcfg.LogsConfig` (committed: `441937be feat(otelcfg): add LogsConfig for new logs export signal`)

Mirrored `TracesConfig` field-for-field for settings that generalize (endpoint/protocol/batch/queue/retry/backoff), with two intentional deviations from a blind copy:
- Backoff fields are namespaced per-signal (`OTEL_EBPF_LOGS_BACKOFF_*`) instead of reusing `TracesConfig`'s generically-named env vars — logs is a new signal with no existing precedent of those being intentionally *shared* across signals, so introducing a new shared name would have been an assumption, not a fact.
- `SDKLogLevel` *is* intentionally the same env var as `TracesConfig`/`MetricsConfig` use, since that one is already an established shared knob.

`NormalizeQueueConfig()` reuses the same "reject queue smaller than 2x batch size" guard `TracesConfig` has, for the same reason: the collector's memory queue silently drops every batch with `"element size too large"` if this is misconfigured, and that failure mode isn't obvious from the config alone.

### Task 3 — `logsgen.GenerateLogs` (committed: `e5586dab feat(logsgen): build queue/processing log records from spans`)

Deliberately thin: reuses `tracesgen.GroupSpans` (filtering/sampling), `tracesgen.TraceAppResourceAttrs`/`AttrsToMap` (resource attribute construction), and the newly-exported `tracesgen.SpanStartTime` from Task 1 (gap detection) — none of that logic is reimplemented. The only new logic is the record shape: one `plog.LogRecord` per request with an observable gap, `EventName` fixed to `"request.queue_processing"`, `trace_id`/`span_id` set directly from the request span (the whole correlation mechanism — no new ID-coordination machinery needed, per the design-phase investigation into the eBPF layer noted under Solution Approach above), and `queue.duration`/`processing.duration` as `double` attributes in seconds.

Requests without an observable gap produce zero log records for that request, not a record with zero-valued durations — verified explicitly by `TestGenerateLogs_NoObservableGap`, since inventing a record with meaningless data would be worse than emitting nothing.

### Task 4 — `otel.LogsReceiver` / `getLogsExporter` (committed: `794f4d49 feat(otel): add LogsReceiver and getLogsExporter`)

Mirrors `traces.go`'s `tracesOTELReceiver`/`getTracesExporter` closely enough that it reuses several of that file's package-private helpers directly instead of duplicating them: `emptyHost`, `udsHost`, `udsMiddleware`, `getTraceSettings`, `convertHeaders` — all defined once in `traces.go` and shared since `logs.go` lives in the same `otel` package.

One structural difference from `TracesReceiver`: `LogsReceiver` takes both `otelcfg.LogsConfig` *and* `otelcfg.TracesConfig`, not just its own config. This is intentional, not leftover scaffolding — the log record describes timing that's *part of* a trace, so it must be subject to the exact same accept/sample decision as that trace. Passing `tracesCfg` through gives `LogsReceiver` the same `InstrumentationSelection` and `Sampler` implementation the traces pipeline uses, so a request that traces would have dropped can't produce an orphaned log record with no corresponding span for a backend to correlate it against.

`processSpans` skips `exp.ConsumeLogs` entirely when a span group produces zero log records (no request in the batch had an observable gap), rather than sending an empty `plog.Logs` batch — avoids exercising the exporter's batching/retry machinery for a no-op.

### Task 5 — Wiring (committed: `664880b7 feat(obi): wire the logs export pipeline into instrumentation graph`)

Added `Config.Logs otelcfg.LogsConfig` and `Config.QueueProcessingAsLogsEnabled()` to `pkg/obi/config.go`, gated on `c.Logs.Enabled() && c.Logs.QueueProcessingLogs` — deliberately both, not just the toggle, so a user who sets `queue_processing_logs: true` without a working logs endpoint never silently loses the queue/processing timing data; the old spans keep flowing until there's actually somewhere for the replacement log record to go. Flipped Task 1's hardcoded `false` to this real value in `instrumenter.go`, and added the new `OTELLogsReceiver` swarm node alongside `OTELTracesReceiver`, both subscribing independently to the same `exportableSpans` queue.

### Task 6 — Integration test + weaver registry (committed: `12ee7e0c test(integration): verify queue/processing logs toggle end-to-end`)

Added `TestSuite_QueueProcessingLogs` as a wholly separate top-level test (own `docker.ComposeSuite`, own config file) rather than a new subtest inside the existing `TestSuite_Go`, specifically so the existing toggle-off suite and its `require.Len(ct, trace.Spans, 3)` assertion stay completely untouched. Registered `queue.duration`/`processing.duration` in `schemas/obi/groups/traces.yaml` as `stability: development` attributes, matching the existing registry's structure.

---

## Code Changes

- **Files modified:**
  - `pkg/appolly/instrumenter.go` — `suppressQueueProcessingSpans`/`QueueProcessingAsLogsEnabled()` threaded into `TracesReceiver`'s call site; new `OTELLogsReceiver` swarm node
  - `pkg/export/otel/traces.go` — `suppressQueueProcessingSpans` threaded through `TracesReceiver`/`makeTracesReceiver`/`tracesOTELReceiver`/`processSpans`
  - `pkg/export/otel/traces_test.go` — new subtest for the span-suppression toggle; test-helper signature updates
  - `pkg/export/otel/tracesgen/tracesgen.go` — new `suppressQueueProcessingSpans` param on `GenerateTracesWithSelectedResourceAttributes`/`generateTracesWithAttributes`; `spanStartTime` exported as `SpanStartTime`
  - `pkg/export/otel/otelcfg/common.go` — `envLogsProtocol`/`envLogsHeaders` constants
  - `pkg/export/otel/otelcfg/config_logs.go` *(new)* — `LogsConfig` struct + endpoint/protocol resolution, mirroring `TracesConfig`
  - `pkg/export/otel/otelcfg/config_logs_test.go` *(new)*
  - `devdocs/config/config-schema.json`, `devdocs/config/CONFIG.md` — autogenerated by the repo's pre-commit hook from the `LogsConfig` struct tags
  - `pkg/export/otel/logsgen/logsgen.go` *(new package)* — `GenerateLogs`, building `plog.Logs` records from spans
  - `pkg/export/otel/logsgen/logsgen_test.go` *(new)*
  - `pkg/export/otel/logs.go` *(new)* — `LogsReceiver`/`getLogsExporter`, mirroring `traces.go`
  - `pkg/export/otel/logs_test.go` *(new)*
  - `pkg/obi/config.go` — `Config.Logs` field, `QueueProcessingAsLogsEnabled()`, `NormalizeQueueConfig()` validation call
  - `pkg/obi/config_test.go` — new test for `QueueProcessingAsLogsEnabled`; `TestConfig_Overrides`'s expected struct updated for the new shared-env-tag `Logs.CommonEndpoint` field
  - `internal/test/integration/configs/obi-config-logs.yml` *(new)* — toggle-on integration test config
  - `internal/test/integration/logs_test.go` *(new)* — `TestSuite_QueueProcessingLogs`
  - `schemas/obi/groups/traces.yaml` — `queue.duration`/`processing.duration` weaver registry entries

- **Key commits:**
  - [12773ed6 — feat(traces): add span-suppression toggle](https://github.com/mlhv/opentelemetry-ebpf-instrumentation/commit/12773ed6bcbcc4f24afe173f95460b23a2a02bed)
  - [441937be — feat(otelcfg): add LogsConfig for new logs export signal](https://github.com/mlhv/opentelemetry-ebpf-instrumentation/commit/441937bee7ca5fa8aa598ee5347313443a7b209a)
  - [18bebdf6 — feat(otelcfg): committed autogenerated config.md](https://github.com/mlhv/opentelemetry-ebpf-instrumentation/commit/18bebdf6b1a92de0927f2969a631a31303f09a5c)
  - [e5586dab — feat(logsgen): build queue/processing log records from spans](https://github.com/mlhv/opentelemetry-ebpf-instrumentation/commit/e5586dab743bbed42a364349b4ea8c81d45bf68e)
  - [794f4d49 — feat(otel): add LogsReceiver and getLogsExporter](https://github.com/mlhv/opentelemetry-ebpf-instrumentation/commit/794f4d49ebe8c0a824eb0295c4f5911fa43e2192)
  - [664880b7 — feat(obi): wire the logs export pipeline into instrumentation graph](https://github.com/mlhv/opentelemetry-ebpf-instrumentation/commit/664880b7cdf44328672a4252c10c44c7410d3f4a)
  - [12ee7e0c — test(integration): verify queue/processing logs toggle end-to-end](https://github.com/mlhv/opentelemetry-ebpf-instrumentation/commit/12ee7e0c420f89ac97749a54f419eccad3f5419c)

  All 7 commits are pushed and on `origin/queue-processing-spans-to-logs` — every link above resolves.

- **Approach decisions:**
  - `LogsReceiver`'s enable gate originally checked only `cfg.Enabled()` (verbatim from the implementation plan). Caught in the final whole-branch review: a user with the common `OTEL_EXPORTER_OTLP_ENDPOINT` already set (very common — it's how most deployments already fan traces/metrics to one collector) but the `queue_processing_logs` toggle left off would get `Enabled() == true` and start receiving brand-new log records they never opted into, while span suppression correctly stayed off — the two halves of the gate disagreed. Fixed to require both: `!cfg.Enabled() || !cfg.QueueProcessingLogs`, matching `Config.QueueProcessingAsLogsEnabled()`'s logic exactly, plus a regression test (`TestLogsReceiver_QueueProcessingLogsDisabled`) pinning the previously-broken quadrant.
  - `processSpans`'s per-service export gate originally used `ExportModes.CanExportTraces()` (also verbatim from the plan) to decide whether to emit a log record — caught during Task 4's own task-scoped review, since a service opting out of the *logs* signal specifically has no effect if the gate checks the *traces* permission. Switched to `ExportModes.CanExportLogs()`, which had zero prior call sites anywhere in the repo — `LogsReceiver` is the first consumer of that particular per-signal opt-out.
  - `logsgen.GenerateLogs` originally set the log record's `span_id` from `span.SpanID` unconditionally (again, verbatim from the plan). Caught in the final review: `tracesgen`'s suppressed-span path falls back to a freshly generated random `SpanID` when the eBPF-assigned one is invalid, a value `logsgen` has no way to reproduce — so an invalid `SpanID` would have produced a log record whose `span_id` could never match its actual parent span. Fixed by skipping the log record entirely in that case, rather than emitting one with a permanently orphaned ID.
  - Reused `tracesgen.GroupSpans`/`TraceAppResourceAttrs`/`AttrsToMap`/`SpanStartTime` in `logsgen` instead of reimplementing any of that logic — same reasoning as Harbor's `config.SplitAndTrim` reuse: it's already correct, already tested, and a second copy would just be a second place for the two signals' behavior to silently drift apart.
  - For manual verification, deliberately did not try to reuse the CI integration suite's Jaeger backend, since Jaeger has no logs-ingestion path at all — used a disposable Grafana Tempo+Loki stack instead, and verified via both backends' query APIs directly (cross-referencing `trace_id` between them) before trusting anything the UI showed.

---

## Pull Request

**PR Link:** *(not yet opened)*

**PR Description:** *(to be written once the branch is pushed and ready for review)*

**Maintainer Feedback:**
- *(none yet)*

**Status:** Not yet opened
