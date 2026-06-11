---
name: rahul-dev-cold-review
description: Unbiased fresh-session code review - fetches PR diff cold, posts findings to PR. Triggers on cold review, /rahul-dev-cold-review, review the PR, fresh review, review my work.
model: sonnet
---

> **Cloud sessions:** Requires the `superpowers` plugin for the `/superpowers:requesting-code-review` sub-step. Install it with `/plugin install superpowers@claude-plugins-official` if not already available.

# /rahul-dev-cold-review — Unbiased fresh-session code review

## Purpose

Code review only catches what the reviewer can see *without* inheriting the author's blind spots. This skill ensures the reviewer starts cold:

1. Confirms the user has cleared context (or instructs them to)
2. Reads only the integration PR diff — not the prior planning conversation
3. Runs `/superpowers:requesting-code-review` against the diff
4. Posts findings to the PR for the author to act on

Designed to be invoked as the **second** session in a `/rahul-dev-work` → `/rahul-dev-cold-review` cycle.

## Where to run this

Run from the **main repo root** (e.g. `~/Desktop/GTC/hub/`). This is **Session 2** of the 3-session pipeline:

```
Session 1 (done):       /rahul-dev-work        ← built the feature
Session 2 (THIS):       /rahul-dev-cold-review  ← you are here
Session 3 (after merge):/rahul-dev-heal         ← run ONLY after fixes land
```

Do NOT run `/rahul-dev-heal` in this same session. It needs the fix commits to already be merged into the integration branch to be useful.

## Pre-flight

### 1. Cold-context check

If this session has prior conversation about the feature being reviewed, **stop and instruct**:

```
This session has prior context. To get an unbiased review:

1. Type /clear to reset context
2. Then re-run /rahul-dev-cold-review

The whole point of /rahul-dev-cold-review is fresh eyes — without /clear first, you're just re-asking the author.
```

If session is fresh (no prior dev conversation), proceed.

### 2. Find the integration PR

```bash
# Find the open integration PR for this repo
PR_INFO=$(gh pr list --base main --search 'head:integration/batch-' --state open --json number,title,headRefName,url -q '.[0]')
PR_NUM=$(echo "$PR_INFO" | jq -r .number)
INTEGRATION=$(echo "$PR_INFO" | jq -r .headRefName)
PR_URL=$(echo "$PR_INFO" | jq -r .url)
```

If no integration PR exists, ask: "No open `integration/batch-*` PR found — did `/rahul-dev-work` finish? Want me to look for a feature branch instead?"

### 3. Pull the diff

```bash
gh pr diff "$PR_NUM" > /tmp/rahul-dev-cold-review-diff.patch
gh pr view "$PR_NUM" --json title,body,files -q '{title,body,files: [.files[].path]}' > /tmp/rahul-dev-cold-review-meta.json
```

## Review

Invoke `/superpowers:requesting-code-review` with the diff + PR metadata as inputs. The skill spawns its 5 standard subagents (CLAUDE.md compliance, bug scan, blame/history context, prior PR comments, comment-rule compliance).

**Constraint:** the subagents read only:
- The diff
- The CLAUDE.md chain for touched directories
- Recent git blame for touched lines
- Prior comments on this PR (if it's been reviewed before — don't repeat)

They do **not** read the planning conversation, the original GitHub issue body (unless directly necessary), or unrelated repo files.

## Doc accuracy check

If the diff contains any `packages/<pkg>/CLAUDE.md` file:

1. For each `[NEW|CHANGED|REMOVED]` bullet in that CLAUDE.md's diff, verify the named
   function, endpoint, or table actually appears in the surrounding code diff for that package.
2. If a bullet's claim cannot be confirmed in the diff, flag it as a 🟡 SHOULD-FIX finding:
   "CLAUDE.md bullet `[CHANGED]: foo` — no matching code change found in diff."
3. If the diff touched code in `packages/<pkg>/` but `packages/<pkg>/CLAUDE.md` has no
   `[NEW|CHANGED|REMOVED]` bullets at all, flag it as a 🟡 SHOULD-FIX:
   "CLAUDE.md for `<pkg>` not updated — add `[NEW|CHANGED|REMOVED]` bullets for changed symbols."

Include these findings in the consolidated comment below.

## File Reference Existence Check (auto, every PR)

For each file in the diff, scan for string literals that are file references:
- `netlify.toml` — `to = "/<path>.html"` → verify `packages/website/<path>.html` via `git ls-files packages/website/<path>.html`
- JS/GS — `require('./<rel>')`, `import '…'` → resolve relative to the calling file, check `git ls-files`
- HTML — `href="<path>.html"`, `src="<path>"` → check against repo root or `packages/website/`

If any referenced file is absent in the repo:
  🔴 BLOCKER — "`<file>` references non-existent file: `<path>`. Add the file or fix the reference before merge."

Include these findings in the consolidated comment below.

## Production-domain redirect check

If the diff modifies any function that:
- references `REDIRECT_ALLOWED_ORIGINS`, or
- builds redirect URLs with a hardcoded production domain (`genetherapyconsultancy.com`, `topicboard.genetherapyconsultancy.com`)

Post a 🟡 SHOULD-FIX:
  "Redirect target hardcodes production domain — preview branches always route this code path to prod.
   Preview testing cannot validate this end-to-end. Document as a known limitation or add a deploy-preview override."

## Cross-scope dependency check

For each function, constant, or export **modified** in the diff:

1. Grep all other `packages/` directories for the symbol name: `grep -r "<symbol>" packages/ --include="*.js" --include="*.gs" --include="*.ts" -l`
2. If found in a **different** scope package:
   🟡 SHOULD-FIX: "Cross-scope dependency: `<symbol>` is modified here (`<scope>`) and referenced in `<other-scope>` (`<file>:<line>`). Verify the calling code is compatible with the new signature or behaviour."

## Post findings to the PR

After the subagents return, group findings by severity:

- 🔴 **BLOCKER** — must fix before merge (security, data loss, broken tests)
- 🟡 **SHOULD-FIX** — quality issues (missing error handling, unclear naming, missing tests for edge cases)
- 🟢 **NICE-TO-HAVE** — style, minor refactoring, extra documentation

Post one consolidated comment:

```bash
gh pr comment "$PR_NUM" --body "$(cat <<'EOF'
## Cold-review findings (fresh session, $(date +%Y-%m-%d))

### 🔴 Blockers
<list with file:line refs>

### 🟡 Should-fix
<list with file:line refs>

### 🟢 Nice-to-have
<list>

---
Run `/rahul-dev-work fix review comments on PR #$PR_NUM` to address blockers and should-fix.
After they're merged, run `/rahul-dev-heal` to capture systemic patterns.
EOF
)"
```

## Hand off

Print to user:

```
✅ Cold review posted to PR #<PR_NUM>: <PR_URL>

  🔴 <N> blockers
  🟡 <N> should-fix
  🟢 <N> nice-to-have

─── What to do next (in order) ────────────────────────────────
1. Fix blockers:
   - New session → cd ~/Desktop/GTC/hub
   - /rahul-dev-work fix review comments on PR #<PR_NUM>

2. After fixes are pushed and merged into integration:
   - New session → cd ~/Desktop/GTC/hub
   - /rahul-dev-heal   ← NOT before merge, or it has nothing to analyse

3. After /rahul-dev-heal:
   - /rahul-dev-done to wrap the session
───────────────────────────────────────────────────────────────
```

## Hard rules

1. **Always require fresh context.** If the user pushes back, explain why fresh context = unbiased review = catches more.
2. **Never modify code in this skill.** Review only. Fixes happen in a new `/rahul-dev-work` cycle or directly in the integration worktree.
3. **Always post findings to the PR**, never just to the chat — the PR is the record.
4. **Never run on `main`.** Always against the integration PR.
