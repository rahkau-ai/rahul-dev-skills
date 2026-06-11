# rahul-dev — Claude Code / Cowork Plugin

Rahul's core dev-cycle skills, packaged as a public Claude Code plugin for cloud and Cowork sessions.

## Install (cloud / phone sessions)

```
/plugin install rahul-dev@rahul-dev-skills
```

## Skills included

| Skill | Trigger | What it does |
|-------|---------|-------------|
| `/grill-me-with-facts` | `grill me`, `check for contradictions` | Scans the repo for factual contradictions across all sources (git, memory, CLAUDE.md chain, GitHub, code) and resolves them interactively. |
| `/rahul-magic-dev` | `magic-dev`, `multi-scope` | Outer orchestrator: splits a plan by scope and spawns one autopilot session per scope in its own worktree + integration branch. |
| `/rahul-dev-autopilot` | `autopilot`, `walk-away` | Walk-away build loop: issue → worktree → plan → build → cold-review → fix until clean → PR. |
| `/rahul-dev-cold-review` | `cold review`, `review the PR` | Fresh-session unbiased code review: fetches the PR diff cold and posts findings back to the PR. |
| `/rahul-dev-harden-project` | `harden`, `security audit` | 12-phase hardening audit covering security, tests, config, observability, docs, and deploy. |

### Bundled dependencies (invoked automatically by the skills above)

- `/rahul-dev-work` — single-package at-desk dev pipeline (autopilot dependency)
- `/rahul-dev-phase-gate` — runbook gate enforcer (harden dependency)
- `/rahul-dev-bootstrap-devlog` — `_devlog/` scaffold setup (harden dependency)

## Cloud notes

- `live-state.sh` gate is unavailable on cloud VMs — skills skip it gracefully and continue.
- Git worktrees (`rahul-magic-dev`, `rahul-dev-autopilot`) require the hub repo to be cloned on the cloud VM.
- `/rahul-dev-cold-review` requires the `superpowers` plugin for its sub-review step.
- Hardening runbook is bundled in `skills/rahul-dev-harden-project/claude-code-hardening-runbook.md`.

## Auto-sync

On laptop sessions, the `sync-dev-skills.sh` Stop hook pushes changes from `~/.claude/skills/` to this repo automatically at session end.
