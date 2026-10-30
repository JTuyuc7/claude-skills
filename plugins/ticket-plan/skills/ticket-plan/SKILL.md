---
name: ticket-plan
description: Create or update a per-ticket work plan in a shared cross-repo work journal (default ~/work-journal/, override with the WORK_JOURNAL_DIR env var). Use when starting work on a ticket/spike, when its PR is created, when the PR is merged, or when a review comment needs follow-up. Invoked manually as "/ticket-plan <subcommand> ..." — never triggers automatically on git/gh events.
---

# Ticket plan

Maintains a lightweight, English-language work journal shared across all your repos, so plans and decisions for one repo's tickets are readable from a session working in a different repo without digging through that other repo's local Claude memory.

## Journal location

Root (`<journal>` below): the value of the `WORK_JOURNAL_DIR` environment variable if set (check with `echo "$WORK_JOURNAL_DIR"`), otherwise `~/work-journal/`. Expand `~` to the real home directory. If the root doesn't exist yet, tell the user which path will be used and create it on first `start`.

Layout: `<journal>/<repo-name>/<TICKET-ID>-<short-slug>/plan.md`, plus a root `<journal>/index.md` that lists every repo and, per repo, a table of its tickets with status.

This is always in **English**, regardless of the conversation's language — same rule as the `pr-summary` skill: the artifact must be shareable with any teammate.

## Subcommands

Parse `args` for one of: `start`, `pr-created`, `merged`, `review-comment`. If no subcommand is given, ask which one.

### Resolving repo and ticket

- **Repo name**: basename of the git top-level dir (`git rev-parse --show-toplevel`) of the current working directory. If not a git repo, ask the user.
- **Ticket ID**: if passed explicitly in args, use it. Otherwise infer from the current git branch name — look for a tracker key like `ABC-1234` (Jira style) or an issue number like `123-...` / `issue-123` (GitHub style). If ambiguous or no git repo, ask the user. For `pr-created` / `merged` / `review-comment`, the ticket folder must already exist (created by `start`); if it doesn't, run the `start` flow first using whatever context is available, then continue.

### `start <TICKET-ID> [short description]`

1. Resolve repo name and ticket ID as above. Build the slug: `<TICKET-ID>-<kebab-case-short-description>` (ask for a short description if none given — 2-5 words, e.g. `analytics-product-page`).
2. Create `<journal>/<repo-name>/<slug>/plan.md` from this template, filled in with what's actually known from the conversation (don't leave placeholder brackets unfilled — ask if something load-bearing is missing, otherwise omit the section):

   ```markdown
   # <TICKET-ID>: <title>

   **Repo:** <repo-name>
   **Branch:** <branch name>
   **Status:** In progress
   **Started:** <today's date>

   ## Goal

   <1-3 sentences: what this ticket/spike is trying to accomplish and why>

   ## Related context

   <links to other journal tickets or shared knowledge folders that this ticket builds on or should update — omit if none>

   ## Approach

   <bullet list of the planned approach, as currently understood — this is a living section, update it as the approach evolves, don't just append>

   ## Open questions

   <bullet list — omit section if none>

   ## Follow-ups

   <empty at start; this is where review-comment items accumulate>

   ## Timeline

   - <today's date>: ticket started
   ```

3. Update `<journal>/index.md`: create it from the template below if missing, add the repo section if it doesn't exist yet, add a row to that repo's ticket table linking to the new `plan.md`, status "In progress".
4. Confirm the path created, in one line.

### `pr-created <PR URL>`

1. Resolve repo/ticket, locate its `plan.md`.
2. Update **Status** to `In review`, append to **Timeline**: `<today's date>: PR created — <PR URL>`.
3. Add a `## Pull request` section (if not present) with the URL.
4. Update the status cell for this ticket in `<journal>/index.md`.

### `merged`

1. Resolve repo/ticket, locate its `plan.md`.
2. Update **Status** to `Merged`, append to **Timeline**: `<today's date>: merged`.
3. Update the status cell in `<journal>/index.md`.
4. If the **Follow-ups** section has unchecked items, point that out — merging doesn't clear them.

### `review-comment "<comment text or summary>"`

1. Resolve repo/ticket, locate its `plan.md`.
2. Append an unchecked item to **Follow-ups**: `- [ ] <comment summary> (<today's date>)`. Summarize the comment in one line if it's long — don't paste a huge verbatim block.
3. Don't change **Status** — a comment doesn't mean the PR left review state; use judgment only if the comment text itself says otherwise (e.g. "changes requested").

## `index.md` template (create once, at `<journal>/index.md`)

```markdown
# Work Journal — Index

Cross-repo log of tickets and spikes. Updated manually via the `/ticket-plan` skill at four points: ticket start, PR created, PR merged, review comment to address. Not auto-synced with git/GitHub — if something changed outside this journal, it may be stale; check the linked plan.md and the actual PR/branch before relying on status here.

## <repo-name>

| Ticket | Scope | Status | Plan |
|---|---|---|---|
| <TICKET-ID> | <short description> | <status> | [plan.md](<repo-name>/<slug>/plan.md) |
```

One `##` section per repo, one row per ticket, most recently updated ticket near the top of each repo's table. If the user keeps topic-specific knowledge folders (not per-ticket), link them under the intro paragraph.

## Notes

- This skill never runs on its own on a `git push`/`gh pr create`/`gh pr merge` — the user (or the assistant, when explicitly asked to run it after performing that action) invokes the matching subcommand by hand. Don't claim a plan was auto-updated unless this skill was actually invoked.
- Keep `plan.md` entries terse — this is a journal to resume context quickly, not a design doc. If a decision needs real depth (architecture rationale, spike numbers), it belongs in a shared knowledge folder, with a link from here.