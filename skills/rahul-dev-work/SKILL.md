---
name: rahul-dev-work
description: One-command dev entry: questions to issue to worktree to plan to build to verify to merge. Triggers on /rahul-dev-work, fix this, add this feature, lets build, new feature.
model: sonnet
---

# /rahul-dev-work — Sequential dev pipeline (issue → worktree → plan → build → integrate)

> ⚙️ **Primarily an internal dependency.** `/rahul-dev-autopilot-tick` dispatches this skill
> for its `build` and `fix` modes — it is the engine of the walk-away loop, so it must not be
> removed. For walk-away / multi-package / plan-file work, start with `/rahul-dev-autopilot`,
> NOT this skill. Direct use is reserved for the one documented case in
> `~/.claude/CLAUDE.md` § THE Pipeline: a **single-package change you drive at-desk**.

## Purpose

Cover **100% of dev work** with one command. The user types `/rahul-dev-work <description>` and answers a few questions. Everything else happens automatically. No decision about parallel vs sequential. No 700-line plan files. No `/dispatch`/`/compile-sessions` orchestration overhead.

This is the **manual single-operator equivalent of an Archon workflow** (https://github.com/coleam00/Archon): GitHub issue as spec → isolated worktree → fresh-context phases.

## Before you begin

**Directory check — run this first:**

```bash
pwd
git symbolic-ref -q HEAD >/dev/null || { echo "⚠️ Main repo is in detached HEAD. Run: git checkout main — then re-run /rahul-dev-work."; exit 1; }
```

The detached-HEAD guard halts gracefully — `/rahul-dev-work` needs a normal branch checkout.

`/rahul-dev-work` must be run from the **main repo root** (e.g. `~/Desktop/GTC/hub/`). If CWD is inside a worktree (path contains `.worktrees/` or `.claude/worktrees/`), stop and instruct:

```
You're inside a worktree. /rahul-dev-work must run from the main repo root.

  cd ~/Desktop/GTC/hub
  then re-run /rahul-dev-work
```

If the user typed `/rahul-dev-work` with no description, ask: "What do you want to ship?" then proceed.

## Step 1 — detect the repo

The user works across three Netlify-hosted repos. Auto-detect from CWD or ask:

| Repo | Path | Remote |
|------|------|--------|
| GTC Hub | `~/Desktop/GTC/hub/` | `gtc-hub` |
| TopicBoard | `~/Desktop/GTC/hub/` (worktree) or `gtc-topicboard` | `gtc-topicboard` |
| Website | `~/Desktop/GTC/website/Website_V5/` | `Website_V5` |

If CWD doesn't match any, ask: "Which repo? (1) hub, (2) topicboard, (3) website".

## Step 2 — clarifying questions (max 3)

Ask **only** what isn't obvious from the user's description. Skip questions where the answer is obvious. Examples:

- For a bug: "Regression or always-broken? Reproducible locally or only prod? Hot-fix or planned cleanup?"
- For a feature: "Roughly how big — 1 file or 10+? Any existing pattern in the repo to follow? Out-of-scope hard limit?"
- For a refactor: "What's the smell that triggered this? Behavioural change or pure refactor? Test coverage exists?"

If the description is already detailed enough, skip questions and move on.

## Step 3 — create the GitHub issue

```bash
# Pick template from the answer to "what kind of work":
#   bug → bug.yml, feature → feature.yml, refactor → refactor.yml
gh issue create --repo <owner>/<repo> \
  --title "<type>: <one-line summary>" \
  --label <feature|bug|refactor> \
  --body "$(cat <<'EOF'
## Why
<reason from user>

## What
<5-10 bullets — concrete changes>

## Out of scope
<hard limits — what this issue does NOT cover>

## Acceptance
<how we know it's done — tests pass, deploy preview shows X, etc.>
EOF
)"
```

**Hard cap: 150 lines.** If the body grows past that, the work is too big — split into 2 issues and stop.

Capture the new issue number (e.g. `#87`) for the rest of the pipeline.

## Step 4 — create the worktree off the integration branch

The branch format is canonical in `~/.claude/CLAUDE.md` § Multi-worktree integration
protocol — do not restate or hardcode it here. In short: detect the `SCOPE`, reuse the
open `integration/batch-<scope>-*` branch if one exists, else create a fresh dated one.

```bash
# Detect SCOPE from the description (scope list lives in ~/.claude/CLAUDE.md), then:
git fetch origin "+refs/heads/integration/batch-$SCOPE-*:refs/remotes/origin/integration/batch-$SCOPE-*"
INTEGRATION=$(git branch -r --list "origin/integration/batch-$SCOPE-*" | head -1 | sed 's|.*origin/||' | tr -d ' ')

if [ -z "$INTEGRATION" ]; then
  INTEGRATION="integration/batch-$SCOPE-$(date +%Y-%m-%d)"
  git fetch origin main
  git push origin "origin/main:refs/heads/$INTEGRATION"
  git fetch origin
fi

# Feature branch is feature/issue-<N>-<slug>, created off the integration branch
gh issue develop <ISSUE_N> --base "$INTEGRATION" --branch-name "feature/issue-<ISSUE_N>-<slug>" --checkout

# Add a worktree for isolation
git worktree add ".worktrees/issue-<ISSUE_N>" "feature/issue-<ISSUE_N>-<slug>"
cd ".worktrees/issue-<ISSUE_N>"
```

If a worktree already exists for this issue (re-running `/rahul-dev-work`): `cd` into it, don't recreate.

## Step 5 — plan in the worktree

Switch into plan mode. Invoke `/superpowers:writing-plans`. The plan should:
- Read the GitHub issue with `gh issue view <ISSUE_N>` as spec
- Reference any relevant files via `/CODEBASE_MAP.md` lookup or `graphify` query
- Output to `docs/plans/issue-<ISSUE_N>-<slug>.md`
- **Hard cap: 300 lines.** If the plan exceeds 300, the issue was too big — close the plan, split the issue (back to step 3), restart.

After the user approves the plan, exit plan mode and continue.

## Step 6 — build + verify (same Sonnet session)

- Write test stubs from the plan's Evals section first — for bug fixes the failing reproduction test is mandatory before any fix code. Implement to green. Run `/superpowers:verification-before-completion` against the plan's behavioral checklist. Do not push until verification passes.
- Commit incrementally with conventional-commit style: `feat(scope): …`, `fix(scope): …`, `refactor(scope): …`.
- Final commit body must include `Closes #<ISSUE_N>` so the issue auto-closes when integration merges to main.
- Run `/superpowers:verification-before-completion` before declaring done. **Non-negotiable.** Do not push until verification passes.

```bash
git push -u origin "feature/issue-<ISSUE_N>-<slug>"
```

## Step 7 — integrate (multi-worktree integration protocol per global CLAUDE.md)

```bash
cd ../..   # back to main repo or pick the integration worktree
git checkout "$INTEGRATION"
git pull --ff-only origin "$INTEGRATION"
git merge --no-ff "feature/issue-<ISSUE_N>-<slug>" -m "Merge issue #<ISSUE_N>: <one-line>"
git push origin "$INTEGRATION"
```

**PR handling** — exactly one open `integration/batch-*` PR at a time (global rule):

```bash
PR_NUM=$(gh pr list --base main --head "$INTEGRATION" --state open --json number -q '.[0].number')
if [ -z "$PR_NUM" ]; then
  gh pr create --base main --head "$INTEGRATION" \
    --title "Integration batch ${INTEGRATION#integration/batch-}" \
    --body "## Features included\n- [x] #<ISSUE_N> <one-line>\n\n## Pre-merge checklist\n- [ ] \`/rahul-dev-done\` run (session captured — branch safe to auto-delete)"
else
  # Append to existing PR body's "Features included" checklist
  gh pr view "$PR_NUM" --json body -q .body > .tmp/pr-body.md
  echo "- [x] #<ISSUE_N> <one-line>" >> .tmp/pr-body.md
  gh pr edit "$PR_NUM" --body-file .tmp/pr-body.md
fi
```

## Step 8 — hand off to /rahul-dev-cold-review

Print to user:

```
✅ Issue #<N> shipped to integration PR #<PR_NUM>.
Deploy Preview: <netlify URL — usually pulled from gh pr checks>

─── 3-session pipeline ────────────────────────────────────────
Session 1 (THIS session): /rahul-dev-work  ← you are here
Session 2 (NEW session):  /clear, then /rahul-dev-cold-review
Session 3 (after merge):  /rahul-dev-heal  ← only after fixes land
───────────────────────────────────────────────────────────────

Next: open a new terminal tab, type /clear, then /rahul-dev-cold-review.
Both cold-review and heal MUST be separate sessions — /clear is mandatory.
/rahul-dev-heal runs AFTER cold-review comments are fixed and merged, not before.
```

Suggest `/rahul-dev-done` only at end of full session, not after each `/rahul-dev-work`.

## Hard rules

1. **One `/rahul-dev-work` = one GitHub issue = one feature branch = one merge into integration.**
2. **Never** run `/dispatch`, `/plan-done --parts N`, `/compile-sessions`, or `/handoff` — they're archived.
3. **Never** push the feature branch directly to main. Always via integration.
4. **Never** open a per-feature PR against main. Always via the integration PR.
5. If the plan exceeds 300 lines or the issue body exceeds 150 lines: **stop and split.**
6. If `/cost` shows the session has crossed 150K tokens mid-feature: paste current state into the GitHub issue as a comment, `/clear`, resume in a new session.

## What gets captured automatically

The hub's `PostToolUse` hook (`_devlog/scripts/capture-tool-use.sh`) logs every Edit/Write/Bash to `_devlog/inbox/session-<date>-buffer.md`. You don't trigger it — it just happens. `/rahul-dev-done` promotes the buffer to durable Obsidian notes at end of session.
