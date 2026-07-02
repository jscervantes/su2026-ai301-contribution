# Contribution [1]: [React Doctor] no-noninteractive-tabindex: Don't add tabIndex to non-interactive elements — keyboard users would have ... (1 occurrence)

**Contribution Number:** 1 
**Student:** Jose Cervantes 
**Issue:** https://github.com/wso2/product-is/issues/27930
**Status:** Phase IV [Completed]

---

## Why I Chose This Issue

This issue looked like an approachable first contribution: a single flagged occurrence of an
accessibility lint rule, scoped to one component, in a large and well-maintained open-source
codebase. That combination let me focus on learning the *process* of contributing to WSO2
(forking, branching, changesets, the PR template, the CLA) without getting lost in a sprawling
change.

It also turned out to be a good lesson in not judging an issue by its size. What looked like a
one-line "delete the `tabIndex`" fix was actually an accessibility trap: deleting the attribute
would silence the linter but break keyboard access. Working through it taught me how ARIA roles,
focus order, and keyboard event handling fit together, and how a real codebase encodes those
conventions.

---

## Understanding the Issue

### Problem Description

React Doctor flags one accessibility violation (jsx-a11y/no-noninteractive-tabindex): a non-interactive element has been given a tabIndex, placing it in the keyboard tab order even though it isn't a real control. The rule's guidance is to either remove the tabIndex from non-interactive elements, or turn them into genuine interactive controls (with a role and keyboard handling) so keyboard users get the behavior they'd expect when they land on it.

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

Setting up the WSO2 development environment involved installing OpenJDK 23, Apache Maven, and the
latest WSO2 Identity Server release. That part was straightforward.

The important discovery during setup: the repo's own lint (`pnpm lint`) does **not** flag this
issue. `eslint.config.js` requires `eslint-plugin-jsx-a11y` and registers it under `plugins`, but
the `rules` block never enables any `jsx-a11y/*` rule (unlike the testing-library / jest-dom rule
sets, which are spread in). So the plugin is loaded but inert. That means "run the project's linter"
is not a faithful reproduction — I had to reproduce the exact rule React Doctor ran in isolation.

### Steps to Reproduce


1. Confirmed `tabIndex={ 0 }` on line 491 of
   `features/admin.organizations.v1/components/organization-switch/organization-switch-breadcrumb.tsx`.
2. Built a throwaway ESLint harness that enables only the single flagged rule, using a flat config:

   ```js
   // eslint.config.js
   const jsxA11y = require("eslint-plugin-jsx-a11y");
   const tsParser = require("@typescript-eslint/parser");

   module.exports = [{
       files: ["**/*.tsx"],
       languageOptions: {
           parser: tsParser,
           parserOptions: { ecmaFeatures: { jsx: true }, sourceType: "module" }
       },
       plugins: { "jsx-a11y": jsxA11y },
       rules: { "jsx-a11y/no-noninteractive-tabindex": "warn" }
   }];
   ```

3. Ran the isolated rule against the file:

   ```
   node /tmp/rd-repro/node_modules/eslint/bin/eslint.js \
     --config /tmp/rd-repro/eslint.config.js --no-config-lookup \
     features/admin.organizations.v1/components/organization-switch/organization-switch-breadcrumb.tsx
   ```

4. **Observed result:**

   ```
   .../organization-switch-breadcrumb.tsx
     491:33  warning  `tabIndex` should only be declared on interactive elements  jsx-a11y/no-noninteractive-tabindex

   ✖ 1 problem (0 errors, 1 warning)
   ```

#### What I'm doing here:

Setting up a throwaway ESLint that runs only React Doctor's flagged rule, then pointing it at the
file in the issue. This reproduces exactly what React Doctor saw, independent of the repo's own
(inert) a11y config.


### Reproduction Evidence

**My findings:** `tabIndex={ 0 }` sits on a non-interactive `<div className="organization-breadcrumb">`
at line 491. The same `<div>` also carries `onClick` (toggling `isDropDownOpen`) and `onBlur`, but
no `role` and no keyboard handler — so it's a mouse-only control that's nonetheless in the tab
order. Extending the harness to the sibling rules also showed `click-events-have-key-events`
firing on this element (line 490), confirming the element is the real problem, not just the
`tabIndex` attribute.

---

## Solution Approach

### Analysis

The root cause is that the element **is** functionally interactive — `onClick` toggles the
org-switch dropdown — but it was made focusable and clickable **without** being given an interactive
role or keyboard activation. So the bug is not "`tabIndex` shouldn't be here"; it's that a
click-handling `<div>` was never turned into a real control. That's why the naive fix (delete
`tabIndex`) is wrong: it would remove keyboard reachability from a control that keyboard users need.

A subtlety I had to get right: this component's local `isDropDownOpen` state (declared at line 75)
only drives the breadcrumb chevron icon (line 328). The **actual** dropdown menu open/close state
lives in `TenantDropdown` (`tenant-dropdown.tsx:198`, driving `<Dropdown open={…} />` at `:784`),
which receives the breadcrumb as its `dropdownTrigger`. The two states usually move together but can
drift, so any keyboard handler must follow the *same event path as a real click* rather than poking
the local state directly.

### Proposed Solution

Turn the focusable `<div>` into a legitimate dropdown-trigger control:

- Add `role="button"` so the `tabIndex` is justified and screen readers announce it as a control.
- Add `aria-haspopup="true"` so assistive tech identifies it as a dropdown trigger.
- Add an `onKeyDown` handler so **Enter** and **Space** activate the same path as a mouse click,
  via `e.currentTarget.click()` — this lets the click bubble to the Semantic UI `Dropdown` that owns
  the real menu state, instead of only flipping the local chevron state.
- **Do not** add `aria-expanded`: binding it to the local `isDropDownOpen` would announce a stale
  "expanded" state whenever the real menu closes through a `TenantDropdown`-internal path the
  breadcrumb never observes — arguably worse than omitting it.

This satisfies `no-noninteractive-tabindex` and pre-empts the sibling rules
(`click-events-have-key-events`, `interactive-supports-focus`) that a role-less clickable div would
otherwise trip.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** A non-interactive `<div>` carries `tabIndex={ 0 }` + `onClick` but no interactive
role and no keyboard handler. It's reachable by Tab yet exposes no role/behavior to assistive tech.
Make it a real control without losing keyboard reachability.

**Match:** The repo already uses this exact pattern. The closest precedent is the clear-search
control in `features/common.workflow-approvals.v1/pages/approvals.tsx:472-487`:

```tsx
role="button"
tabIndex={ 0 }
onClick={ handleSearchClear }
onKeyDown={ (e: KeyboardEvent<HTMLElement>): void => {
    if (e.key === "Enter" || e.key === " ") {
        e.preventDefault();
        handleSearchClear();
    }
} }
```

I copied this house style (including `e.preventDefault()` on Space to stop page scroll, and the
`e.key === " "` spelling the repo uses) so the change stays idiomatic and review-friendly.

**Plan:**
1. Edit `organization-switch-breadcrumb.tsx` (the `triggerOrganizationDropdown` render, ~line 490):
   add `role="button"`, `aria-haspopup="true"`, and an `onKeyDown` handler that activates the same
   behavior as click on Enter/Space (`e.currentTarget.click()`).
2. Add `KeyboardEvent` to the file's React import block (lines 32-39) so the handler is typed like
   the precedent.
3. Add a `patch` changeset under `.changeset/` for `@wso2is/admin.organizations.v1` describing the
   a11y fix (the repo uses `@changesets/cli`).
4. No unit test currently asserts this markup, and a behavioral test isn't expected for a
   one-attribute a11y fix; verification is via the isolated linter run + a manual keyboard check.

Proposed edit:

```tsx
<div
    role="button"
    aria-haspopup="true"
    tabIndex={ 0 }
    onBlur={ () => setIsDropDownOpen(false) }
    className="organization-breadcrumb"
    onClick={ () => setIsDropDownOpen(!isDropDownOpen) }
    onKeyDown={ (e: KeyboardEvent<HTMLDivElement>): void => {
        if (e.key === "Enter" || e.key === " ") {
            e.preventDefault();
            e.currentTarget.click();
        }
    } }
>
```

**Implement:**
- Fork: https://github.com/jscervantes/identity-apps
- Branch: `fix/org-breadcrumb-noninteractive-tabindex` (off `master`)
- Commits: [links to commit(s) as work proceeds]

**Review (self-checklist):**
- [x] Isolated ESLint run for `no-noninteractive-tabindex` at `:491` now reports 0 warnings.
- [x] Sibling `click-events-have-key-events` on this element is also resolved; no
      `interactive-supports-focus` warning introduced.
- [x] `KeyboardEvent` import added; handler is typed.
- [x] Change follows the repo's precedent pattern (`approvals.tsx`).
- [x] `patch` changeset added for `@wso2is/admin.organizations.v1`.
- [ ] Manual keyboard verification in the running console (see Manual Testing).
- [ ] CLA signed; PR template action items completed.

**Evaluate:** Re-run the isolated rule (must print 0 warnings), and manually verify keyboard parity
in the running console — Enter/Space open the real `TenantDropdown` menu and menu items can be
reached and activated by keyboard.

---

## Testing Strategy

### Unit Tests

- [ ] No existing unit test asserts this component's markup, and a dedicated test isn't warranted
      for a one-attribute a11y fix — noting this explicitly rather than adding a low-value test.
      (Will confirm against reviewer feedback.)

### Integration Tests

- [ ] Integration coverage for the org-switch dropdown is out of scope for this change; the fix
      doesn't alter data flow, only keyboard/ARIA semantics.

### Manual Testing

Static (primary, deterministic) — the isolated ESLint harness:

- **Before fix:** `491:33 warning … no-noninteractive-tabindex` (+ `click-events-have-key-events`
  at line 490 when sibling rules are enabled).
- **After fix:** 0 warnings for the target element. (Two pre-existing `click-events-have-key-events`
  warnings remain elsewhere in the file, on unrelated elements — out of scope for this single-issue
  fix.)

Manual keyboard test (in the running console — `pnpm install && pnpm build`, then
`cd apps/console && pnpm start`):

1. Tab to the org-switch breadcrumb; confirm it's announced as a button / dropdown trigger.
2. Press Enter, then Space; confirm each opens the **actual** `TenantDropdown` menu (not just the
   chevron flip).
3. **Open question to verify, not assume:** the trigger keeps `onBlur={ () => setIsDropDownOpen(false) }`
   and Semantic UI's `Dropdown` has its own `onBlur`. After opening with Enter, Tab toward the menu
   items and confirm focus leaving the trigger does **not** close the menu before an item is
   reached/activated. If it does, keyboard parity isn't truly achieved and the `onBlur` needs a
   follow-up.

<img width="1509" height="910" alt="image" src="https://github.com/user-attachments/assets/b7e59ff8-6178-4541-9c61-e03e57581490" />

---

## Implementation Notes

### Week 5 Progress

Reproduced the issue with an isolated ESLint harness, confirmed the repo's own lint doesn't enforce
`jsx-a11y`, traced the breadcrumb → `TenantDropdown` state relationship, and implemented the fix
following the `approvals.tsx` precedent. Verified the warning is resolved statically.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:**
  - `features/admin.organizations.v1/components/organization-switch/organization-switch-breadcrumb.tsx`
    (added `role="button"`, `aria-haspopup="true"`, `onKeyDown` handler; added `KeyboardEvent`
    import).
  - `.changeset/a11y-org-breadcrumb-tabindex.md` (new; `patch` bump for
    `@wso2is/admin.organizations.v1`).
- **Key commits:** [links]
- **Approach decisions:**
  - Chose `e.currentTarget.click()` over calling `setIsDropDownOpen` directly, so keyboard
    activation follows the same event path as a mouse click and reaches the `TenantDropdown` that
    owns the real menu state.
  - Deliberately omitted `aria-expanded` to avoid announcing a stale expanded state, since the local
    breadcrumb state can drift from the real menu state.

---

## Pull Request

**PR Link:** [[GitHub PR URL when submitted]](https://github.com/wso2/identity-apps/pull/10501)

**PR Description:** Fix the `jsx-a11y/no-noninteractive-tabindex` accessibility warning (reported by React Doctor) on
the organization-switch breadcrumb trigger in
`features/admin.organizations.v1/components/organization-switch/organization-switch-breadcrumb.tsx`.

The trigger was a non-interactive `<div>` carrying `tabIndex={ 0 }` and an `onClick` handler, but no
interactive `role` and no keyboard handler. As a result it was reachable by Tab yet exposed no role
or keyboard behavior to assistive technology, and was effectively mouse-only (Enter/Space did
nothing once focused).

Rather than simply removing `tabIndex` — which would silence the linter but make the control
unreachable by keyboard (a regression) — this PR turns the element into a legitimate dropdown
trigger:

- `role="button"` so the `tabIndex` is justified and screen readers announce it as a control.
- `aria-haspopup="true"` so assistive tech identifies it as a dropdown trigger.
- An `onKeyDown` handler that activates on **Enter**/**Space** via `e.currentTarget.click()`, so
  keyboard activation follows the same event path as a mouse click and reaches the `TenantDropdown`
  that owns the real menu open/close state.

`aria-expanded` is intentionally omitted: the breadcrumb's local `isDropDownOpen` state only drives
the chevron icon and can drift from the actual `TenantDropdown` menu state, so binding
`aria-expanded` to it could announce a stale "expanded" state. This follows the existing keyboard
button pattern in `features/common.workflow-approvals.v1/pages/approvals.tsx`.


**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review]

---

## Learnings & Reflections

### Technical Skills Gained

- How `jsx-a11y` accessibility rules work and how ARIA roles (`role="button"`, `aria-haspopup`),
  focus order (`tabIndex`), and keyboard handlers (`onKeyDown`) combine to make an element a *real*
  control.
- Reading an unfamiliar component to trace ownership of state across components (breadcrumb vs.
  `TenantDropdown`) before changing behavior.
- WSO2's contribution mechanics: forking, changesets (`@changesets/cli`), the PR template, and the
  CLA.

### Challenges Overcome

- Realizing the "obvious" fix (delete `tabIndex`) was wrong and would introduce an accessibility
  regression.
- Discovering that the repo's own `pnpm lint` doesn't enforce `jsx-a11y`, and building an isolated
  harness to reproduce exactly what React Doctor reported.

### What I'd Do Differently Next Time

The most challenging part here was scoping the issue and understanding the contribution process for this project specifically. Next time, I would set my ego aside and ask the maintainers more questions about where PRs should be submitted and how they should be formatted.

---

## Resources Used

- Issue: https://github.com/wso2/product-is/issues/27930
- `eslint-plugin-jsx-a11y` — `no-noninteractive-tabindex` rule docs:
  https://github.com/jsx-eslint/eslint-plugin-jsx-a11y/blob/main/docs/rules/no-noninteractive-tabindex.md
- WSO2 identity-apps `CONTRIBUTING.md` and root `pull_request_template.md`
- WSO2 developer docs: https://wso2.github.io/
- In-repo precedent: `features/common.workflow-approvals.v1/pages/approvals.tsx`
