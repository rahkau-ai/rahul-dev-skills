# /rahul-magic-dev — Procedure

Read this file before any tool call. SKILL.md only carries the invocation summary; the
operational logic lives here.

---

## 1. Mental model (the composability rule)

| Layer | Skill | Job |
|---|---|---|
| Outer orchestrator | **`/rahul-magic-dev` (this skill)** | Split a multi-scope plan into per-scope jobs, manage one integration branch PER scope (singleton across sessions), spawn one autopilot session per scope, exit. |
| Inner engine | **`/rahul-dev-autopilot`** | Build + cold-review + fix loop for ONE scope inside ONE worktree. Halts at `awaiting_human`. |

**Magic-dev NEVER reimplements autopilot's fix loop.** It invokes autopilot N times in N
worktrees via `mcp__ccd_session__spawn_task`, then enters a passive monitoring loop via
`/rahul-magic-dev-tick`. There is no magic-dev fix loop and no mutable magic-dev state —
the monitoring tick is read-only and only reads state files owned by the autopilot sessions.

**Why this matters (the bug that archived the prior magic-dev):** a previous version bundled
work touching email + topicboard + website + LinkedIn + chrome extension into ONE
integration branch. The bundled diff was unreviewable — Rahul could not verify one part
was working without merging all of them. This procedure exists to prevent that recurrence.
Per-scope branches + per-scope reviewable PRs is non-negotiable.

---

## 2. Inputs and resolution

| Form | Behaviour |
|---|---|
| `--plan <path>` | Read the file. Cap 300 lines (existing v6 plan cap). If longer, halt with `plan exceeds 300-line cap — split or summarize`. |
| `--issues N,M,K` | For each: `gh issue view <N> --repo rahkau-ai/gtc-hub --json title,body,labels`. Concatenate bodies as the working spec; each issue becomes its own block. |
| bare invocation | Scan the current conversation transcript for the most recent plan / scope-spanning discussion. If ambiguous (multiple candidate plans, no clear primary), refuse and ask the user to pass `--plan <path>`. |
| `--dry-run` | Perform steps 3–6 only. Print the exit summary table (step 9) AND write the spawn receipt (step 9) with every scope `status:"dry_run"`. DO NOT call `gh issue create` (step 7), DO NOT call `mcp__ccd_session__spawn_task` (step 8). |
| `--batch-id <id>` | Use this exact value as `BATCH_ID` instead of generating a timestamp. The dev cycle passes its `CYCLE_ID` so the spawn receipt path is deterministic. |
| `--isolated` | **Throwaway / proof runs (the dev cycle's live proof always passes this).** Opt OUT of v6 singleton batching: each scope gets a BATCH_ID-namespaced integration branch `integration/batch-<scope>-proof-<BATCH_ID>` that is NEVER looked up or joined, so two same-day same-scope runs can never collide (Fix J). Step 5 skips the singleton lookup and always creates fresh; cleanup tears the proof branch down. Without this flag, magic-dev keeps v6 singleton-join behaviour. |

`BATCH_ID` — if `--batch-id <id>` was passed, `BATCH_ID="<id>"`; otherwise
`BATCH_ID = date +%Y-%m-%dT%H-%M-%S`. Capture ONCE per magic-dev invocation, reuse in
all per-scope file paths, worktree names, and the spawn receipt below.

`ISOLATED` — `true` iff `--isolated` was passed, else `false`. Capture ONCE alongside `BATCH_ID`.

**Repo-root resolution (Fix F — never hardcode the main repo).** Derive the repo root from
the caller's CWD so magic-dev operates on whatever checkout the orchestrator chose (the dev
cycle may run in a clone). Every `git -C` call below uses `$REPO_ROOT`, never a literal path:

```bash
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || echo "")
[ -n "$REPO_ROOT" ] || { echo "⛔ magic-dev must run inside a git repo (cwd=$(pwd))"; exit 1; }
```

---

## 3. Scope detection

**Canonical scope table:** `~/.claude/skills/rahul-dev-autopilot/SKILL.md` Step 1C (under
heading "1C — Detect scope(s)"). DO NOT duplicate the table here — read it from autopilot.

The v6 scopes are: `extension · website · dashboard · email · brand · video · core ·
business-os · topicboard · global`.

For each block of work in the input, match the keyword table against mentioned files,
folders, and feature names. A single block can touch multiple scopes — split it into
per-scope sub-blocks. Output of this step:

```
Map<scope, list-of-work-blocks>
```

Special cases:
- `topicboard` — hard-stop redirect per autopilot Step 1B. If detected and the spawn would
  not occur with CWD inside `packages/topicboard/`, the spawned autopilot will halt; for
  magic-dev's purposes, set the per-scope worktree path under `packages/topicboard/.worktrees/`.
- `global` — no worktree, no branch, no PR (per autopilot Step 1C). For `global`-scope work,
  DO NOT spawn an autopilot session — instead include `global` in the exit-summary table
  as `status: deferred (global — edit ~/.claude/ directly)` so Rahul handles it manually.
- `core` first — if `core` is among detected scopes and another scope depends on a migration
  in `core`, order matters. Magic-dev does NOT block other scopes on `core` (each runs in
  its own worktree off its own integration branch). If cross-scope migration ordering is
  needed, the user must serialize manually — surface this in the exit summary as a warning.

---

## 4. Per-scope spec synthesis

For each non-`global` scope, write a per-scope plan file at:

```
C:\Users\Me\.claude\plans\magic-dev-<BATCH_ID>-<scope>.md
```

Contents (in order):

1. `# magic-dev <scope> — batch <BATCH_ID>` title
2. `parent plan: <absolute path to original --plan file, or "conversation scan", or "issues N,M,K">`
3. (if applicable) the `## DO NOT TOUCH (active in sibling worktree <path>)` section from step 6
4. `## Work blocks` — the work blocks assigned to this scope (verbatim from the input, NOT rewritten)
5. `## Files identified` — bulleted list of file paths the work block(s) target (extracted from input or from the work block's `Files to modify` section)

Cap each per-scope plan at 300 lines. If exceeded, halt with `scope <X> exceeds 300-line
cap after split — manual triage needed`.

---

## 5. Singleton-branch resolution per scope

For each non-`global`, non-deferred scope:

### 5a. `--isolated` runs (Fix J) — BATCH_ID-namespaced, never join

If `ISOLATED=true`, SKIP the singleton lookup entirely and ALWAYS create a fresh
BATCH_ID-namespaced branch. Because `BATCH_ID` is unique per invocation, two same-day
same-scope runs can never name the same branch, so the collision that contaminated a real
PR (Fix J root cause) cannot recur:

```bash
INTEGRATION_BRANCH="integration/batch-<scope>-proof-$BATCH_ID"
# Defense in depth: the namespaced branch must NOT already exist (BATCH_ID is unique).
if git -C "$REPO_ROOT" show-ref --verify --quiet "refs/remotes/origin/$INTEGRATION_BRANCH" \
   || git -C "$REPO_ROOT" show-ref --verify --quiet "refs/heads/$INTEGRATION_BRANCH"; then
  echo "⛔ isolated branch $INTEGRATION_BRANCH already exists — refusing (re-run with a fresh --batch-id)"; exit 1
fi
git -C "$REPO_ROOT" branch "$INTEGRATION_BRANCH" main
git -C "$REPO_ROOT" push -u origin "$INTEGRATION_BRANCH"
WORKTREE_PATH="$REPO_ROOT/.worktrees/<scope>-$BATCH_ID"
git -C "$REPO_ROOT" worktree add -b "magic-dev/<scope>-$BATCH_ID-staging" \
  "$WORKTREE_PATH" "$INTEGRATION_BRANCH"
```

Isolated proof branches are EXEMPT from the v6 "≤1 open integration PR per scope"
invariant precisely because they are BATCH_ID-unique and torn down by `/rahul-dev-cleanup`.
Skip step 6 (interference check) for isolated runs — there are no siblings on a fresh
unique branch. Then continue at step 7.

### 5b. Normal runs — v6 singleton resolution (with foreign-proof guard)

```bash
# Look up the existing open integration branch for this scope across ALL sessions.
gh pr list --repo rahkau-ai/gtc-hub \
  --search "head:integration/batch-<scope>- state:open" \
  --json number,headRefName,headRefOid,updatedAt
```

Before acting on the results, **drop any `-proof-` branches** from the candidate set — an
isolated proof branch is a private throwaway and must never be joined by a normal batching
run (the inverse of Fix J):

```bash
CANDIDATES=$(gh pr list ... | jq '[ .[] | select(.headRefName | test("-proof-") | not) ]')
```

**0 candidates** → create a new integration branch + worktree:
```bash
INTEGRATION_BRANCH="integration/batch-<scope>-$(date +%Y-%m-%d)"
git -C "$REPO_ROOT" branch "$INTEGRATION_BRANCH" main
git -C "$REPO_ROOT" push -u origin "$INTEGRATION_BRANCH"
WORKTREE_PATH="$REPO_ROOT/.worktrees/<scope>-<BATCH_ID>"
# Feature branch created in step 7 once the issue number is known. Use a placeholder
# branch name `magic-dev/<scope>-<BATCH_ID>-staging` off integration HEAD for now:
git -C "$REPO_ROOT" worktree add -b "magic-dev/<scope>-<BATCH_ID>-staging" \
  "$WORKTREE_PATH" "$INTEGRATION_BRANCH"
```

**≥1 candidates** → JOIN the most recently updated integration branch:
```bash
INTEGRATION_BRANCH=$(echo "$CANDIDATES" | jq -r 'sort_by(.updatedAt) | reverse | .[0].headRefName')
git -C "$REPO_ROOT" fetch origin "$INTEGRATION_BRANCH"
WORKTREE_PATH="$REPO_ROOT/.worktrees/<scope>-<BATCH_ID>"
git -C "$REPO_ROOT" worktree add -b "magic-dev/<scope>-<BATCH_ID>-staging" \
  "$WORKTREE_PATH" "origin/$INTEGRATION_BRANCH"
```

Hard invariants (NEVER violate):
- Exactly ONE open integration branch per scope across ALL sessions — EXCEPT isolated
  `-proof-<BATCH_ID>` branches, which are exempt (5a) and torn down by cleanup.
- NEVER create a normal `integration/batch-<scope>-YYYY-MM-DD` if one already exists open.
- NEVER join a `-proof-` branch from a normal run (foreign-proof guard above).
- NEVER aggregate two scopes into one integration branch (the archived design's bug).

---

## 6. Interference check (only when joining an existing integration branch)

Skip this step if `ISOLATED=true` (step 5a — fresh BATCH_ID-namespaced branch, no siblings)
or if step 5b created a fresh integration branch (no siblings possible).
Otherwise, before spawning autopilot for a scope that joined a pre-existing branch:

1. **Enumerate sibling worktrees on the same integration branch:**
   ```bash
   git -C "$REPO_ROOT" worktree list --porcelain \
     | awk '/^worktree /{wt=$2} /^branch /{br=$2; print wt"|"br}' \
     | grep -v "$WORKTREE_PATH" \
     | while IFS='|' read wt br; do
         # Keep only worktrees whose branch traces back to the same integration branch.
         git -C "$wt" merge-base --is-ancestor "origin/$INTEGRATION_BRANCH" HEAD 2>/dev/null && echo "$wt"
       done
   ```

2. **For each sibling worktree, collect in-flight files:**
   ```bash
   git -C <sibling-wt> diff --name-only main...HEAD       # committed-but-unmerged
   git -C <sibling-wt> diff --name-only HEAD              # unstaged
   git -C <sibling-wt> diff --name-only --staged          # staged
   ```
   Union all three lists → "in-flight files for this scope".

3. **Compare with planned files for the new job** (the `## Files identified` list from
   step 4).

4. **Overlap handling:**

   | Overlap | Action |
   |---|---|
   | None | Proceed. No guard needed. |
   | Partial | Write a `## DO NOT TOUCH (active in sibling worktree <path>)` section at the TOP of the per-scope spec (step 4 file), listing the overlapping files. Spawned autopilot reads plan files top-down, so it will see this first. |
   | Total (every planned file is in-flight) | Mark scope as `blocked`. Do NOT spawn. Record in exit summary: `scope <X> blocked — all planned files in flight in <sibling-wt>, retry after that work merges`. |

---

## 7. GitHub issue creation per scope (skip in --dry-run)

For each non-blocked, non-`global` scope:

```bash
ISSUE_NUMBER=$(gh issue create \
  --repo rahkau-ai/gtc-hub \
  --title "<scope>: <slug>" \
  --body-file "C:/Users/Me/.claude/plans/magic-dev-<BATCH_ID>-<scope>.md" \
  --label "magic-dev,scope:<scope>" \
  | grep -oE '[0-9]+$')
```

Then rename the placeholder feature branch from step 5 to `feature/issue-<N>-<slug>`:

```bash
git -C "$WORKTREE_PATH" branch -m "magic-dev/<scope>-<BATCH_ID>-staging" "feature/issue-${ISSUE_NUMBER}-<slug>"
```

`<slug>` = lowercase-kebab of the first 4–6 words of the per-scope plan's title.

---

## 8. Spawn (skip in --dry-run)

For each non-blocked, non-`global`, non-deferred scope, call `mcp__ccd_session__spawn_task`
ONCE:

```
mcp__ccd_session__spawn_task({
  cwd: "<WORKTREE_PATH>",
  title: "magic-dev <scope>: issue-<N>",
  tldr: "<one-sentence summary of the scope's work>",
  prompt: "/rahul-dev-autopilot --plan C:\\Users\\Me\\.claude\\plans\\magic-dev-<BATCH_ID>-<scope>.md\n\n" +
          "Context: spawned by /rahul-magic-dev batch <BATCH_ID>.\n" +
          "Integration branch for this scope: <INTEGRATION_BRANCH>.\n" +
          "Feature branch (already created): feature/issue-<N>-<slug>.\n" +
          "If the plan file contains a `## DO NOT TOUCH` section, treat those files as immutable in this run."
})
```

After all spawns issued, magic-dev prints the exit summary (Step 9) and starts the batch monitor.

---

## 9. Exit summary

Print one Markdown table:

| scope | issue # | worktree | integration branch | status |
|---|---|---|---|---|
| email | 412 | `.worktrees/email-<BATCH_ID>` | `integration/batch-email-2026-05-28` | spawned |
| website | 413 | `.worktrees/website-<BATCH_ID>` | `integration/batch-website-2026-05-26` (joined) | spawned |
| topicboard | 414 | `packages/topicboard/.worktrees/topicboard-<BATCH_ID>` | `integration/batch-topicboard-2026-05-28` | spawned |
| extension | — | — | — | blocked (overlap in `.worktrees/extension-2026-05-27`) |
| global | — | — | — | deferred (edit `~/.claude/` directly) |

Then write the **spawn receipt** (one shot, never re-opened) and exit:

```bash
mkdir -p "$REPO_ROOT/_devlog/magic-dev"
# Build the scopes array from the same rows printed in the exit table.
# Each row: scope, issue (0 if none), worktree (abs path or ""), integration_branch (or ""),
# status ("spawned" | "blocked" | "deferred" | "dry_run"), reason (optional, for blocked/deferred).
jq -nc \
  --arg bid "$BATCH_ID" \
  --arg plan "${PLAN_FILE:-}" \
  --arg dry "${DRY:-false}" \
  --arg iso "${ISOLATED:-false}" \
  --argjson scopes "$SCOPES_RECEIPT_JSON" \
  '{ batch_id:$bid, created_at:(now|todate), plan_file:$plan,
     dry_run:($dry=="true"), isolated:($iso=="true"), scopes:$scopes }' \
  > "$REPO_ROOT/_devlog/magic-dev/$BATCH_ID.json"
echo "📝 Spawn receipt: $REPO_ROOT/_devlog/magic-dev/$BATCH_ID.json"
```

The spawn receipt is **not a state file** (rule 7 stands): magic-dev writes it exactly once
at exit and never re-reads, polls, or mutates it. It is a write-once manifest so the dev
cycle can locate the spawned work deterministically (scope → worktree / integration_branch /
issue / status) instead of globbing `.worktrees/`. The spawn_task chips remain the persistent
UI; per-scope autopilot state files at `<WORKTREE_PATH>/_devlog/autopilot/<RUN_ID>.json`
remain the only mutable runtime state. The receipt deliberately does NOT record `RUN_ID`
(autopilot generates that at its own startup, after magic-dev has exited).

Print the monitoring footer before starting the loop:

```
🔄 Batch <BATCH_ID> — monitoring active (wakes every ~5 min until all scopes terminal).
   Keep this window open. It will show aggregate scope status and send a push notification
   when all scopes complete. Close only after batch completion is confirmed.
```

Then invoke the monitoring loop:

```
/loop /rahul-magic-dev-tick <BATCH_ID>
```

`--dry-run` behavior: the receipt records all scopes as `status:"dry_run"` (no real spawns).
The tick immediately detects zero spawned scopes and exits with no ScheduleWakeup — safe to
start in all cases.

---

## 10. Hard rules

1. **NEVER aggregate scopes into one integration branch.** The archived design's bug. Per-scope branches only.
2. **NEVER spawn more than one autopilot per scope per invocation.**
3. **NEVER auto-merge integration → main.** Halt at `awaiting_human` (inherited from autopilot).
4. **NEVER skip the interference check** when joining an existing integration branch.
5. **NEVER use `git add -A`** in any spawned per-scope work (inherited from `/rahul-dev-work` rules).
6. **NEVER push `feature/*` directly to `main`.**
7. **NEVER write a magic-dev *state* file.** Fire-and-forget — no file magic-dev re-reads, polls, or mutates after exit. Mutable state lives in (a) spawn_task chips and (b) per-worktree autopilot JSON. The write-once **spawn receipt** at `_devlog/magic-dev/<BATCH_ID>.json` (step 9) is NOT a state file: it is authored once at exit and never touched again — a read-only manifest for the dev cycle.
8. **NEVER run an autopilot fix loop inside magic-dev's session.** Magic-dev's job ends at the exit summary in step 9. All builds, reviews, and fixes happen inside spawned sessions.
9. **NEVER reuse a worktree path.** `BATCH_ID` is in the worktree directory name to guarantee uniqueness across magic-dev invocations.
10. **NEVER skip `--dry-run` if asked.** When `--dry-run` is set, step 7 (`gh issue create`) and step 8 (`spawn_task`) MUST be skipped; only the plan files (step 4) and the exit summary table (step 9) are produced.
11. **NEVER hardcode the repo path.** Derive `$REPO_ROOT` from CWD (`git rev-parse --show-toplevel`) so the dev cycle can run magic-dev in a clone; every `git -C` uses `$REPO_ROOT` (Fix F).
12. **In `--isolated` mode, NEVER look up or join an existing scope branch** — always create `integration/batch-<scope>-proof-<BATCH_ID>` (Fix J). In NORMAL mode, NEVER join a `-proof-` branch (foreign-proof guard). The two branch namespaces never cross.

---

## See also

- `~/.claude/skills/rahul-dev-autopilot/SKILL.md` — inner engine. Canonical source for v6 scope keyword table (Step 1C), branch model, state-file schema.
- `~/.claude/CLAUDE.md` § "Multi-worktree integration protocol — single-tier (v6)" — invariants this skill enforces.
- `C:/Users/Me/Desktop/GTC/hub/_devlog/adr/0001-magic-dev-singleton-branch-protocol.md` — the decision record for why this skill exists and why per-scope singleton branches are non-negotiable.
- `~/.claude/skills/_archive-deprecated-2026-05-07/rahul-magic-dev/` — the previous (archived) design. DO NOT pattern-match against it; the bundling assumption is wrong.
