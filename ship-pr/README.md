# ship-pr

A Claude Code skill that turns "I think this PR is ready" into "merged" — in one command.

You finish your work on a branch. You push. You think the PR is good enough to publish. Instead of switching into babysit mode — watching CI, reading review comments, pushing fixups, checking whether the merge queue picked it up — you invoke `ship-pr` once and hand the loop to the agent.

The agent:

1. **Self-checks before publishing.** Reads the diff, looks for obvious issues (stray debug output, stale PR description, missing tests), fixes trivial ones in-place, reconciles the PR description against the actual commits, runs local checks, then marks the PR ready.
2. **Waits for [`review-quill`](https://github.com/krasnoperov/patchrelay/tree/main/packages/review-quill) to post a review** via `review-quill pr status --wait`. On `REQUEST_CHANGES` (exit 2), it reads the reviewer's inline comments and failing-check details out of the JSON, fixes the code literally as requested, pushes, and re-enters the wait.
3. **Waits for [`merge-steward`](https://github.com/krasnoperov/patchrelay/tree/main/packages/merge-steward) to deliver the PR** via `merge-steward pr status --wait`. On `checks_failing` / `evicted` (exit 2), it reads the queue incident or failing-job names, fixes the root cause, pushes, and restarts from the review wait.
4. **Finishes** when `merge-steward` reports `merged` and reports back to you with the merge commit SHA plus what happened along the way.

Throughout, there is no outer polling loop. The agent blocks on each `--wait` call and only wakes on a terminal state. That is the contract.

## Install

```
/plugin marketplace add krasnoperov/patchrelay-agents
/plugin install ship-pr@patchrelay
```

The skill is then invoked as `/ship-pr:ship-pr` in Claude Code. In practice you can also just say "ship this PR" / "ship it" — Claude will recognize the skill by description.

## Prerequisites

The skill drives CLIs; it does not bundle them. On your machine you need:

- [`review-quill`](https://github.com/krasnoperov/patchrelay/tree/main/packages/review-quill) (`npm install -g review-quill`) attached to the target repo.
- [`merge-steward`](https://github.com/krasnoperov/patchrelay/tree/main/packages/merge-steward) (`npm install -g merge-steward`) attached to the target repo.
- `gh` for PR metadata, creating drafts, and pulling CI logs on failure.

Attach each tool to the target repo once:

```
review-quill repo attach owner/repo
merge-steward attach owner/repo
```

If you want these services driven by webhooks instead of your agent, install [`patchrelay`](https://github.com/krasnoperov/patchrelay) as well — it runs the same loop autonomously.

## When to use it

- You have pushed a branch, opened (or are about to open) a PR, and you consider the work complete.
- You want to close the laptop and come back to a merged PR, not babysit CI for 45 minutes.
- You want to parallelize many PRs across many agents without each one rolling its own polling logic.

## When **not** to use it

- The PR needs scope negotiation, not mechanical review fixes. Talk to the human reviewer first.
- The repo is not attached to both `review-quill` and `merge-steward`. The skill will exit with a clear error, but you are better off attaching them first.
- You want the agent to *also* implement the feature. This skill takes over at "I think the work is done" — it is not an implementation loop.

## License

MIT. See [LICENSE](../LICENSE).
