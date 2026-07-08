# Contribution [#]: [Issue Title]

**Contribution Number:** [1 / 2 / 3]  
**Student:** [Your Name]  
**Issue:** [GitHub issue link]  
**Status:** [Phase I / Phase II / Phase III / Phase IV] [In Progress / Complete]

---

## Why I Chose This Issue

I picked this issue because the problem is concrete and the impact is easy to measure. uv makes up about 64% of Hatch's downloaded wheel size, but a lot of users either already have uv installed or don't need it at all, especially in CI pipelines and containers where the environment gets thrown away after one job. I've run into this myself deploying my own projects through CI, where install times and image sizes actually matter. Fixing something that saves resources for thousands of downstream users without breaking anything for people who want uv's speed felt like a contribution worth making.

It also fits where I am as a developer. The issue is well scoped since the original author already pointed out the internal environments that hardcode uv as the installer, it's tagged as a `good first issue`, and it touches Python packaging internals like optional dependencies and how a tool bootstraps its own environments. I use this stuff every day but I've never worked on it from the maintainer side. Working through how Hatch structures its internal environment configs and building a clean fallback from uv to virtualenv/pip is a good way to actually understand the packaging ecosystem instead of just using it. And since Hatch is a PyPA project with an open linked PR on this issue ([#2325](https://github.com/pypa/hatch/pull/2325)), I get to see how maintainers think about backwards compatibility and review standards on tooling that real people depend on.

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

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
