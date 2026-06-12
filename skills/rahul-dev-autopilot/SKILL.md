---
name: rahul-dev-autopilot
description: Walk-away multi-package build loop - build, cold-review, fix until clean, PR + notification. Never auto-merges (auto-merge is cycle-only, for non-preview scopes). Triggers on /rahul-dev-autopilot, autopilot, walk-away mode.
model: opus
---

# /rahul-dev-autopilot — Walk-away dev loop (v6)

## Purpose

You type ONE command, walk away, and come back to a clean integration PR ready for your
final approval. The autopilot:

1. Runs `/rahul-dev-work` (issue → worktree → plan → build → push to integration)
2. Spawns a **fresh sub-agent** (no author bias) to run `/rahul-dev-cold-review`
3. Parses blockers + should-fix from the review
4. Loops back to `/rahul-dev-work` with those findings as the new spec
5. Repeats until cold-review returns clean (no blockers, no should-fix)
6. Pushes a narrative PR body, sends a PushNotification, and waits for human merge

**Hard rule:** the loop NEVER squash-merges integration → main. Final merge stays manual.

## v6 changes (what's different from v2)

- **No lock file.** A second run is never blocked — same scope just gets a one-line warning.
- **Single-tier branches.** `integration/batch-<scope>-YYYY-MM-DD`, one per scope. No per-issue
  integration branches.
- **No handoff markdown files.** The run narrative is pushed into the GitHub PR body via
  `gh pr edit`. The only persisted artifact is the compact `<RUN_ID>.json` state file.
- **Pre-work triage** (Step 0) shows what's already open for the scope before creating a worktree.
- **`business-os`** is a recognized scope.

## When to use this vs `/rahul-dev-work`

| Mode | When | Command |
|---|---|---|
| Walk-away (this) | Well-specified work, you'll be away 30+ min | `/rahul-dev-autopilot <description>` |
| Walk-away from a plan file | Approved plan exists with a "Files to modify" section | `/rahul-dev-autopilot --plan <path>` |
| At-the-desk | You want to drive each step yourself, or the spec is fuzzy | `/rahul-dev-work <description>` |

If unsure, prefer `/rahul-dev-work`. Autopilot is for work where the spec is clear enough
that you trust the loop without supervision.

**Multi-package work:** if the description or plan file touches more than one package,
autopilot records each package in the run's `packages` array (array order = build order,
`core` first). The tick processes them sequentially, each with its own
`integration/batch-<scope>-YYYY-MM-DD` branch and PR.

---

## ⚠️ READ FIRST — How this skill behaves with plan mode

**This skill is an ORCHESTRATOR, not a builder.** Its only job in the parent conversation
is to bootstrap state and hand off to `/loop`. The actual building, reviewing, and fixing
happens in **fresh sub-agents** spawned by the tick skill — that's how cold review stays unbiased.

The global rule "Every new task must start in planning mode" WILL fire when you invoke this skill.

| Phase | What happens | What you (the agent) do |
|---|---|---|
| Plan mode fires automatically | Explore + write plan | Use this for the clarifying-questions gate (Step 2). The plan content becomes the autopilot's `description`. |
| `ExitPlanMode` is approved | System says "Auto Mode Active. Execute immediately." | **DO NOT** start building. "Execute" here means **run Step 0–3 + invoke `/loop` + exit**. |
| `/loop` is invoked | The tick skill spawns fresh sub-agents | This conversation's job is **DONE**. Stop responding to anything except confirmations. |

**The single most important rule:** after Step 4 (`/loop` invocation), you do NOT edit
files, run tests, commit, push, or invoke other skills in this conversation. All execution
is delegated to sub-agents.

---

## Step 0 — Pre-work triage (before creating anything)

Before any worktree exists, show what's already open for the scope so you don't duplicate work.

1. **Detect scope** from the description / plan file / CWD keywords (see Step 1's table).
2. **List open integration PRs for the scope:**
   ```bash
   gh pr list --repo rahkau-ai/gtc-hub --search "head:integration/batch-<scope>" \
     --json number,title,state
   ```
3. **List scope worktrees:** `git worktree list --porcelain` — keep entries whose path or
   branch name contains `<scope>` (approximate match).
4. For each existing scope worktree: `gh pr list --repo rahkau-ai/gtc-hub --head <branch> --json state`.
5. **Classify each:**
   - **MERGE-READY** — open PR with green checks → show it, ask "Merge this first? (y/N)".
   - **IN-PROGRESS** — dirty or open work → check file overlap (next point).
   - **MERGED-UNCLEAN** — branch merged, worktree still on disk → flag for cleanup via `/rahul-dev-done`.
6. **"Related" is DETERMINISTIC.** A new feature is related to an existing worktree ONLY IF
   the plan's identified file paths intersect that worktree's `.spec-scope.json` `in_scope`
   array. If `.spec-scope.json` is absent → treat as unrelated. On overlap, ask:
   "Your new feature touches the same files as [issue-N: description]. Add there? (y/N)".
7. **Default:** create a new worktree off `integration/batch-<scope>-*`.

Print the triage as a short table, then proceed once the user has answered any prompts.

---

## Step 1 — Detect repo + scope(s)

### 1A — Detached-HEAD guard

```bash
git symbolic-ref -q HEAD >/dev/null || {
  echo "⚠️ Main repo is in detached HEAD. Run: git checkout main"
  echo "   Autopilot needs a normal branch checkout to operate. Halting."
  exit 1
}
```

### 1B — Repo routing

```bash
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
```

| Description / CWD signals | Repo | Required CWD |
|---|---|---|
| topicboard, topic board, expert portal, MembersDB, Control Room, Kanban Intake | **gtc-topicboard** (nested at `packages/topicboard/`) | `cd packages/topicboard` |
| Anything else | gtc-hub (this repo) | repo root |

**topicboard hard-stop redirect:** if the description contains topicboard keywords AND CWD
is NOT inside `packages/topicboard/`, halt with:

```
TopicBoard work uses the gtc-topicboard repo, not gtc-hub.
  cd ~/Desktop/GTC/hub/packages/topicboard
  /rahul-dev-autopilot <same description>
```

### 1C — Detect scope(s)

Scan the description / plan file's "Files to modify" section / CWD for ALL keyword matches.
Each match adds a scope to the run's `packages` array.

| Keywords | Scope |
|---|---|
| extension, linkedin extension, chrome extension, popup, sidebar | `extension` |
| website, homepage, landing, hero, nav, footer, pricing page, blog | `website` |
| dashboard, linkedin dashboard, leads, port 5000, analytics | `dashboard` |
| email, nurture, apps script, expert network, onboarding email | `email` |
| brand, design tokens, starter kit, html template, pdf | `brand` |
| video, remotion, animation, rendering | `video` |
| migrations, supabase, schema, sql, database | `core` |
| business-os, health check, audit page, ai health, "fix it" | `business-os` |
| topicboard, topic board portal | `topicboard` (HARD STOP redirect, see 1B) |
| skill, claude skill, ~/.claude/skills, hooks, CLAUDE.md | `global` |

**Scope list (all recognized):** `extension` · `dashboard` · `website` · `email` ·
`topicboard` · `brand` · `video` · `core` · `business-os` · `global`.

**Ordering:** if `core` is among the detected scopes, place it first in the `packages`
array (migrations ship before code that reads them). Otherwise alphabetical.

**Special scope `global`** (skills / `~/.claude/CLAUDE.md` / hooks): no worktree, no branch,
no PR. Edits land directly in `~/.claude/...` and are logged to
`~/.claude/_devlog/global-changes.md`. The tick handles this scope without review.

### 1D — Validate

| Result | Action |
|---|---|
| Empty | HALT: "no detectable scope; rephrase the description or supply `--plan`" |
| 1–6 scopes | proceed |
| 7+ scopes | HALT: "too broad — split into 2 autopilot runs" |

---

## Step 2 — Clarifying questions (max 3) — usually satisfied by plan mode

Plan mode's explore + AskUserQuestion phases typically cover these. If not, ask now:

- Is the spec clear enough that 5 fix iterations could realistically converge?
- Any out-of-scope hard limits the loop must NOT cross?

If the user can't answer cleanly, suggest `/rahul-dev-work` instead.

**Do NOT ask the user about sensitive paths.** Security review is automatic: every
`review_pending` tick runs `git diff --name-only` on the iteration's actual changes and
runs `/security-review` if any path is sensitive (see `rahul-dev-autopilot-tick`). The
`sensitive_paths_detected` field in the state schema is a *record the tick writes*, not an
up-front guess — leave it initialised `false`.

---

## Step 3 — Bootstrap state (no lock)

```bash
RUN_ID=$(date +%Y-%m-%dT%H-%M-%S)
STATE_DIR="$REPO_ROOT/_devlog/autopilot"
STATE_FILE="$STATE_DIR/$RUN_ID.json"
mkdir -p "$STATE_DIR" "$REPO_ROOT/.tmp/autopilot"
```

**Soft concurrency warning (never blocks):**

```bash
for f in "$STATE_DIR"/*.json; do
  [ -f "$f" ] || continue
  RUNNING=$(jq -r '.global_mode // ""' "$f" 2>/dev/null)
  RSCOPE=$(jq -r '[.packages[].name] | join(",")' "$f" 2>/dev/null)
  if [ "$RUNNING" = "running" ]; then
    echo "⚠️  Another autopilot run is active ($(basename "$f" .json), scope: $RSCOPE). Proceeding anyway."
  fi
done
```

Different scope or same scope — autopilot proceeds either way. No halt, no lock file.

### Write the state file

Set `description` to the approved plan summary (one short paragraph; the full plan path
goes in `plan_file`). One entry per detected scope:

```json
{
  "run_id": "<RUN_ID>",
  "schema_version": 3,
  "description": "<one-paragraph summary>",
  "plan_file": "<absolute path or null>",
  "repo_root": "<REPO_ROOT>",
  "packages": [
    {
      "name": "core",
      "kind": "package",
      "integration_branch": "integration/batch-core-<YYYY-MM-DD>",
      "feature_branch": null,
      "worktree_path": null,
      "issue_number": null,
      "pr_url": null,
      "mode": "plan",
      "iter": 0,
      "max_iter": 5,
      "history": [],
      "tried_fixes_hashes": [],
      "halt_reason": null,
      "sensitive_paths_detected": false,
      "staging_verified": false
    }
  ],
  "global_mode": "running",
  "started_at": "<ISO8601 UTC>",
  "max_wallclock_hours": 4,
  "max_cost_usd": 15,
  "cost_so_far_usd": 0,
  "halt_reason": null
}
```

Notes:
- `packages` is processed in array order (no separate ordering field). `core` is placed
  first at detection time so migrations land before consumers.
- A `global`-kind package has `integration_branch: null` and adds a `files_to_edit` array.
- `global_mode` lifecycle: `running` → `awaiting_human` (all packages clean) →
  `halted` (any package halted) → `session_closed` (after `/rahul-dev-done` archives it).
- Branch model: the tick reuses an existing open `integration/batch-<scope>-*` PR if one
  is found (`gh pr list --search "head:integration/batch-<scope>"`); otherwise it creates
  a fresh dated branch off `main`. Feature branches are `feature/issue-<N>-<slug>`, created
  off the integration branch, with the integration branch as the PR base.

---

## Step 4 — Hand off to the loop AND EXIT

**This is the only execution step. Run it, print the success message, and stop.**

Invoke `/loop /rahul-dev-autopilot-tick <RUN_ID>`. The loop self-paces via `ScheduleWakeup`.
The tick skill (running in fresh sub-agents) builds, cold-reviews, fixes, and on completion
pushes the narrative into the PR body and sends a PushNotification.

Print to the user (substitute fields from state):

```
🤖 Autopilot started.
   Run ID:        <RUN_ID>
   Repo:          <REPO_ROOT>
   Scope(s):      <comma-list of package names>
   State file:    <STATE_FILE>
   Stop limits:   5 iter per scope / 4h / $15

The loop is now running in fresh sub-agents. This conversation has finished its job.

You'll get a PushNotification when:
  ✅ Code-complete — narrative pushed to the PR body, PR(s) ready for review
  ⛔ Halted — a scope is stuck (the halt reason is posted as a PR comment)

Mid-run controls:
  /rahul-dev-whereami        — see live progress per scope
  /rahul-dev-done            — wrap + clean up merged worktrees when you're finished

⚠️  Final merge is always yours. Review the Deploy Preview / local verification, then
    squash-merge the integration PR. Autopilot never merges to main.
```

After printing, your turn ends. Do not run additional tools.

---

## Hard rules (read every time)

1. **Never auto-merge any integration PR → main.** The run ends at `global_mode=awaiting_human`
   with the PR(s) ready for human merge.
2. **No lock file.** A second run is never blocked. Same scope → one-line warning, proceed.
3. **No handoff markdown files.** The run narrative goes into the GitHub PR body via `gh pr edit`.
4. **State file is the source of truth.** Every tick reads + updates `<RUN_ID>.json` atomically
   (write to `.tmp`, `mv` over). The tick never deletes it — `/rahul-dev-done` archives it.
5. **Never invoke `/ultrareview` from inside the loop.** The tick posts a recommendation to
   the PR if it halts.
6. **One integration branch per scope.** `integration/batch-<scope>-YYYY-MM-DD`. Reuse the
   open one if it exists; never open a second for the same scope.
7. **🔒 Parent-conversation lockdown.** After Step 4 (`/loop` invocation): do NOT edit files,
   run tests/builds/git, or invoke other skills. The only allowed responses are confirmations
   or `/rahul-dev-whereami` if the user asks for status. Violating this makes you author +
   reviewer in one head — exactly what cold review avoids.

---

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Cold review never runs; you build everything in one conversation | After ExitPlanMode, agent treats "execute" as "build now" | Re-read the plan-mode handshake table above. |
| State file in the wrong repo | Skipped Step 1 routing / topicboard redirect | Always run Step 1 before Step 3. |
| Same findings hash twice → loop halts with `same_findings_twice` | Sub-agent fixing the wrong layer | Halt is correct — user must intervene. |
| Loop stops after a reboot | The `/loop` process died with the laptop | The tick's orphan detection catches this on the next manual tick; resume with `/rahul-dev-autopilot-tick <RUN_ID> --resume`. |

## See also

- `~/.claude/skills/rahul-dev-autopilot-tick/SKILL.md` — the per-tick state machine
- `~/.claude/skills/rahul-dev-work/SKILL.md` — the build/integration machinery the loop reuses
- `~/.claude/skills/rahul-dev-cold-review/SKILL.md` — the review machinery + guardrails
- `~/.claude/skills/rahul-dev-whereami/SKILL.md` — shows live autopilot runs across all repos
- `~/.claude/CLAUDE.md` § Multi-worktree integration protocol — the canonical branch rules
