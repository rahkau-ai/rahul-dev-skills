---
name: rahul-magic-dev
description: Multi-scope outer orchestrator. Splits a plan by scope, spawns one autopilot session per scope in its own worktree + integration branch. Enforces singleton-branch invariant. Triggers on /rahul-magic-dev, magic-dev, batch-multi-scope, multi-package autopilot.
model: sonnet
---

# /rahul-magic-dev — Multi-scope batch orchestrator

## Invocation

```
/rahul-magic-dev [--plan <path>] [--issues N,M,K] [--dry-run] [--batch-id <id>] [--isolated]
```

- `--plan <path>` — read a plan file (cap 300 lines) and split by scope
- `--issues N,M,K` — fetch GitHub issues and concatenate their bodies as the working spec
- bare invocation — scan the current conversation for the most recent plan / scope-spanning discussion
- `--dry-run` — perform the split + branch + interference check, print the report, do NOT spawn anything
- `--batch-id <id>` — use this exact BATCH_ID instead of a fresh timestamp (the dev cycle passes its CYCLE_ID so the spawn receipt is found deterministically). Default: own `date +%Y-%m-%dT%H-%M-%S`.
- `--isolated` — throwaway / proof runs (the dev cycle's live proof passes this): each scope gets a BATCH_ID-namespaced branch `integration/batch-<scope>-proof-<BATCH_ID>` that is never looked up or joined, so two same-day same-scope runs can't collide (Fix J). Normal runs keep v6 singleton-join and refuse to join any `-proof-` branch. See PROCEDURE.md §5.

The repo root is derived from CWD (`git rev-parse --show-toplevel`), never hardcoded — so the dev cycle can run magic-dev inside a clone (Fix F).

On exit, magic-dev writes a one-shot **spawn receipt** at `_devlog/magic-dev/<BATCH_ID>.json`
(scope → worktree / integration_branch / issue / status). This is NOT a state file — it is
written exactly once and never mutated or polled by magic-dev; it lets the dev cycle locate
the spawned work deterministically instead of globbing worktrees. See PROCEDURE.md §9.

## What it does (one paragraph)

Outer orchestrator. Splits the input plan by v6 scope, finds or creates one integration branch
PER scope (singleton across all sessions), runs an interference check against any sibling
worktrees already on that branch, then spawns ONE `/rahul-dev-autopilot` session per scope
via `mcp__ccd_session__spawn_task` — each in its own worktree. Magic-dev's own session exits
after spawning. No magic-dev state file. The spawn_task chips + per-worktree autopilot state
files are the only persisted state. Spawned autopilot sessions run independently to
`awaiting_human`.

## Composability rule (DO NOT FORGET)

Magic-dev = OUTER orchestrator (split + spawn + exit). Autopilot = INNER engine (build +
cold-review loop per scope). Magic-dev NEVER reimplements autopilot's fix loop, scope
detection internals, or cold-review logic — it invokes autopilot N times in N worktrees.

## Before any tool call

Read `PROCEDURE.md` in this directory. SKILL.md is a stub; the full 10-section procedure
(inputs, scope detection, per-scope spec synthesis, singleton-branch resolution, interference
check, GitHub issue creation, spawn, exit summary, hard rules) lives there.
