# Contribution #2: Switch in_queue and processing internal spans to logs

**Contribution Number:** 2
**Student:** Minh Le
**Issue:** https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/1193
**Status:** Phase 3 [In Progress]

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

- [x] Test Case 1: No restriction configured — `IsLoginAllowed` returns `true` when `oidc_login_groups` is empty; confirms the feature is fully opt-in and backward compatible
- [x] Test Case 2: User in allowed group — returns `true` when the user's token groups contain an exact match
- [x] Test Case 3: User not in any allowed group — returns `false` when the user's groups have no overlap with the configured list
- [x] Test Case 4: User in one of multiple allowed groups — returns `true` for a comma-separated list, matching on any one group
- [x] Test Case 5: Spaces around group names are trimmed — `" harbor "` in the config matches `"harbor"` in the token
- [x] Test Case 6: No group claim in token and restriction set — returns false and logs an `ERROR`; handles the misconfiguration case where `oidc_groups_claim` is not set
- [x] Test Case 7: No group claim and no restriction — returns true; no group claim is not a problem when no restriction is configured
- [x] Test Case 8: hitespace-only restriction — treats "   " the same as empty; no restriction applied

All 8 cases run with `go test ./pkg/oidc/... -run TestIsLoginAllowed -v` — no database required. (The TestMain was modified to only call test.`InitDatabaseFromEnv()` when `POSTGRESQL_HOST` is set, making these the first pure unit tests in the pkg/oidc package.)

### Integration Tests

- [ ] Browser login denied for a user not in `oidc_login_groups` (verified manually — see below)
- [ ] CLI secret (`docker login`) denied for a user not in `oidc_login_groups` — the `oidc_cli.go` enforcement path, found during code review and added as part of this PR

### Manual Testing

1. I rebuilt and restarted Harbor first.
2. I then set the `oidc_login_groups` to `harbor` via the API curl (frontend admin dashboard does not work for local)
3. I tested alice (member of `harbor` group) and it succeeded! (lands on `harbor/projects`)
4. I then logged out and tested bob (no group) and it returned a 401 error saying that "user is not a member of any authorized OIDC group"
5. I cleared the setting with curl again to verify backward compatibility and logged in with Bob. This time it succeeded (no restriction as login groups field is empty)

---

## Implementation Notes

### Week 3 Progress

Built the `oidc_login_groups` feature end-to-end across all layers of the Harbor stack.

[Working branch link](https://github.com/mlhv/harbor/tree/feat/oidc-login-groups)

What was built: 
- New config key wired through all four config layers: `const.go` (key name constant) → `metadatalist.go` (registration) → `model.go` (struct field) → `userconfig.go` (builder wire-up). Missing any one of these would silently return an empty value from the config system. 
- `IsLoginAllowed(info *UserInfo, setting OIDCSetting)` bool in `pkg/oidc/helper.go` — the core enforcement function.
- Enforcement in two login paths: `Callback()` in `core/controllers/oidc.go` (browser login) and `Generate()` in
`server/middleware/security/oidc_cli.go` (docker CLI / API basic auth with CLI secret). The CLI path was discovered only during
the final code review — missing it would have let blocked users authenticate via `docker login` indefinitely.
- Swagger API model updated (`api/v2.0/swagger.yaml`) and Go models regenerated via `make gen_apis`. Without the swagger update,
the new field is silently stripped when the frontend reads and writes config via the API.
- Angular config UI (`config-auth.component.html`, `config.ts`) and `i18n` strings for all 10 supported locales.

Challenges faced:
- Two separate login paths: The browser OIDC callback (`Callback()`) and the CLI secret middleware (`oidc_cli.go`) are completely independent code paths. A restriction set in `Callback()` does not affect `docker login` at all — discovered during code review.
- Swagger → generated models gap: The Go models in `src/server/v2.0/models/` are generated from `swagger.yaml`, not hand-written. Editing only the config code while forgetting the swagger spec means the API silently drops the field during JSON marshal/unmarshal.
- Unit tests for `pkg/oidc` require `postgres` (the existing TestMain calls `test.InitDatabaseFromEnv()` unconditionally). Solved by making the DB init conditional on `POSTGRESQL_HOST` being set, allowing pure unit tests to run locally without a DB connection.
- Config system has four separate layers that all need updating. Forgetting `userconfig.go` (the builder wire-up) is the most subtle miss — the DB stores the value correctly but `OIDCSetting()` always returns "".

### Week 4 Progress

Set up the local PostgreSQL dev environment so the `security` and `pkg/oidc` test suites can run end-to-end on macOS, then fixed the test issues surfaced by running them.

- Local DB setup:
Harbor's `src/server/middleware/security/` test suite calls `test.InitDatabaseFromEnv()` in `TestMain` unconditionally — without a reachable PostgreSQL it fatals before any test runs. On macOS, `harbor-db` runs on Docker's private `make_harbor` network with no host port mapping. Solved with a `socat` relay container:
`docker run -d --rm --name harbor-db-socat --network make_harbor -p 5432:5432 alpine/socat TCP-LISTEN:5432,fork TCP:harbor-db:5432`

Also discovered that `golang-migrate` resolves its migration scripts path relative to the test binary's working directory, not the repo root — overridden via `POSTGRES_MIGRATION_SCRIPTS_PATH`.

Test fixes surfaced by running locally:
- `TestOIDCCli` (the pre-existing happy-path test) broke after our change added `config.OIDCSetting(ctx)` to `Generate()`. In production, the request context carries an ORM session; in tests it doesn't. Fixed by adding `config.InitWithSettings(map[string]any{common.OIDCLoginGroups: ""})` to the test, switching config to an in-memory backend that needs no ORM.
- Removed two unreachable `t.Skip("requires POSTGRESQL_HOST")` guards added earlier in `pkg/oidc/helper_test.go.` These were dead code — `TestMain` fatals before individual tests are reached when `POSTGRESQL_HOST` is unset, so the skip is never evaluated.


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
