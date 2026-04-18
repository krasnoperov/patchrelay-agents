---
name: ship-pr
description: "Single-command handoff: user says 'ship it', agent owns the PR from pre-flight self-check through merge. Marks the PR ready, blocks on review-quill and merge-steward via --wait gates, reacts to requested changes and failing checks by fixing code and pushing, re-enters the wait, and finishes when merge-steward reports merged. Use when the user has completed their work on a branch and wants an agent to deliver the PR."
---

# ship-pr

## The contract

When the user invokes this skill, they are handing you the PR. They have done the work, they think it is ready, and they expect **one command** to get it merged. From this point you own the loop. Do not ask the user what to watch for, do not spin up a cron timer, do not hand back partway.

You are delivering the PR into a stack with two GitHub-native services:

- `review-quill` reviews every merge-ready head and publishes a normal `APPROVE` / `REQUEST_CHANGES` review.
- `merge-steward` admits approved, green PRs into a merge queue, speculatively integrates each one on top of the latest `main`, validates, and fast-forwards `main` to the tested result.

Both tools expose `pr status --wait` with stable exit codes. That is the contract you block on. You only wake when there is real work.

## Phase 1 — pre-flight self-check

The user handed you a PR. First make sure the PR you are about to publish is worth publishing. Most requested-changes loops are avoidable if you catch trivial issues here instead of making a reviewer catch them.

1. **Find the PR.** From the branch's checkout, `gh pr view --json number,isDraft,title,body,headRefName`. If no PR exists yet, create one as a draft (`gh pr create --draft`) with a title + body that reflect the actual commits. If the branch is not pushed, push it first.
2. **Self-review the diff.** Read `git diff origin/main...HEAD` (or the repo's base branch). Look for:
   - stray debug output, commented-out code, leftover TODO markers you added mid-session
   - imports or symbols you no longer use
   - tests you added that would obviously fail the review rubric (missing assertions, flaky setup)
   - schema/API changes not reflected in the changelog or docs the repo expects
3. **Fix trivial issues inline.** Do not file a new PR. Amend or add a commit on the same branch.
4. **Reconcile the PR description against the final diff.** If the description has stale bullets from an earlier iteration, update it. Never include an "I ran these commands" / "Verification" section — CI owns pass/fail signal.
5. **Run local checks on touched packages** if the repo has them (`npm run ci`, `cargo test`, `pytest` — whatever the repo uses). Green locally is a cheap sanity gate.
6. **Mark the PR ready** (`gh pr ready <num>`) once the above is clean. This is the moment review-quill and merge-steward start caring.

Push any fixups before moving on. A clean head SHA gets reviewed once; a dirty one costs you a full review cycle.

## Phase 2 — wait for review-quill

From the PR's checkout (cwd resolves repo+PR automatically):

```bash
review-quill pr status --wait --timeout 1800 --poll 10 --json
```

Interpret the exit code:

- **`0`** (`approved` or `skipped`) — review-quill is done. Go to Phase 3.
- **`2`** (`declined`, `errored`, `cancelled`) — read the `failureDetails` block of the JSON:
  - `reviewRequest.body` and `reviewRequest.inlineComments[]` (path, line, body, author) when a reviewer requested changes.
  - `failedChecks[]` and `pendingChecks[]` (`name`, `status`, `conclusion`, `detailsUrl`) when CI blocked the review.

  Triage before you fix — see [Triage before you fix](#triage-before-you-fix) below. Restore review-readiness, not just the cited line. Then push a new commit and re-enter Phase 2. The latest head SHA supersedes prior attempts automatically.
- **`4`** — wait timed out. Before extending, run `review-quill attempts --json` to see if a review is actually in flight. If yes, extend `--timeout` and rerun. If not, something is wrong — surface it.
- **`1`** — usage or config error. Fix the invocation (unattached repo, bad flags). Do not retry blindly.

## Phase 3 — wait for merge-steward

review-quill's approval fires a webhook that the steward already listens to. You do not enqueue manually. Then:

```bash
merge-steward pr status --wait --timeout 3600 --poll 15 --json
```

Interpret the exit code:

- **`0`** (`merged` or `merged_outside_queue`) — shipped. You are done.
- **`2`** — terminal failure. Read `kind`:
  - `changes_requested` — a reviewer requested changes *after* approval was recorded. Restart from Phase 2.
  - `checks_failing` — required CI checks failed on the PR head or the speculative integrated SHA. The JSON's `github.checks[]` lists failing names and `detailsUrl`. Pull the actual failure with `gh run view <run-id> --log-failed`, triage the root cause (see [Triage before you fix](#triage-before-you-fix)), fix it plus any adjacent jobs at risk of the same cause, push, restart from Phase 2.
  - `evicted` / `dequeued` — the queue removed the entry. Run `merge-steward queue show --pr <num> --json` for the event / incident trail. Typical causes: stale speculative branch, conflicting `main` advance (rebase needed), upstream incident. Fix the root cause, push, restart from Phase 2.
  - `closed` — the PR was closed without merging. Stop and ask the user.
- **`4`** — timed out. Run `merge-steward queue show --pr <num>`. If healthy and slow, extend the timeout. If an incident is pending, treat as exit `2`.
- **`1`** — usage or config. Fix.

## Triage before you fix

Both exit-2 paths are easy to get wrong in opposite directions. Avoid both.

### Do not blindly patch the cited symptom

A reviewer's comment on line 42 is **evidence** the branch is not review-ready, not an isolated chore. A failing CI job is **one instance** of a root cause that may affect other jobs in the suite. Before pushing a fix:

1. Read **all** inline review comments for the PR, not just the triggering one. `gh api repos/<owner>/<repo>/pulls/<num>/comments?per_page=100` is the quickest way. The goal is to see the full surface of what the reviewer thinks is unready.
2. Read the full diff the reviewer saw: `review-quill diff` when available, or `git diff origin/main...HEAD` as a fallback. Do not infer from the comment alone.
3. **Infer the underlying concern or invariant.** Is this comment about a missing null check, or about a class of missing null checks the branch has elsewhere? Is this failing test about one bad assertion, or about a behavior change the test is correctly catching?
4. **Inspect adjacent code paths that could fail for the same reason.** Fix them too. A PR that patches the one cited line and ignores the three other sites with the same bug will come back with another round of requested changes.
5. For a CI failure, read the failing log (`gh run view <run-id> --log-failed`), identify the root cause, and check whether that same cause would fail other jobs in the suite. Fix the root cause, not the first visible symptom.

### Do not silently expand into a broader rewrite

You are restoring review-readiness, not refactoring the codebase. The reviewer did not hand you a license to restructure the module.

- Stay inside the concrete concern raised. If a comment asks for a null check, add the null check and the adjacent ones in the same flow — do not extract helpers, rename types, or reshape surrounding code.
- Do not widen into unrelated polish or follow-up cleanup. If you notice a broader inconsistency, **mention it in your final report to the user** as follow-up context; do not fix it in this PR.
- Do not change workflow files, dependency installation, CI config, or unrelated tests unless the failing logs clearly point there.
- Do not change test expectations unless the test is genuinely wrong.

### When triage says "stop"

If the underlying concern is not a mechanical fix — the reviewer is asking for a different approach, the failing check is infrastructure-level, the merge conflict is a semantic contradiction between two intents — stop and surface it to the user instead of guessing. Two iteration cycles on the same failure with no clear diagnosis is the other hard stop. See [Failures to escalate](#failures-to-escalate) below.

## The iteration loop

Every time you push a fix, restart from **Phase 2**. review-quill must re-approve the new head before merge-steward admits it again. A clean sequence looks like:

1. Phase 1 (self-check, mark ready)
2. Phase 2 → exit 2 (reviewer asked for a null check)
3. Fix, push
4. Phase 2 → exit 0
5. Phase 3 → exit 2 (flaky test on speculative SHA)
6. Fix (or rerun the job with `gh run rerun --failed <run-id>`), push
7. Phase 2 → exit 0 (re-review because the head changed)
8. Phase 3 → exit 0 (merged)

If the same fix gets bounced twice with unclear signal, surface to the user instead of guessing a third time. Two cycles is enough to rule out simple bugs; a third indicates something the agent cannot see from the diff.

## Failures you can fix on your own

These are the patterns where you should not escalate. Fix and keep going:

- Reviewer asked for a rename, a missing test, a missing null check, extracted helper. Apply the change — and check whether the same concern exists in adjacent code (see Triage).
- Lint / typecheck red on the speculative SHA, green locally. Usually version drift or interaction with a commit that landed on `main` since you branched. Rebase, rerun local checks, push.
- Flaky test. If `detailsUrl` shows a failure clearly unrelated to the diff that reproduces intermittently, the steward's `flakyRetries` may already handle it; otherwise `gh run rerun --failed <run-id>`.
- Merge conflict surfaced by speculative invalidation. Rebase on `main`, resolve, push.
- Stale PR description. Update it in Phase 1 proactively; if you missed it and the reviewer called it out, fix and push.

## Failures to escalate

These are the patterns where you should stop and ask the user:

- `kind: closed` — someone closed the PR out from under you.
- Reviewer requested a scope change (new feature, different approach, rewrite). Not a mechanical fix.
- Two full iteration cycles on the same failure with no clear diagnosis.
- The repo is not attached to both `review-quill` and `merge-steward` (exit `1` with "not attached"). The user needs to attach it first.

## Decision rules

- **No outer loop.** The `--wait` form is the contract. Do not wrap these commands in a cron, `/loop`, or busy-poll.
- **No polling without `--wait`.** If you are running `pr status` in a `while` loop, you are doing it wrong.
- **Always read `failureDetails` before pushing a fix.** Speculative "maybe this helps" commits cost another full review + queue cycle.
- **Never `gh pr merge` a PR attached to `merge-steward`.** The steward is the one that writes to `main`.
- **Stay on the branch.** Do not cut a new PR for fixups; push to the existing branch.

## Definition of done

- `review-quill pr status --wait` exited `0` on the final head SHA.
- `merge-steward pr status --wait` exited `0` with `kind: merged` or `merged_outside_queue`.
- No unresolved queue incidents for this PR.

Report back to the user with the merge commit SHA and any notable events that happened during the loop (number of iterations, what got fixed).
