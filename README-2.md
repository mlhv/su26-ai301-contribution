# Contribution #2: Switch in_queue and processing internal spans to logs

**Contribution Number:** 2
**Student:** Minh Le
**Issue:** https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation/issues/1193
**Status:** Phase 1 [Complete]

---

## Why I Chose This Issue

The issue interests me as it asks for a new feature request, and will likely involve looking through the whole codebase, and interacting with APIs; which is exactly what I'm looking for as I'm aiming for backend/infrastructure roles. The codebase itself is written in Go and Typescript which is also 2 of my most used languages (and I want to gain more expertise in them) 

The feature itself asks for OIDC configs, which means that I would have to research about how OIDC works under the hood and how it all ties in with authentication (and a little bit of authorization from how it ties into OAuth 2.0). This will allow me to understand more about how security and auth is implemented from a deeper level. I hope to learn more about OIDC security protocols, and how to better navigate a large Go codebase.

---

## Understanding the Issue

### Problem Description

There is no functionality to restrict users who are not in any Harbor groups to be able to login into specific Keycloak groups. (aka limit login to specific groups based on Keycloak OIDC configurations)

### Expected Behavior

According to Keycloak OIDC configs, users should be/should not be able to access or login into specific Keycloak groups.

### Current Behavior

All users (especially ones who are not in any harbor group) can log into configured group scopes from Keycloak.

### Affected Components

- `src/common/const.go` — OIDC config key constants
- `src/lib/config/metadata/metadatalist.go` — config key registry
- `src/lib/config/models/model.go `— OIDCSetting struct
- `src/pkg/oidc/helper.go` — OIDC token parsing and group logic
- `src/pkg/oidc/helper_test.go` — unit tests for OIDC helpers
- `src/core/controllers/oidc.go` — OIDC callback handler (login enforcement point)
- `src/portal/src/app/base/left-side-nav/config/auth/config-auth.component.html` — admin UI
- `src/portal/src/app/base/left-side-nav/config/config.ts` — frontend config model
- `src/portal/src/i18n/lang/en-us-lang.json — UI strings`

---

## Reproduction Process

### Environment Setup

I first had to fork the repo and then follow the Contribution.md file that they provided. Then I created my own make/harbor.yml file based on the provided template which involved adding my own hostname, specifying whether an https connection was needed, as well as the default data volumes for the application once we ran the make build. (NOTE: harbor had macOS bugs for Docker Desktop so I had to open separate PRs to fix those!)

### Steps to Reproduce

1. Build Harbor from the official Golang image: make install COMPILETAG=compile_golangimage
2. Start a local Keycloak instance via Docker
3. In Keycloak: create a harbor realm, a harbor client with client authentication enabled, a harbor group, user alice (member of harbor group), and user bob (no groups). Add a Group Membership mapper with token claim name groups, full group path OFF.
4. Configure Harbor's OIDC settings via the API to point at the Keycloak realm, with oidc_groups_claim: groups
5. Log in as alice via the OIDC button → succeeds
6. Log in as bob via the OIDC button → also succeeds (this is the bug)


### Reproduction Evidence

- **Commit showing reproduction:** [\[Link to commit in your fork\]](https://github.com/goharbor/harbor/pull/23371/changes/0fd4ad5d7b4a82fd3ccad8b4be69c73909a69b68)
NOTE: This is a commit showing my attempts at reproducing the issue, but really its just the bugs/attempts in setting up the dev environments which required me to commit fixes and a PR. Reproducing the actual issue required me to tinker with the OIDC configs in harbor more than the actual codebase.
- **Actual fix branch going forward**: https://github.com/mlhv/harbor/tree/feat/oidc-login-groups
- **My findings:** 
macOS is using VirtioFS on Docker Desktop. This conflicted with the setup configurations for Harbor's container as they tried mounting directories inside one another. I had to create a PR and modify the source code to support this. P.S there is no official arm64 image of harbor yet so this was to be expected.

---

## Solution Approach

### Analysis

Harbor is reading the groups token claim in the JWT (from Keycloak) and currently does not do anything to restrict login. 

Harbor already has two group-related config fields that follow the same pattern we need:
- `oidc_admin_group` — a single group name; members get sysadmin rights
- `oidc_group_filter` — a regex; only matching groups are stored in the DB

Neither of these gates login. The JWT group data is already being extracted correctly — the only missing piece is a check
before the session is created.

The enforcement point is `Callback()` in `src/core/controllers/oidc.go`, specifically after `oidc.UserInfoFromToken()` returns (groups are available) and before oc.PopulateUserSession() is called (session is created).

### Proposed Solution

If groups is empty, reject users' login.

Add a new config field `oidc_login_groups` (comma-separated group names). After Harbor extracts user info from the OIDC token, check whether the user's groups intersect with the configured allowed groups. If `oidc_login_groups` is non-empty and the user has no matching group, return a 401. If `oidc_login_groups` is empty, all users are allowed (fully backward compatible).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Harbor needs a configurable allowlist of OIDC groups. Users not in any listed group must be denied login before a session is established.

**Match:** The pattern mirrors `oidc_admin_group` — a config string that Harbor reads at callback time and compares against the user's extracted groups. The f`ilterGroup()` function in `helper.go` shows the existing pattern for group-based logic. The `userInfoFromClaims()` function already
sets `info.hasGroupClaim` which tells us whether the token included a groups claim at all.

**Plan:** [Step-by-step implementation plan]
1. Add constant OIDCLoginGroups = "oidc_login_groups"
2. Register new key as a StringType in OIDC group
3. Add LoginGroups string field to OIDCSetting struct
4. Write failing tests for IsLoginAllowed() covering: user allowed in group, no user allowed, no group claim present, etc...
5. Implement IsLoginAllowed function
6. In Callback(), after UserInfoFromToken(), call config.OIDCSetting() and oidc.IsLoginAllowed().
7. Add oidc_login_groups
8. Modify frontend UI

**Implement:** https://github.com/mlhv/harbor/tree/feat/oidc-login-groups

**Review:** 
- [ ] Follows existing OIDC config patterns (oidc_admin_group, oidc_group_filter)
- [ ] Backward compatible — empty field = no restriction
- [ ] Unit tests for the new IsLoginAllowed() function
- [ ] Signed commits (-s flag) per Harbor DCO requirement
- [ ] No changes to unrelated files

**Evaluate:** 
1. Run unit tests
2. Set oidc_login_groups: harbor via API, log in as alice (in harbor group) → succeeds
3. Log in as bob (no group) → receives 401 "user is not a member of any authorized OIDC group"
4. Clear oidc_login_groups to empty, log in as bob → succeeds (backward compatibility confirmed)

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
