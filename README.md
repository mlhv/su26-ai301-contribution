# Contribution #1: Question: Keycloak OIDC Configuration, limit login to specific groups

**Contribution Number:** 1
**Student:** Minh Le
**Issue:** https://github.com/goharbor/harbor/issues/22730
**Status:** Phase 1 [In Progress]

---

## Why I Chose This Issue

The issue interests me as it asks for a new feature request, and will likely involve looking through the whole codebase, and interacting with APIs; which is exactly what I'm looking for as I'm aiming for backend/infrastructure roles. The codebase itself is written in Go and Typescript which is also 2 of my most used languages (and I want to gain more expertise in them) 

The feature itself asks for OIDC configs, which means that I would have to research about how OIDC works under the hood and how it all ties in with authentication (and a little bit of authorization from how it ties into OAuth 2.0). This will allow me to understand more about how security and auth is implemented from a deeper level. I hope to learn more about OIDC security protocols, and how to better navigate a large Go codebase.

---

## Understanding the Issue

### Problem Description

There is no functionality to restrict users who are not in any Harbor groups to be able to login into specific Keycload groups. (aka limit login to specific groups based on Keycloak OIDC configurations)

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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
