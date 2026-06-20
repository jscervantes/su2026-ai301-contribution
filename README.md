# Contribution [1]: [React Doctor] no-noninteractive-tabindex: Don't add tabIndex to non-interactive elements — keyboard users would have ... (1 occurrence)

**Contribution Number:** 1 
**Student:** Jose Cervantes 
**Issue:** https://github.com/wso2/product-is/issues/27930
**Status:** Phase II [In Progress]

---

## Why I Chose This Issue

This issue seems like an easy win. Refactoring what seems to be a single occurance in the codebase.

---

## Understanding the Issue

### Problem Description

A React accessibility warning from react-doctor reports a non-interactive element in the code has a tabIndex set (1 violation). It recommends removing tabIndex from non-interactive elements or giving them an interactive role so keyboard users get expected behavior.

### Expected Behavior

Non-interactive elements should not be focusable. Only elements that provide an interactive behavior (links, buttons, controls) should receive keyboard focus (via native semantics or tabIndex when appropriate).
Breadcrumb / organization-switch items should be implemented with proper semantics (e.g., <a> or <button>) so keyboard users can both focus and activate them with expected keyboard interactions.
Accessibility linters (react-doctor) should not report react-doctor/no-noninteractive-tabindex for this component.

### Current Behavior

A non-interactive element in admin.organizations.v1/components/organization-switch/organization-switch-breadcrumb.tsx (reported at line ~491) has a tabIndex (e.g., tabIndex={0}) applied, making it focusable but without corresponding interactive behavior or keyboard handlers. When keyboard users focus that element, nothing happens or behavior is unexpected — this triggers the react-doctor/no-noninteractive-tabindex warning.
This creates an accessibility regression: focusable elements that are not interactive confuse keyboard and assistive technology users.

### Affected Components

admin.organizations.v1/components/organization-switch/organization-switch-breadcrumb.tsx (primary)
organization-switch UI (breadcrumb rendering and any parent component that composes the breadcrumb)
Any styles or click/keyboard handlers tied to breadcrumb items (if they assume non-focusable semantics)
Accessibility linting/config for the frontend (react-doctor rule enforcement)

---

## Reproduction Process

### Environment Setup

Setting up the WSO2 development environment involved installing OpenJDK 23, Apache Maven, and the most recent release of WSO2. Pretty straightforward.

### Steps to Reproduce

1. I was able to confirm `tabIndex={ 0 }` on line 491 of `features/admin.organizations.v1/components/organization-switch/organizat
  ion-switch-breadcrumb.ts`
2. I ran:

```
~/P/c/a/identity-apps master ❱ node /tmp/rd-repro/node_modules/eslint/bin/eslint.js --config  /tmp/rd-repro/eslint.config.js --no-config-lookup features/admin.organizations.v1/components/organization-switch/organization-switch-breadcrumb.tsx
```
And got:
```
/****/features/admin.organizations.v1/components/organization-switch/organization-switch-breadcrumb.tsx
  491:33  warning  `tabIndex` should only be declared on interactive elements  jsx-a11y/no-noninteractive-tabindex

✖ 1 problem (0 errors, 1 warning)
```
#### What I'm doing here:

Setting up a throwaway ESLint that runs only on React Doctor's flagged specfied rule, then pointing it at the file in the issue. We are essentially reproducing what React Doctor saw.

### Reproduction Evidence

- **My findings:** `tabIndex={0}` is inside of a non-interactive `<div>` on line 491.

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
