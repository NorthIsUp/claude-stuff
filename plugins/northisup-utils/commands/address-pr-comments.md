---
description: Address every review comment, suggestion, and CI annotation on a PR
argument-hint: "[PR number]"
---

Address every review comment, suggestion, issue, and annotation on the current PR.

Arguments: $ARGUMENTS (optional PR number; defaults to the PR for the current branch)

## 1. Discover the PR

- If `$ARGUMENTS` contains a number, use it as `PR_NUMBER`.
- Otherwise run `gh pr view --json number,headRefName,baseRefName,url,headRepository,headRepositoryOwner` and extract the number.
- Capture the repo as `OWNER/REPO` from `gh repo view --json nameWithOwner -q .nameWithOwner` — every later `gh api` call uses this, never a hardcoded value.
- Stop with a clear message if no PR is open for this branch.

## 2. Collect every annotation

Fetch in parallel:

- `mcp__github_extensions__get_review_comments` — inline review threads (with `thread_node_id` for resolve). If the GitHub MCP server isn't installed, fall back to `gh api repos/$OWNER/$REPO/pulls/$PR_NUMBER/comments`.
- `gh pr view $PR_NUMBER --json reviews,comments,statusCheckRollup --repo $OWNER/$REPO` — review bodies, top-level PR comments, CI status.
- `gh api repos/$OWNER/$REPO/pulls/$PR_NUMBER/reviews` — full review records with state + body (states: `APPROVED`, `CHANGES_REQUESTED`, `COMMENTED`, `DISMISSED`).
- `gh api repos/$OWNER/$REPO/commits/$HEAD_SHA/check-runs` — CI check-run annotations (failed checks with line-level output).

Four annotation sources — treat each as its own category so nothing slips through:

| Source | Example | Resolvable? | How to reply |
| --- | --- | --- | --- |
| `review_thread` | inline suggestion on line 42 | yes, via `resolve_review_thread` | `gh api repos/$OWNER/$REPO/pulls/$PR/comments/$ID/replies --method POST -f body=...` |
| `review_body` | top-level "Requesting changes: ..." summary | no (can dismiss with `gh api .../dismissals -X PUT`) | `gh pr comment $PR --body ...` referencing the review |
| `issue_comment` | bot CI summary, top-level human comment | no | `gh pr comment $PR --body ...` |
| `check_annotation` | CI failure pointing at file:line | no (re-run check instead) | Fix the underlying code; no explicit reply unless a human summarized it |

Deduplicate by `(source, id)`. Skip anything already `RESOLVED`, already replied to by the current user, or authored by the current user as the original comment.

Print a categorized summary table:

```
PR #N — Annotations
──────────────────────────────
  Review bodies:         N (APPROVED: N, CHANGES_REQUESTED: N, COMMENTED: N, DISMISSED: N)
  Inline threads:        N
  Top-level comments:    N
  CI check failures:     N
  Already resolved:      N (skipped)
──────────────────────────────
  To process:            N
```

## 3. For each unresolved item, do all six steps

Process items one at a time. Never batch these steps across items — finish one completely before starting the next. The six steps apply uniformly to all four sources; only step 3.6 (resolve) varies by source.

### 3.1 — Analyze against the current code

- For inline threads / check annotations: read the file at the exact line pointed at (the code may have moved since the comment was posted).
- For review bodies and top-level comments: extract each distinct concern from the body and map it to a file/line where possible. A single `CHANGES_REQUESTED` review can spawn multiple action items.
- If the line/function referenced no longer exists, note that explicitly — the item may already be addressed.
- Honor anything in the project's own `CLAUDE.md`, `AGENTS.md`, or `.claude/rules/`. Invoke project skills routed there before deciding.

### 3.2 — Decide accept or reject, and react

- Decide ACCEPT or REJECT based on the current code and project conventions. Stale suggestions against already-fixed code count as ACCEPT with "already addressed".
- Post a reaction on the originating comment via `mcp__github_extensions__add_reaction` (or `gh api repos/$OWNER/$REPO/issues/comments/$ID/reactions -f content=...`):
  - ACCEPT — `+1`
  - REJECT — `-1`
  - Needs discussion / question back — `eyes`
- Reactions attach to individual comments. For a `review_body`, react on the review summary comment; for `issue_comment` and `review_thread` items, react on the comment itself.
- For `check_annotation` items there is nothing to react to — skip the reaction and go straight to 3.3.
- The reaction is the first signal so the reviewer sees your stance immediately, before any code changes land.

### 3.3 — Incorporate the suggestion

- ACCEPT + suggestion block: `mcp__github_extensions__apply_suggestion` (or `apply_suggestions_batch` for adjacent suggestions from the same reviewer).
- ACCEPT + change request/question with a concrete fix: edit the code directly. Keep changes minimal and scoped to the comment.
- REJECT: make no code changes.
- Honor project rules while editing — read `CLAUDE.md` and any `.claude/rules/` files. Don't add comments/docstrings on lines you didn't change. Don't refactor surrounding code.

### 3.4 — Verify the outcome

- Re-read the edited file and confirm the change actually addresses the comment (not just compiles).
- For non-trivial changes, run the scoped check the reviewer implied: typecheck, the one test, lint on changed files. Do not run the full test suite unless the comment explicitly asks.
- If verification fails, iterate on the fix. Do not proceed to 3.5 with a broken change.

### 3.5 — Post a resolution comment

- `review_thread`: reply on the thread via `gh api repos/$OWNER/$REPO/pulls/$PR/comments/$COMMENT_ID/replies --method POST -f body="..."` so the reply is threaded, not a floating PR comment.
- `review_body`: post a new PR-level comment via `gh pr comment $PR --body "..."` that explicitly references the review author (e.g. `Re: @reviewer's review ($STATE) — ...`).
- `issue_comment`: same as review_body. If the original was a bot summary (security review, dependency review), one PR comment addressing each concern is enough.
- `check_annotation`: no reply needed. The fix itself plus a reference to it in the follow-up commit message is sufficient. If it was a flake, note that in the final report.
- Keep the body to one or two sentences:
  - ACCEPT + applied: what you changed and where (include commit SHA if pushed).
  - ACCEPT + edited manually: same — what/where.
  - REJECT: the reason, grounded in code or project rule.
  - Already-addressed: point to the commit/line that addressed it.

### 3.6 — Mark resolved

- `review_thread` + ACCEPT (applied, edited, or already-addressed): call `mcp__github_extensions__resolve_review_thread` with the `thread_node_id` (`PRRT_...`).
- `review_thread` + REJECT: leave unresolved so the reviewer can close it after reading your reasoning.
- `review_body` + `CHANGES_REQUESTED`: if every concern is addressed, dismiss via `gh api repos/$OWNER/$REPO/pulls/$PR/reviews/$REVIEW_ID/dismissals --method PUT -f message="Addressed in $SHA; see reply above." -f event=DISMISS`. If any concern is rejected or pending, leave the review as-is and push for re-review.
- `review_body` + `APPROVED` / `COMMENTED`: nothing to resolve — reply + reaction is the full response.
- `issue_comment` and `check_annotation`: no thread state — the reply (or the fix itself) is the full response.

Log one line per item as you go, prefixed with the source:

```
[thread]  [ACCEPT + APPLIED + RESOLVED] path/to/file.py:42 — short summary
[thread]  [ACCEPT + EDITED + RESOLVED]  path/to/file.py:42 — short summary
[thread]  [REJECT + REPLIED]            path/to/file.py:42 — reason
[thread]  [STALE + REPLIED + RESOLVED]  path/to/file.py:42 — addressed in <sha>
[review]  [ACCEPT + EDITED + DISMISSED] CHANGES_REQUESTED by @alice — short summary
[review]  [REJECT + REPLIED]            COMMENTED by @bob — reason
[issue]   [REJECT + REPLIED]            @bot's CI summary — out of scope
[check]   [ACCEPT + EDITED]             "Test Frontend" run #123 — fixed selector
```

## 4. Commit and push code changes

After all items are processed, if any files changed:

- Create a single focused commit like `chore(review): address PR #N feedback`. If the `commit-commands:commit` skill is installed, invoke it; otherwise commit directly.
- Push to the same branch.
- Edit any reply that referenced "will fix in a follow-up commit" with the new commit SHA (`mcp__github_extensions__edit_review_comment`).

## 5. Re-verify nothing is missed

Re-fetch all four sources. Any unresolved thread, un-dismissed `CHANGES_REQUESTED` review, unanswered comment, or still-failing check that isn't intentionally left open gets processed now.

## 6. Final report

```
Address PR Comments — PR #N
────────────────────────────
  Items processed:        N
    - review threads:     N
    - review bodies:      N
    - issue comments:     N
    - check annotations:  N
  Accepted + applied:     N
  Accepted + edited:      N
  Rejected:               N
  Already addressed:      N
  Resolved on GitHub:     N (threads)
  Reviews dismissed:      N
  Left open:              N
────────────────────────────
Left open intentionally:
  - [thread] path/to/file.py:42 — reason
  - [review] CHANGES_REQUESTED by @alice — waiting on clarification
```

## Rules

- Every comment AND every review body gets a reaction (when reactable) and a written reply. Reactions alone are not a response.
- Resolve threads only after the fix is verified (or the comment was already addressed). Never resolve speculatively.
- Dismiss `CHANGES_REQUESTED` reviews only when every concern in the review body is addressed — partial fixes get a reply but not a dismissal.
- Never resolve a thread you rejected — that is the reviewer's call.
- Read the code before replying. The suggestion may already be stale.
- Stay scoped: address the item, nothing else. No opportunistic refactors.
- The repo for every `gh`/`gh api` call is the one resolved in step 1, never a hardcoded value.
