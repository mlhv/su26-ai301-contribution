# Contribution #1: Question: Keycloak OIDC Configuration, limit login to specific groups

**Contribution Number:** 1
**Student:** Minh Le
**Issue:** https://github.com/goharbor/harbor/issues/22730
**Status:** Phase 3 [In Progress]

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

1. Build goharbor from the official Golang image
2. Run own Keycloak instance
3. Setup Keycloak realms, client (harbor), as well as test users and group. One user should be in a group, while the other is not.
4. Change harbor's OIDC config to use keycloak
5. Login using the 2 test users from step 3. They are both able to log in (which is the bug)

### Reproduction Evidence

- **Commit showing reproduction:** [\[Link to commit in your fork\]](https://github.com/goharbor/harbor/pull/23371/changes/0fd4ad5d7b4a82fd3ccad8b4be69c73909a69b68)
NOTE: This is a commit showing my attempts at reproducing the issue, but really its just the bugs/attempts in setting up the dev environments which required me to commit fixes and a PR. Reproducing the actual issue required me to tinker with the OIDC configs in harbor more than the actual codebase.
- **Actual fix branch going forward**: https://github.com/mlhv/harbor/tree/feat/oidc-login-groups
- **Screenshots/logs:** [If applicable]
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

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
