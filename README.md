# patchrelay-agents

Agent skills for pairing Claude (and any other agentic development tool) with the PatchRelay service family.

This repository is a Claude Code **plugin marketplace**. It ships skills; it does not ship the services themselves. Install a skill from here, install the services you need from their own packages, and your agent gets a fully automated PR delivery loop.

## What you actually get

Two services that together give your agents two superpowers:

### [`review-quill`](https://github.com/krasnoperov/patchrelay/tree/main/packages/review-quill) — GitHub-native PR review bot

- Watches every PR on repos you attach. Sees a new head, reviews it, publishes an ordinary GitHub `APPROVE` or `REQUEST_CHANGES` review with inline comments.
- Reviews from a **real local checkout of the exact head SHA**, not the GitHub files API. Reviewers see the same state your tests saw.
- Ships with a `--wait-for-green-checks` gate if you want, but the default is to review as soon as the head updates — with decent dev tests your PRs are usually green and you do not need the gate.
- Zero coupling to PatchRelay. Zero coupling to Claude. All signaling goes through normal GitHub reviews, so every agent, IDE, or human in your org can read the output without installing anything else.

### [`merge-steward`](https://github.com/krasnoperov/patchrelay/tree/main/packages/merge-steward) — GitHub-native serial merge queue

- Admits approved, green PRs into a queue and **speculatively integrates each one on top of the current `main`**. CI runs on the integrated SHA, not just the PR head.
- Fast-forwards `main` to the tested result, so what lands is exactly what was validated.
- Evicts with a durable incident record and a GitHub check run on failure. An agent (or a human) sees the incident, fixes the cause, and the PR gets re-admitted.
- Zero coupling to PatchRelay. Zero coupling to Claude. Communicates through GitHub: labels, checks, reviews, pushes.

### Why this combination is transformational

- **PRs are delivered fully tested against the latest `main`.** No more "CI was green yesterday, breaks on merge today." The queue catches the integration bug before `main` ever sees it.
- **Most failures have mechanical fixes.** Address the reviewer's inline comment. Rerun the flaky job. Rebase on `main`. Resolve a conflict the steward surfaced during speculation. None of these require human judgment in the usual case — they need an agent with access to the diff.
- **No prerequisites beyond GitHub.** No Linear, no self-hosted control plane, no proprietary SDK. A GitHub App, a webhook, and `npm install -g`.

## Two ways to drive them

### 1. Fully autonomous — [`patchrelay`](https://github.com/krasnoperov/patchrelay)

PatchRelay receives GitHub webhooks (review posted, CI failed, queue evicted, merge landed) and automatically starts Codex `app-server` sessions to repair the PR. No human in the room. This is the right choice when:

- You want to hand off a backlog and come back to merged PRs.
- You have a Linear workspace that can delegate issues to the harness.
- You want restartable, durable loops across crashes.

Setup lives in the main repo's README.

### 2. Supervised — install a skill here and drive from your agent

When you are already sitting in Claude Code (or Cursor, or Codex CLI, or anything that reads Claude Code skills / plugins), you do not need PatchRelay's full harness. You just need your agent to know how to **cooperate with** `review-quill` and `merge-steward` — wait for approvals, read the failure reason, fix it, push, and repeat.

That is what this repo ships.

## Skills in this marketplace

| Skill | Purpose |
|-|-|
| [`ship-pr`](./ship-pr/) | Shepherd a non-draft PR from "pushed" to "merged". Blocks on `review-quill pr status --wait` and `merge-steward pr status --wait`, interprets exit codes, fixes requested changes and failing checks, re-enters the wait. No polling loop. |

More skills will land here as patterns harden.

## Install

Add this marketplace once:

```
/plugin marketplace add krasnoperov/patchrelay-agents
```

Install a skill:

```
/plugin install ship-pr@patchrelay
```

Plugins are cached under `~/.claude/plugins/cache/` and survive across sessions. Run `/plugin uninstall ship-pr@patchrelay` to remove one.

For a local-development install (e.g. you cloned this repo and want to iterate on the skill):

```
/plugin marketplace add /path/to/patchrelay-agents
/plugin install ship-pr@patchrelay
```

## Prerequisites

The skills drive CLIs; they do not bundle them. Install them from their own packages once per machine:

```bash
npm install -g review-quill
npm install -g merge-steward
```

Attach each tool to the target repo:

```bash
review-quill repo attach owner/repo
merge-steward attach owner/repo
```

Full setup instructions (GitHub App, webhook, systemd, TLS ingress) live in each service's README:

- [review-quill README](https://github.com/krasnoperov/patchrelay/tree/main/packages/review-quill)
- [merge-steward README](https://github.com/krasnoperov/patchrelay/tree/main/packages/merge-steward)
- [Main PatchRelay repo](https://github.com/krasnoperov/patchrelay) — for the full three-service story

## Design principle: stable contracts, not tool-specific magic

The skills in this marketplace are intentionally thin. They compose CLI verbs (`pr status --wait`, `queue show`, `attempts`) with stable exit codes. That means:

- The skill is not locked to Claude. The same exit-code contract is what you would write in a bash pipeline or a GitHub Actions job.
- The services can evolve without breaking the skill, as long as the exit-code contract is preserved.
- You can swap the agent (Claude Code, Cursor, Codex, a custom LangGraph runtime) without changing the services or the skill logic.

That is the part worth stealing even if you do not use the skills here: design your internal agent tools around a small number of blocking-gate verbs with stable exit codes, and the orchestration collapses to almost nothing.

## License

MIT. See [LICENSE](./LICENSE).
