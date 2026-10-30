---
name: branch-test-coverage
description: Add or extend unit tests (in English) for the code changed on the current branch, following the repo's existing test structure and patterns, to raise coverage of the changed lines. Edits ONLY test files. Use when asked to "add tests for my changes", "improve coverage on this branch", or invoked as /branch-test-coverage [base].
---

# Branch test coverage

Goal: for every source file changed on this branch, cover the **changed lines that are not yet covered**, writing tests the way this repo already writes them. Source code is read-only.

## Language

Always in **English**, regardless of the conversation's language — same rule as the `pr-summary` and `ticket-plan` skills: the output must be shareable with any teammate. This covers everything the skill produces:

- Test code: `describe`/`it` names, comments, fixture data labels, mock names.
- The plan (step 4) and the final report (step 7), so they can be pasted into a PR or ticket as-is.

This is a hard rule, not a default to be overridden by matching the user's language. Only the short conversational lines around those artifacts (e.g. a quick follow-up question) may use the user's language.

## Hard rules

- **Write only test files**: paths matching `__tests__/`, `*.test.*` or `*.spec.*`, plus new fixtures/mocks inside `__tests__/` or `__mocks__/`. Never edit source, configs, `package.json`, lockfiles or snapshots of unrelated tests.
- **Don't weaken existing tests**: add new `it`/`describe` blocks; don't change or delete existing assertions. If an existing test must change, ask first.
- **A failing new test is a finding, not something to "fix"**: if a test fails because the source behaves unexpectedly, keep the test out, report the case (input → actual vs expected) and let the user decide. Never change source to make a test pass.
- **Approval gate**: present the plan (step 4) and wait for the user's go-ahead before writing anything.
- Coverage output goes to the session scratchpad, never into the repo.

## Steps

### 1. Scope the diff

- Detect the base branch (`git symbolic-ref refs/remotes/origin/HEAD`, fall back to `main`); the user may pass one as an argument.
- Changed files, from three sources (merge and dedupe):
  - Committed on the branch: `git diff --name-only <base>...HEAD`
  - Uncommitted changes to tracked files: `git diff --name-only HEAD`
  - New files not yet added to git: `git ls-files --others --exclude-standard`
- Check whether the branch has commits of its own: `git rev-list --count <base>..HEAD`. If it's `0`, say so plainly in one line, e.g. "This branch has no commits of its own yet, so I'm using the uncommitted and untracked changes." If all three sources are empty, stop and tell the user there's nothing to cover.
- Keep source files only. Drop tests, stories, `*.d.ts`, `__mocks__`, configs, generated types, pure type/constant files with no logic. List the files kept, and the ones dropped with a short reason.
- Changed line ranges per file: `git diff -U0 <base>...HEAD -- <file>` plus `git diff -U0 HEAD -- <file>` (parse the `@@ +start,count @@` hunks). Untracked files count as fully changed (every line).

### 2. Find the test project for each file

- Walk up from each file to the nearest folder with `package.json` + a jest/vitest config. Repos can contain several standalone projects (e.g. `bff/` and `mfe/` each with their own `jest.config.ts` and lockfile) — run tests from that folder, not the repo root.
- Read that config: `rootDir`, `testMatch`, `moduleNameMapper` (path aliases to use in imports), `setupFilesAfterEnv`, `testEnvironment`, `coverageThreshold`.

### 3. Learn the local conventions

For each changed file, read the 2–3 nearest existing tests (same folder's `__tests__/` first, then siblings of the same kind: controllers with controllers, hooks with hooks, components with components). Extract and **follow**:

- Location: `__tests__/` folder vs next to the source.
- Suffix: `.spec` vs `.test` — use what the folder already uses (repos are often mixed; follow the folder majority, not a global rule).
- File naming: `<SourceName>.spec.tsx` etc.
- Structure: `describe` per function/component, `it` wording style, `beforeEach`/`afterEach` resets.
- Mocking: `jest.mock` of modules, `jest.spyOn`, MSW handlers, shared fixtures, `__mocks__` folders, custom render helpers / providers (e.g. React Query wrapper), setup file globals.
- Assertions: Testing Library queries in use (`getByRole` vs `getByTestId`), `toHaveBeenCalledWith` style, snapshot usage.
- Console output: if the code logs via `console.error`/`console.warn`, assert with `jest.spyOn(console, 'error').mockImplementation(() => {})` (restore after each test) so expected logs are checked without polluting test output.

Also check whether each changed source file **has module-level state or runs code on import** (top-level initialization, singletons, reading `process.env` at import time, caches). If so, plan to load it fresh per case with `jest.isolateModules` (or `jest.resetModules()` + `require`/dynamic `import` inside the test) and set env vars/mocks *before* loading it; otherwise cases leak state into each other. Call this out in the plan.

Summarize the conventions found in 3–6 bullets in the plan so the user can correct them.

### 4. Baseline coverage + plan (stop for approval)

Run, from the test project folder, only the tests related to the changed files:

```bash
npx jest --findRelatedTests <files...> \
  --coverage --collectCoverageFrom=<each file, relative to rootDir> \
  --coverageReporters=json --coverageDirectory=<scratchpad>/coverage-before \
  --coverageThreshold='{}' --passWithNoTests
```

- `--coverageThreshold='{}'` matters: projects with a global threshold fail when measuring a subset.
- Use the project's package manager to run jest (`pnpm exec jest` if there's a `pnpm-lock.yaml`).
- From `coverage-final.json` get uncovered statements and branches, then **intersect with the changed line ranges**. That intersection is the target; uncovered lines outside the diff are out of scope (mention only if trivial).

Present the plan:

| File | Changed lines | Changed lines covered (before) | Uncovered changed lines | Test file (new / existing) | Proposed cases |
|---|---|---|---|---|---|

- "Changed lines covered (before)" = `X/Y (Z%)`: of the Y executable changed lines, X are already covered. Count only executable lines from `coverage-final.json`'s statement map, not blank lines or type declarations.
- Add a **Total** row summing X/Y across files. The final report repeats this metric as before → after.

Each proposed case = one line: what it exercises (branch/condition/error path) and expected result. Flag lines that can't be reached without changing source (e.g. unexported helper, hardcoded side effect) instead of proposing hacks.

Then ask for approval in English: "Approve this plan, or adjust anything before I write the tests?"

### 5. Write the tests

- Add cases to the existing test file if one exists; otherwise create it in the conventional location.
- Reuse existing mocks/fixtures/helpers before creating new ones.
- Test behavior through the public API (exported function, rendered component, HTTP handler), not internals.
- One behavior per `it`; names describe the behavior, matching the folder's wording style.

### 6. Verify

1. Run the touched test files: all must pass (existing and new). Run twice if anything time-based or async is involved, to catch flakiness.
2. Type-check / lint the test files if the project has those scripts.
3. Re-run step 4's coverage command into `<scratchpad>/coverage-after`.
4. `git status` must show only test files changed — if anything else appears, revert it and report.

### 7. Report

- Before/after per file: % lines and branches, and **changed lines covered X/Y**.
- Tests added (file + case names).
- Remaining uncovered changed lines and why (unreachable, needs refactor, integration-level).
- Findings: any case where the source behaved unexpectedly (from the hard rules).
- What the new tests don't prove (e.g. mocked HTTP clients don't exercise the real network/library), with one or two concrete extra checks if relevant.

Don't commit; leave the changes for the user to review.