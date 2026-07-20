# Contribution #2: Switch in_queue and processing internal spans to logs

**Contribution Number:** 2
**Student:** Minh Le
**Issue:** https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/1193
**Status:** Phase 3 [Complete]

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

*(Covers Tasks 1–4 of the implementation plan above. Task 5 — wiring `LogsConfig`/`LogsReceiver` into `pkg/obi/config.go` and `pkg/appolly/instrumenter.go`'s swarm graph — is in progress in a separate session as of this writing, so nothing below depends on that wiring existing yet.)*

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

- [ ] Toggle-on variant confirming 0 "in queue"/"processing" spans + 1 log record through the real eBPF pipeline (real docker-compose + Jaeger, per the Reproduction Process environment above) — deferred to Task 6. Nothing to exercise end-to-end yet: `LogsReceiver` isn't subscribed to the live span queue until Task 5's `instrumenter.go` wiring lands.
- [ ] Toggle-off regression check (still exactly 3 spans, unchanged) — also deferred to Task 6, since none of Tasks 1–4 change today's default-off behavior in the live pipeline.

### Manual Testing

Not yet performed for this contribution. The pieces built in Tasks 1–4 (suppression toggle, `LogsConfig`, `logsgen.GenerateLogs`, `LogsReceiver`/`getLogsExporter`) are each unit-tested in isolation but aren't wired into the live swarm pipeline yet (Task 5) or exercised through the docker-compose/real-Jaeger repro environment (Task 6). Manual verification will follow once that wiring lands.

---

## Implementation Notes

## Week 7

### Task 1 — Span-suppression toggle (committed: `12773ed6 feat(traces): add span-suppression toggle`)

Threaded a `suppressQueueProcessingSpans bool` parameter through the existing traces call chain end-to-end: `tracesgen.generateTracesWithAttributes` → `tracesgen.GenerateTracesWithSelectedResourceAttributes` → `otel.tracesOTELReceiver` (`makeTracesReceiver`/`processSpans`) → `otel.TracesReceiver` → `pkg/appolly/instrumenter.go`'s `newGraphBuilder`. Hardcoded to `false` at the `instrumenter.go` call site for this task, exactly as planned — zero behavior change until Task 5 flips it to a real derived value.

Renamed the unexported `spanStartTime` to the exported `SpanStartTime` in `tracesgen.go` so `logsgen` (a separate package, added in Task 3) could reuse the exact same "was there an observable queue gap" calculation instead of reimplementing it.

One deliberate behavior decision: when sub-spans are suppressed, the parent span now takes the real eBPF-assigned `SpanID` directly (when valid) rather than generating a fresh random one — matching how the code already behaves for spans that never had sub-spans in the first place, so there's exactly one code path for "no sub-spans," not two.

### Task 2 — `otelcfg.LogsConfig` (uncommitted)

Mirrored `TracesConfig` field-for-field for settings that generalize (endpoint/protocol/batch/queue/retry/backoff), with two intentional deviations from a blind copy:
- Backoff fields are namespaced per-signal (`OTEL_EBPF_LOGS_BACKOFF_*`) instead of reusing `TracesConfig`'s generically-named env vars — logs is a new signal with no existing precedent of those being intentionally *shared* across signals, so introducing a new shared name would have been an assumption, not a fact.
- `SDKLogLevel` *is* intentionally the same env var as `TracesConfig`/`MetricsConfig` use, since that one is already an established shared knob.

`NormalizeQueueConfig()` reuses the same "reject queue smaller than 2x batch size" guard `TracesConfig` has, for the same reason: the collector's memory queue silently drops every batch with `"element size too large"` if this is misconfigured, and that failure mode isn't obvious from the config alone.

### Task 3 — `logsgen.GenerateLogs` (uncommitted)

Deliberately thin: reuses `tracesgen.GroupSpans` (filtering/sampling), `tracesgen.TraceAppResourceAttrs`/`AttrsToMap` (resource attribute construction), and the newly-exported `tracesgen.SpanStartTime` from Task 1 (gap detection) — none of that logic is reimplemented. The only new logic is the record shape: one `plog.LogRecord` per request with an observable gap, `EventName` fixed to `"request.queue_processing"`, `trace_id`/`span_id` set directly from the request span (the whole correlation mechanism — no new ID-coordination machinery needed, per the design-phase investigation into the eBPF layer noted under Solution Approach above), and `queue.duration`/`processing.duration` as `double` attributes in seconds.

Requests without an observable gap produce zero log records for that request, not a record with zero-valued durations — verified explicitly by `TestGenerateLogs_NoObservableGap`, since inventing a record with meaningless data would be worse than emitting nothing.

### Task 4 — `otel.LogsReceiver` / `getLogsExporter` (uncommitted)

Mirrors `traces.go`'s `tracesOTELReceiver`/`getTracesExporter` closely enough that it reuses several of that file's package-private helpers directly instead of duplicating them: `emptyHost`, `udsHost`, `udsMiddleware`, `getTraceSettings`, `convertHeaders` — all defined once in `traces.go` and shared since `logs.go` lives in the same `otel` package.

One structural difference from `TracesReceiver`: `LogsReceiver` takes both `otelcfg.LogsConfig` *and* `otelcfg.TracesConfig`, not just its own config. This is intentional, not leftover scaffolding — the log record describes timing that's *part of* a trace, so it must be subject to the exact same accept/sample decision as that trace. Passing `tracesCfg` through gives `LogsReceiver` the same `InstrumentationSelection` and `Sampler` implementation the traces pipeline uses, so a request that traces would have dropped can't produce an orphaned log record with no corresponding span for a backend to correlate it against.

`processSpans` skips `exp.ConsumeLogs` entirely when a span group produces zero log records (no request in the batch had an observable gap), rather than sending an empty `plog.Logs` batch — avoids exercising the exporter's batching/retry machinery for a no-op.


### Code Changes

- **Files modified:**
- `api/v2.0/swagger.yaml` — new field in both `Configurations` and `ConfigurationsResponse` definitions
- `src/common/const.go` — `OIDCLoginGroups` constant
- `src/core/controllers/oidc.go` — `IsLoginAllowed` enforcement in browser callback
- `src/lib/config/metadata/metadatalist.go`— config key registration
- `src/lib/config/models/model.go` — `LoginGroups` field on `OIDCSetting`
- `src/lib/config/userconfig.go` — builder wire-up
- `src/pkg/oidc/helper.go` — `IsLoginAllowed` function
- `src/pkg/oidc/helper_test.go`— 8 unit test cases + conditional DB init in `TestMain`
- `src/portal/src/app/base/left-side-nav/config/auth/config-auth.component.html` — UI input field
- `src/portal/src/app/base/left-side-nav/config/config.ts` — field declaration + constructor init
- `src/portal/src/i18n/lang/` (10 files) — `OIDC_LOGIN_GROUPS` and `OIDC_LOGIN_GROUPS_INFO` strings
- `src/server/middleware/security/oidc_cli.go` — `IsLoginAllowed` enforcement in CLI secret path

- **Key commits:**
- [46996671 — feat(oidc): added IsLoginAllowed function](https://github.com/mlhv/harbor/commit/46996671e5a1f7dfed5a0803e584e0fa34c454a2)
- [2fa5bf2e — feat(api): exposed oidc_login_groups in config API model](https://github.com/mlhv/harbor/commit/2fa5bf2eaeb97001ae52483850f443b9f264f549)
- [892718d1 — feat(oidc): enforce login group restriction in OIDC callback](https://github.com/mlhv/harbor/commit/892718d1c51aea7b5207986388ba3e6b21d2ce70)
- [c4e257da — feat(portal): add oidc_login_groups field to OIDC configuration UI](https://github.com/mlhv/harbor/commit/c4e257da10c7b4f969dfb032f3b0aaf46c52332c)
- [393d30b5 — fix(oidc): patched cli secret (docker login) path ← found in final code review](https://github.com/mlhv/harbor/commit/393d30b507f285df938c0f282ffe23516cb3204b)

- **Approach decisions:** 
- Used `config.SplitAndTrim` (an existing utility in `src/lib/config/userconfig.go`) instead of writing a manual split+trim loop — also fixes a subtle edge case where a trailing comma would produce an empty string entry.
- When `oidc_login_groups` is set but `oidc_groups_claim` is empty, the log level is `ERROR` (not Warning) because this is a misconfiguration that silently blocks all users — it needs to stand out in logs. 
- The `IsLoginAllowed` function operates on raw pre-filter groups (from the token, before `oidc_group_filter` is applied). This is intentional and documented with a comment — `oidc_login_groups` and `oidc_group_filter` are orthogonal controls. 
- Group name matching is case-sensitive by design, consistent with the existing `oidc_group_filter` and the OIDC spec. Documented in a code comment.

---

## Pull Request

**PR Link:** https://github.com/goharbor/harbor/pull/23443

**PR Description:** Adds a new `oidc_login_groups` configuration field that restricts OIDC login to users who are members of at least one specified group. When the field is empty (the default), all OIDC users can log in — existing gbehaviour is fully preserved.

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review]

---

## Learnings & Reflections

### Technical Skills Gained

Learned how to write Go tests for packages, operating Docker Compose, knowledge of OIDC claims/authentication and JWT tokens.

### Challenges Overcome
Setting up the local dev environment (especially when official macOS images are not available yet) I solved this challenge by raising a separate issue and PR surprisingly, which now puts into light more macOS support from now on in the repo.

### What I'd Do Differently Next Time

- Making sure that I reach out to the maintainers for help regarding the initial dev environment setup. 
- Checking to see for every code change, if there is a corresponding test file to write tests in.

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- https://github.com/goharbor/harbor/issues/22730 - Main issue to be resolved.
