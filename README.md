# dev-workflow-skills

A small collection of [Claude Code](https://claude.com/claude-code) skills for day-to-day development work. Each skill is its own plugin, so you install only the ones you want.

They work in any repo and follow the same rules: output is always in English (so it can be shared with any teammate), they work from the real state of your branch rather than guesses, and they never take irreversible actions (commit, push, open PRs) on their own.

## Install

In Claude Code:

```
/plugin marketplace add TuyucJaime/claude-skills
```

Then install the ones you want:

```
/plugin install pr-summary@dev-workflow-skills
/plugin install ticket-plan@dev-workflow-skills
/plugin install branch-test-coverage@dev-workflow-skills
```

Or run `/plugin` and pick them from the list. To get updates later: `/plugin marketplace update dev-workflow-skills`.

## Skills

### `pr-summary`

**What it does:** writes a PR description with two sections, **Summary** and **Test plan**, built from the branch's full diff against its base branch.

**Why it's useful:** every PR ends up with the same, reviewer-friendly format. The test plan lists only checks that were actually run (with exact commands), and leaves unchecked boxes for what's still pending, so reviewers know what was and wasn't verified.

**Use it:** "write the PR summary" or `/pr-summary`. It prints the text for you to paste; it doesn't create the PR.

### `ticket-plan`

**What it does:** keeps a lightweight work journal across all your repos: one `plan.md` per ticket (goal, approach, open questions, follow-ups, timeline) plus an `index.md` with the status of every ticket.

**Why it's useful:** when you jump between repos or come back to a ticket after days, the context (what was decided, what's pending, which review comments are still open) is in one place, readable from any repo's session.

**Use it:**

```
/ticket-plan start ABC-123 analytics product page
/ticket-plan pr-created https://github.com/org/repo/pull/42
/ticket-plan review-comment "rename the helper to match the hook"
/ticket-plan merged
```

**Setup (optional):** the journal lives in `~/work-journal/` by default. To use another folder, set `WORK_JOURNAL_DIR` in your shell profile, e.g. `export WORK_JOURNAL_DIR="$HOME/Projects/work-journal"`.

### `branch-test-coverage` (beta)

**What it does:** finds the lines you changed on your branch that aren't covered by tests, learns how the repo already writes tests (folder layout, `.spec` vs `.test`, mocks, helpers, naming), proposes test cases, and, after you approve, writes them. **It only edits test files.**

**Why it's useful:** it targets coverage on what you actually changed, not the whole file, and the new tests look like the rest of the suite. If a new test exposes unexpected behavior in the source, it reports it as a finding instead of changing your code.

**Use it:** "add tests for my changes" or `/branch-test-coverage [base-branch]`. Supports jest and vitest, including repos with several independent test projects (e.g. `frontend/` and `backend/` each with their own config).

**Flow:** scope the diff → learn conventions → measure baseline coverage → **plan (waits for your approval)** → write tests → verify (tests pass, only test files changed) → before/after report.

## Contributing

Each skill lives in `plugins/<name>/skills/<name>/SKILL.md`. To try changes locally before pushing:

```
/plugin marketplace add /path/to/your/clone
```

Validate the manifests with `claude plugin validate .`. Bump `version` in the plugin's `.claude-plugin/plugin.json` when you change a skill.