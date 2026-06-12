---
name: rahul-dev-harden-project
description: Run a full hardening audit on a package - security, test coverage, error handling, doc freshness. Triggers on harden, /rahul-dev-harden-project, security audit, run the audit.
---

# /rahul-dev-harden-project — Generic 12-Phase Hardening

## Purpose
Wraps [claude-code-hardening-runbook.md](C:\Users\Me\Desktop\AI\claude-code-hardening-runbook.md) for any project. Project-agnostic — no GTC-specific naming. For GTC Hub packages with collision-safe per-package naming, use `/harden` instead.

## Pre-flight (scoped contradiction scan + reversibility — per §15.9 and §15.10)

Fires at phases that touch deploy target, production data, or main branch. Specifically:
- **Phase 0 (Snapshot)** — creates a pre-hardening tag that will be pushed to remote. Scan fires (touches remote).
- **Phase 8 (Deploy & Rollback)** — documents and simulates rollback, then the gate approves the real deploy plan. Scan + reversibility guard both fire.
- **Phase 9 (Data & Backup)** — tests a real restore against prod-equivalent data. Scan + reversibility guard fire.
- **Phase 12 (Final Verification & Merge)** — merges `hardening` worktree into main, tags, deploys. Scan + reversibility guard fire.
- **All other phases** — local / docs-only. Scan skipped.

For Phase 8, 9, 12:
1. Contradiction scan per §15.9 on deploy target + merge policy.
2. Reversibility guard (§15.10) before the destructive command:
   ```
   ⚠️ <Phase> — irreversible step.
   Action: <merge hardening → main> / <deploy to prod> / <restore backup onto staging>
   Reach: <who / what is affected>
   Revert cost: <what a git revert won't undo>
   Approve? (y/n)
   ```
3. On `n`, stay at the current phase, log "AWAITING RAHUL APPROVAL" to `harden-state.json`.

## State
- **State file:** `_devlog/harden-state.json`
- Schema:
  ```json
  {
    "project": "<name>",
    "started": "YYYY-MM-DD HH:MM",
    "current_phase": 0,
    "last_gate": "none",
    "worktree": "hardening",
    "model_log": [],
    "phase_status": {"0": "pending", "1": "pending", ...}
  }
  ```

## Subcommands

### `/rahul-dev-harden-project start`
1. Check for `_devlog/` (run `/rahul-dev-bootstrap-devlog` first if missing).
2. Check for existing state file — if present with `current_phase` > 0, ask: "Resume at phase N or restart?"
3. Confirm project root with user.
4. Initialise `harden-state.json` with `current_phase: 0`.
5. Create a worktree called `hardening` off main (use `/superpowers:using-git-worktrees` if available).
6. Kick off Phase 0 (snapshot) — see §Phase runner.

### `/rahul-dev-harden-project phase <n>`
Run phase `n`. Refuses to advance if `n > current_phase + 1` (no skipping).

### `/rahul-dev-harden-project status`
Print a table of all 13 phases (0-12) with status, last gate result, worktree, and next recommended action.

### `/rahul-dev-harden-project abort`
Write a final entry to `_devlog/log.md` with reason, set state to `aborted: true`. Does NOT revert the worktree — that's a separate manual decision.

### `/rahul-dev-harden-project 1-day`
Subset runner for time-boxed hardening — only phases 0, 3a, 3b, 6, 8, 9 (see runbook §"IF YOU ONLY HAVE 1 DAY"). Skips the others, state file records the abbreviated run.

## Phase runner (internal)

For each phase number, do:

1. **Read the phase block** from `claude-code-hardening-runbook.md` (specific line ranges below).
2. **Set the model** per the runbook:
   - Opus: phases 3 (security), 5 strategy, 7 (config/secrets), 9 (data/backup), 12 (final)
   - Sonnet: everything else
   - If different from current, tell user: "This phase recommends Opus/Sonnet — `/model` to switch."
3. **Attach the required files** (list from runbook "Files to attach" block).
4. **Paste the phase's "✅ Command to paste"** verbatim. Do not paraphrase.
5. **Run the work** (or walk the user through it for phases requiring live ops).
6. **At the gate**, invoke `/rahul-dev-phase-gate <gate-name>` — which enforces the founder checklist.
7. **On gate pass**, update `harden-state.json`, append to `_devlog/log.md`, update `current-state.md`.
8. **On gate fail**, keep state at current phase. Do not advance.

## Phase index (for quick lookup)
| # | Name                     | Model  | Parallel-safe | Runbook lines |
|---|--------------------------|--------|---------------|---------------|
| 0 | Snapshot & Safety Net    | Sonnet | no            | 29-71         |
| 1 | Inventory                | Sonnet | no            | 73-121        |
| 2 | Baseline Behavior        | Sonnet | no            | 123-168       |
| 3 | Security Audit (a/b/c)   | Opus   | yes (3 terms) | 170-260       |
| 4 | Code Quality & Simplify  | Sonnet | no            | 262-316       |
| 5 | Test Coverage            | Opus→S | no            | 318-378       |
| 6 | Observability            | Sonnet | yes           | 380-447       |
| 7 | Config & Secrets         | Opus   | yes           | 449-504       |
|   | └─ Gate 7 population     | Sonnet | n/a           | sub-agent auto — see `/rahul-dev-phase-gate` skill → `harden config` recipe |
| 8 | Deploy & Rollback        | Sonnet | yes           | 506-564       |
| 9 | Data & Backup            | Opus   | no            | 566-614       |
|10 | Cost & Dependency        | Sonnet | yes (RO)      | 616-669       |
|11 | Documentation            | Sonnet | no            | 671-722       |
|12 | Final Verification       | Opus   | no            | 724-787       |

## Multi-terminal phases (3, 6-8, 10)
When the runbook says "parallel safe with siblings", this skill:
1. Prints the sub-worktree commands the founder should run in separate terminals.
2. Writes per-agent PARALLEL CONTEXT blocks into `_devlog/harden-terminal-<letter>.md` so each terminal has its scope pinned.
3. On merge back, walks through the runbook's merge sequence one-by-one, running baseline demo between each merge.

## Output artefacts (per runbook)
- `/docs/audit/inventory.md` (phase 1)
- `/docs/audit/baseline.md` (phase 2)
- `/docs/audit/security-{secrets,deps,owasp}.md` (phase 3)
- `/docs/audit/quality.md` (phase 4)
- `/docs/audit/test-plan.md` (phase 5)
- `/docs/runbook/configuration.md` (phase 7)
- `/docs/runbook/deploy.md` (phase 8)
- `/docs/runbook/data.md` (phase 9)
- `/docs/runbook/costs.md` (phase 10)
- `/docs/README.md` (phase 11)

## What NOT to do
- Do NOT skip phases without `--force-skip` explicitly. The state machine enforces order.
- Do NOT advance past a failed gate — that's exactly what the gate is for.
- Do NOT mix with `/harden` (GTC per-package) in the same project. Pick one.
- Do NOT auto-commit the hardening branch to main. `/rahul-dev-phase-gate final` → then `/superpowers:finishing-a-development-branch`.

## Handoff during hardening
Hardening is long. Between phases, `/handoff` + `/clear` + `/resume` is strongly recommended — especially before phases 3, 5, 9, 12 (Opus-heavy phases where fresh context saves tokens).
