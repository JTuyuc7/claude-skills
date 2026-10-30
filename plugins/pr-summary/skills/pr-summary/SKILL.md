---
name: pr-summary
description: Generate a pull request summary in English with bullets (Summary + Test plan), scoped to the actual diff on the current branch. Use whenever asked to write up/draft/generate a PR summary or description, in any repo.
---

# PR summary

Generates a pull request summary matching a consistent, review-friendly format — regardless of which repo or what language the conversation with the user was in.

## Format

Always in **English**, always **bullets** — never prose, never the conversation's language if that wasn't English. This is a hard rule, not a default to be overridden by matching the user's language.

```
## Summary

- <bullet per relevant change, most important first>
- ...

## Test plan

- [ ] <verification step, e.g. specific test/lint/typecheck command, coverage, manual check>
- ...

<attribution line, if this session's instructions specify one for PR descriptions — see below>
```

## How to build it

1. Determine the diff scope: everything on the current branch not yet in the repo's base branch (e.g. `git log main..HEAD --oneline` / `git diff main...HEAD` — detect the actual base branch name, don't assume `main`), not just the latest uncommitted change. If the branch's work was already discussed in this conversation, reuse that context instead of re-deriving it from the diff alone.
2. Write **Summary** bullets from the actual change, not the ticket description — each bullet states what changed and, briefly, why it mattered (the defect/limitation/risk it addresses), not a restatement of file names.
3. Reference a ticket/issue key only if it's naturally part of a bullet — don't pad bullets just to cite it.
4. Write **Test plan** as concrete, checkable items: exact commands actually run in this session (detect the repo's package manager/scripts — `pnpm`/`npm`/`yarn` + `test`/`typecheck`/`lint`/`audit` scripts — don't hardcode one toolchain), with pass/fail counts if known, plus any manual verification (story, browser check, screenshot) actually performed. Don't list steps that weren't actually run — mark unchecked (`[ ]`) items only for verification the user still needs to do themselves.
5. Skip empty sections rather than writing "N/A".
6. Attribution line: only add one if this session's own system instructions specify an exact line for PR descriptions (they take precedence over anything here) — don't invent one otherwise.

## Output

Print the summary directly in the response (inside a fenced or plain block) so the user can paste it into `gh pr create --body` or the host's PR UI themselves. Don't create the PR unless the user explicitly asks for that as a separate step.