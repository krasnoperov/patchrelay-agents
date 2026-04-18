---
name: ship-pr
description: "Shepherd a non-draft PR through review-quill approval and merge-steward's merge queue with blocking --wait gates. React to requested changes, failing required checks, and queue incidents instead of polling on a fixed cron. Use when an agent has pushed a PR and needs to watch it to merge, addressing any issues that arise."
---

# ship-pr

## Why this skill exists

You (the agent) just pushed a non-draft PR on a repo that is attached to two independent GitHub-native services:

- `review-quill` reviews every merge-ready head it sees and publishes an ordinary GitHub `APPROVE` or `REQUEST_CHANGES` review.
- `merge-steward` admits approved, green PRs into a serial merge queue, speculates on top of the current `main`, waits for CI on the integrated SHA, and fast-forwards `main` to the tested result.

Together they give you two superpowers:

1. **PRs are delivered fully tested against the latest `main`.** No more "passed CI yesterday, breaks on merge today" — the queue re-validates against real live `main` before it lands.
2. **Most failures have mechanical fixes.** Fix the reviewer's inline comment; rerun the failed check; resolve the conflict surfaced by speculative validation; then push again. An agent can do all of that on its own.

This skill is the glue that makes an agent do it efficiently. Instead of rolling a `while true; sleep 60; gh pr view` loop, the skill uses each tool's `pr status --wait` verb, which blocks until a terminal state and returns a stable exit code. You only wake up when there is real work.

## The blocking contract

Both tools expose the same verb with the same exit codes:

| Tool | Command | Terminal success | Terminal failure | In flight | Wait timeout |
|-|-|-|-|-|-|
| review-quill | `review-quill pr status` | 0 | 2 | 3 | 4 |
| merge-steward | `merge-steward pr status` | 0 | 2 | 3 | 4 |

Both accept `--repo <id> --pr <num>` or resolve both from `origin` + the current branch's PR when run inside a git checkout. Both accept `--wait --timeout <s> --poll <s>` to block until terminal. Both accept `--json` for machine-readable output. Exit code `1` means usage / config error — do not retry it.

## Workflow

### Step 0 — ship a clean PR in the first place

Less review churn = fewer round trips. Before taking the PR out of draft:

- Self-review the diff for dead code, stray debug output, broken imports, missed tests.
- Make sure the PR description matches the final diff. No stale bullets from earlier iterations.
- Do **not** include an "I ran these commands" / "Verification" section in the PR body. CI owns pass/fail signal.
- Run the repo's local checks on the touched package(s) (`npm run ci` or targeted equivalents). Green locally is a cheap sanity gate.
- Never squash-merge on repos that use conventional commits for release planning.

### Step 1 — wait for review-quill to settle

From the PR's checkout (cwd resolves repo+PR automatically), or with explicit flags:

```bash
review-quill pr status --wait --timeout 1800 --poll 10 --json
```

Interpret the exit code:

- **`0`** (`approved` or `skipped`) — review-quill is done. Go to Step 2.
- **`2`** (`declined`, `errored`, `cancelled`) — read the `failureDetails` block of the JSON. It contains:
  - `reviewRequest.body` and `reviewRequest.inlineComments[]` (path, line, body, author) when a reviewer requested changes.
  - `failedChecks[]` and `pendingChecks[]` with `name`, `status`, `conclusion`, `detailsUrl`.

  Address the feedback in code, push a new commit, then re-enter Step 1. The latest head SHA supersedes prior attempts automatically.
- **`4`** — wait timed out. Either extend the timeout and rerun, or inspect `review-quill attempts` / `review-quill transcript` to see why the reviewer is slow before deciding.

### Step 2 — wait for merge-steward to settle

`review-quill`'s approval triggers `merge-steward`'s reconciler on its own (both listen to the same GitHub events). You do not need to manually enqueue. Then:

```bash
merge-steward pr status --wait --timeout 3600 --poll 15 --json
```

Interpret the exit code:

- **`0`** (`merged` or `merged_outside_queue`) — shipped. Verify `main` CI green and move on.
- **`2`** — terminal failure. Read `kind`:
  - `changes_requested` — a reviewer requested changes *after* the approval was recorded. Fix, push, restart from Step 1.
  - `checks_failing` — required CI checks failed on the speculative head or the PR head. The JSON `github.checks[]` lists names and `detailsUrl`; fix the failing job, push, restart from Step 1.
  - `evicted` / `dequeued` — the queue removed the entry (stale speculative branch, conflicting `main` advance, upstream incident, or policy violation). Run `merge-steward queue show --pr <num> --json` to see the event/incident trail, fix the root cause, push, restart from Step 1.
  - `closed` — the PR was closed without merging. Stop and ask the user.
- **`4`** — timed out. Run `merge-steward queue show --pr <num>` to see current queue events. If the queue is healthy and just slow, extend the timeout; if there is an incident, treat as exit `2`.

### Reacting to failures — adjunct commands

- `review-quill attempts --repo <id> --pr <num> --json` — review history for the PR.
- `review-quill transcript --repo <id> --pr <num> --json` — full reviewer thread for the latest attempt.
- `merge-steward queue show --pr <num> --json` — queue events + incidents for one PR.
- `merge-steward queue status --repo <id> --json` — full queue summary when the PR is blocked behind another entry.
- `gh pr checks <num>` and `gh run view <run-id>` — raw CI details when a check's `detailsUrl` points at GitHub Actions.

## Examples of failures you can fix on your own

The whole point of this loop is that most PR failures do not need a human. Some patterns worth knowing:

- **Reviewer asked for a rename / missing test / missing null check.** Apply the change the inline comment literally described, push.
- **Lint/typecheck failure on the speculative SHA that was green locally.** Usually a transient version drift or an interaction with a commit that landed on `main` since you branched. Rebase, rerun local checks, push.
- **Flaky test failure.** Look at the failure in `detailsUrl`. If it is clearly unrelated to the diff and reproduces intermittently, the steward's `flakyRetries` may already handle it; otherwise rerun the failing job with `gh run rerun --failed <run-id>`.
- **Merge conflict after speculative invalidation.** The steward evicts with an incident; rebase the branch onto `main`, resolve, push.
- **Stale PR description / unmerged docs bullet.** Not a failure the tools detect, but worth proactively fixing before Step 1 to avoid requested changes.

## Decision rules

- Do **not** busy-loop on `pr status` without `--wait`. The `--wait` form is the contract.
- Do **not** add an outer cron / `/loop` around these commands. The CLIs already poll internally against local state (queue DB, review DB) with a GitHub fallback.
- On exit `2` from either tool, always inspect the structured failure reason before pushing a fix. Speculative "maybe this helps" commits waste another full review + queue cycle.
- On exit `1`, fix the invocation (bad flags, unattached repo, missing config). Never retry blindly.
- After pushing a fix, restart from **Step 1**. `review-quill` must re-approve the new head before `merge-steward` admits it again.
- Never bypass the queue with `gh pr merge` on repos attached to `merge-steward`. The steward is the one that writes to `main`.

## Definition of done

- `review-quill pr status --wait` exited `0` on the final head SHA.
- `merge-steward pr status --wait` exited `0` with `kind: merged` or `merged_outside_queue`.
- `main` CI is green after the merge.
- No unresolved queue incidents for this PR in `merge-steward queue show`.
