# ship-pr

A Claude Code skill that teaches an agent how to deliver a PR from "pushed" to "merged" by cooperating with two independent services:

- [`review-quill`](https://github.com/krasnoperov/patchrelay/tree/main/packages/review-quill) — reviews the PR and publishes an `APPROVE` or `REQUEST_CHANGES` review.
- [`merge-steward`](https://github.com/krasnoperov/patchrelay/tree/main/packages/merge-steward) — admits approved PRs into a merge queue, validates them against the latest `main` with speculative execution, and fast-forwards `main` to the tested result.

Both tools expose a `pr status --wait` verb with stable exit codes. This skill wires those verbs together so an agent blocks on terminal outcomes (approved / merged / requested-changes / failing-checks / evicted) instead of polling on a fixed timer.

## What the skill does

When invoked, the agent:

1. Self-reviews the diff and the PR description before publishing.
2. Runs `review-quill pr status --wait` and interprets the exit code.
   - On `REQUEST_CHANGES` (exit 2), reads the reviewer's body + inline comments and the failing-check list, fixes the code, pushes, and re-enters the wait.
3. On approval (exit 0), runs `merge-steward pr status --wait`.
   - On `evicted` / `checks_failing` (exit 2), reads the queue incident or failing-check names, fixes the root cause, pushes, and restarts from step 2.
4. Finishes when `merge-steward` reports `merged`.

Throughout, the agent never runs an outer polling loop — the `--wait` flag is the contract.

## Install

```
/plugin marketplace add krasnoperov/patchrelay-agents
/plugin install ship-pr@patchrelay
```

The skill is then invoked as `/ship-pr:ship-pr` in Claude Code.

## Prerequisites

The skill drives CLIs; it does not bundle them. On your machine you need:

- `review-quill` (`npm install -g review-quill`) attached to your repo.
- `merge-steward` (`npm install -g merge-steward`) attached to your repo.
- `gh` for the occasional fallback details link.

Attach each tool to the target repo once:

```
review-quill repo attach owner/repo
merge-steward attach owner/repo
```

If you want these services running autonomously on webhooks (without a human-in-the-loop agent), install [`patchrelay`](https://github.com/krasnoperov/patchrelay) as well — it will drive them through Linear + GitHub webhooks.

## When to use it

Use the skill when you have pushed a non-draft PR and want the agent to own the delivery loop until merge. It is the right choice when:

- You use `review-quill` + `merge-steward` in a supervised mode and want an agent (Claude Code, Cursor, Codex CLI, etc.) to close the loop with you in the room.
- You want to parallelize many PRs across many agents without each one rolling its own polling logic.
- You want deterministic, exit-code-driven flow control rather than LLM-judged "is it done yet?" reasoning.

Use `patchrelay` instead when you want the loop to be fully autonomous, Linear-driven, and restartable across process crashes.

## License

MIT. See [LICENSE](../LICENSE).
