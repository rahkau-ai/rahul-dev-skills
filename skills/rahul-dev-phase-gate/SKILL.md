---
name: rahul-dev-phase-gate
description: Assert the current project phase gate before proceeding (planning/building/review/merge). Triggers on check phase gate, phase gate, /rahul-dev-phase-gate.
---

# /rahul-dev-phase-gate — Runbook Gate Enforcer

## Purpose
Make runbook gates real. Each step of the build runbook ends in a gate ("I SAW it work", "rollback in one command", etc.). This skill pauses at the gate, walks the founder through the checklist, records pass/fail to `_devlog/log.md`, and updates `current-state.md`.

**Non-technical founder mode (default):** for any gate item that is a command the agent can run (e.g. "run the test suite"), the agent runs it on the founder's behalf, shows a clean result table, and asks the founder to visually confirm. The founder never has to type terminal commands. Only genuinely manual items (watching a browser, clicking a URL, checking brand) require the founder to act.

## Pre-flight (scoped contradiction scan + reversibility — per §15.9 and §15.10)

Fires when the gate is for a public-facing / irreversible step. Specifically:
- **`ship`, `live-demo`, harden `final`** — touch deploy target + main branch. Scan fires.
- **All other gates** — internal only. Scan skipped (not a cross-cutting surface).

For `ship` / `final`:
1. Contradiction scan: confirm deploy target in project CLAUDE.md matches what's about to happen; confirm merge policy (main vs. integration/) per global's Review Workflow.
2. Reversibility guard (§15.10 single-gate): before the actual merge or deploy command, print:
   ```
   ⚠️ Public-facing step.
   Action: <merge to main> / <deploy to <URL>>
   Reach: <who sees this, e.g. "gtc-topicboard production — all users">
   Revert cost: <what a code revert won't undo, e.g. "cached pages, indexed URLs, email-sent side effects">
   Approve to proceed? (y/n)
   ```
3. Only execute the merge/deploy on explicit `y`. Do not treat "tests pass" as approval.

## Invocation
```
/rahul-dev-phase-gate <gate-name>
```
Gate names (build runbook): `intent`, `brainstorm`, `scope`, `plan`, `worktree`, `build`, `simplify`, `live-demo`, `rollback`, `self-review`, `founder-brief`, `ship`.

Gate names (hardening runbook): `snapshot`, `inventory`, `baseline`, `security`, `quality`, `tests`, `observability`, `config`, `deploy`, `data`, `cost`, `docs`, `final`.

## Steps

### 1. Load the gate's checklist
Read the corresponding `👀 Founder manual checks` section from the runbook:
- **Build runbook:** `C:\Users\Me\Desktop\AI\claude-code-runbook.md` on laptop sessions. Not bundled in this plugin — fall back to asking the founder to open the runbook manually if file not found.
- **Harden runbook:** bundled in this plugin at `../rahul-dev-harden-project/claude-code-hardening-runbook.md`. Always available.

Do NOT paraphrase — use the exact checkbox items from the runbook.

### 2. Classify each checklist item

For every item in the checklist, decide:

| Class | Meaning | Who runs it |
|-------|---------|-------------|
| **AUTO** | Deterministic command (tests, lint, type check, git diff, script) | Agent runs, parses, presents result |
| **VISUAL-AUTO** | Browser action the agent can do via Claude Preview / screenshot (page renders, click flow) | Agent runs, shows screenshot + snapshot, founder confirms |
| **MANUAL** | Judgment or external-system act (brand voice, "does this feel right", click a prod URL) | Founder does it, pastes evidence |

Err toward AUTO/VISUAL-AUTO. The only time a non-technical founder should be asked to open a terminal is when no agent tool can do the job.

### 3. Present the checklist + auto-run AUTO items

Format:
```
🚦 Gate: <gate-name>

Auto-running the deterministic items now…
```

Then actually run each AUTO item:
- For tests → `npm test` / `pnpm test` / whatever is in `package.json scripts.test` (don't hard-code)
- For type check → the project's type check command
- For git diff → `git diff` or `git log`
- For "deliberate break" items in Gate 5 (tests) → pick one assertion target in source, change one character, re-run tests, confirm a test goes red, revert, confirm all green. Never leave the code in a broken state — always revert before returning.

Parse the output into a clean summary table:
```
| Item | Result | Evidence |
|------|--------|----------|
| Run test suite | ✅ 67/67 | `npm test` → Tests: 67 passed (67) |
| Deliberate break catches a regression | ✅ 66/67 when broken, 67/67 after revert | changed portal-api.mjs:719 amount:10→11, T2 went red, reverted, green |
```

Then present the MANUAL items and ask the founder to confirm each one (yes/no + one line of evidence).

### 4. Decide
- **All AUTO items pass + founder confirms all MANUAL items + gate statement true →** PASS
- **Any AUTO failure, founder "no" on MANUAL, or gate statement false →** FAIL

**STOP rule for test failures:** if tests fail during AUTO execution of Gate 5 (harden) or live-demo (build), do NOT "fix" by changing tests. Investigate each failure: is it (a) a real bug in source code, (b) a stale test written against older code, or (c) an environment/setup issue. Present verdicts to founder and wait for approval before editing anything.

### 5. Record

**On PASS:**
Append to `_devlog/log.md`:
```markdown
## [YYYY-MM-DD HH:MM] GATE PASS — <gate-name>
- **Runbook:** build | harden
- **Evidence:**
  - <item 1>: <evidence>
  - <item 2>: <evidence>
- **Next gate:** <name>
```

Update `_devlog/wiki/context/current-state.md` → bump the "Phase" line to reflect progress.

Append to `C:\Users\Me\Desktop\Obsidian\daily\<today>.md` (create if missing):
```markdown
- HH:MM — <project> — GATE PASS: <gate-name>
```

Then ask: `Context is growing. Run /compact before the next phase? (recommended after gates 4, 8, 12)`.

**On FAIL:**
Append to `_devlog/log.md`:
```markdown
## [YYYY-MM-DD HH:MM] GATE FAIL — <gate-name>
- **Blocking item(s):**
  - <item>: <why it failed>
- **Required fix:** <what needs to happen before retry>
```

Also update the hardening state file (if one exists) — `docs/audit/_progress.json` or `_devlog/harden-state.json` — with `status: "failed_gate"` and a concise notes field explaining the failure category (real bug / stale test / env). Never overwrite a prior `passed` state to `pending` — failures record as `failed_gate`.

Do NOT update current-state.md. Tell Rahul what step in the runbook to go back to, in plain English, with the exact next action.

### 6. Refuse to advance
Never mark a gate as passed on Rahul's behalf. If he skips a MANUAL item or the AUTO items failed, the gate stays FAIL. The entire value of this skill is that it enforces evidence.

## Gate-specific auto-checklists

These are the canonical AUTO recipes. Extend as new gates are added.

### harden `tests` (Gate 5)
1. Find test command: read `package.json` → `scripts.test` (or `test:unit`).
2. Run tests; capture pass/fail counts and failure messages.
3. Pick one "deliberate break" candidate: search source for an assertion target that a test checks (e.g. `expect(x).toBe(N)` → find matching literal in src). Change one character. Re-run tests. Confirm at least one test goes red. Revert. Re-run. Confirm all pass.
4. If step 2 shows any failure, do NOT proceed to step 3. Go to STOP rule above and investigate each failure with the founder before touching anything.

### build `live-demo`
1. Start dev server (or confirm already running) via `preview-worktrees` or `preview_start`.
2. Take a `preview_snapshot` of the main flow endpoint.
3. Read console logs and network requests; flag any errors.
4. Screenshot the key UI states and present to founder for the "does it look right" manual call.

### build `rollback`
1. Confirm the one-line revert command exists and is documented (e.g. `git revert <sha>` or `netlify rollback`).
2. Optionally dry-run it against a throwaway branch.

### harden `config` (Gate 7) — secrets population

Gate 7's founder checklist covers *documentation* quality, but the gate typically stays blocked on a different axis: **required env vars are not yet set in the deploy target**. That blocker is fully agent-automatable with the right CLI access. Dispatch this as a sub-agent (Sonnet is fine; no judgment required).

**Inputs:**
- `docs/runbook/configuration.md` — REQUIRED-var table with purpose + source per var
- `.env.example` — canonical name list
- `_progress.json` — Gate 7 notes often list the specific missing vars
- The deploy target's CLI (Netlify/Render/Vercel/Firebase) — read ambient auth config
- Per-service CLI (Stripe, Supabase, …) — same

**Procedure:**

1. **Diff needed-vs-set.** Read `.env.example` / `configuration.md` for the REQUIRED set. Read the live deploy target with its CLI (`netlify env:list --site <id> --context production`, etc.). Compute the missing set per context (production / deploy-preview / branch-deploy).

2. **Classify each missing var by source:**

   | Source class | Example | How the agent obtains it |
   |--------------|---------|--------------------------|
   | **RANDOM** | `HMAC_SECRET`, `JWT_SECRET`, `SESSION_SECRET` | `openssl rand -hex 32` — generate locally, never share. Scope: all contexts. |
   | **CLI-READABLE-TEST** | `STRIPE_SECRET_KEY` (test), Stripe webhook secrets (test), Supabase anon/service keys | Read from the tool's CLI config or API. Scope: `deploy-preview` + `branch-deploy` only. |
   | **CLI-WRITABLE-TEST** | Stripe webhook endpoints (test mode) | Create via CLI if not present, capture the returned secret. Scope: `deploy-preview` + `branch-deploy` only. |
   | **LIVE-DASHBOARD-ONLY** | `STRIPE_SECRET_KEY` (live), live webhook signing secrets, production DB passwords | **Do not generate, do not guess.** Leave unset. Emit a MANUAL checklist item with the dashboard URL + exact permissions needed. Scope: `production` (for the founder to set later). |

   Gate on the class. **Never write a live / payment-critical value to the `production` context automatically** — even if the CLI can read it — because the failure mode (misrouting real charges) is unrecoverable. The reversibility guard would fire here anyway; short-circuit by not even trying.

3. **Populate the safe classes with the right context scoping:**
   ```bash
   # RANDOM — all non-dev contexts, as --secret
   netlify env:set HMAC_SECRET "$(openssl rand -hex 32)" \
     --context production deploy-preview branch-deploy --site $SITE --secret --force

   # CLI-READABLE-TEST — preview/branch only, as --secret
   netlify env:set STRIPE_SECRET_KEY "$SK_TEST" \
     --context deploy-preview branch-deploy --site $SITE --secret --force

   # CLI-WRITABLE-TEST — create endpoint, capture secret, set it
   WHSEC=$(stripe webhook_endpoints create --url "$PROD_WEBHOOK_URL" \
     --enabled-events checkout.session.completed \
     --enabled-events payment_intent.payment_failed | jq -r '.secret')
   netlify env:set STRIPE_WEBHOOK_SECRET "$WHSEC" \
     --context deploy-preview branch-deploy --site $SITE --secret --force
   ```

   Notes:
   - `--secret` requires an explicit non-dev context list. Passing `--secret` with no context fails with "please specify a non-development context".
   - For the deploy-target site ID, prefer the value hard-coded in the project's CLAUDE.md (under the "Per-site specifics" block or equivalent). Do not infer from the Git remote.

4. **Trigger a rebuild** so functions see the new values:
   ```bash
   git commit --allow-empty -m "chore: trigger rebuild for Gate 7 env vars"
   git push origin <branch>
   ```

5. **Verify** by polling the project's `/health` (or equivalent) endpoint on the new preview deploy. Gate 7 closes only when every REQUIRED key in `/health` reports `true` (or the equivalent positive check) in the preview context.

6. **Report** as the gate evidence table:
   ```
   | Var | Class | Contexts set | Verification |
   |-----|-------|--------------|--------------|
   | HMAC_SECRET | RANDOM | all | /health.checks.hmacSecret = true ✅ |
   | STRIPE_SECRET_KEY | CLI-READABLE-TEST | preview+branch | key validates ✅ |
   | STRIPE_WEBHOOK_SECRET | CLI-WRITABLE-TEST | preview+branch | endpoint we_… created ✅ |
   | STRIPE_SECRET_KEY (live) | LIVE-DASHBOARD-ONLY | **unset** — manual ⚠️ | see MANUAL block below |
   ```

   Then list the MANUAL items the founder must still do for `production` scope. These stay as MANUAL items on the gate — the gate passes on the preview evidence, but the production-context unset vars become explicit items on the "before merging to main" checklist (not Gate 7's gate-statement, which is about docs quality).

**Deliberate-break verification** (same principle as Gate 5): after population, delete one of the RANDOM vars from one context via CLI, re-hit `/health`, confirm it reports `false`, restore it, confirm `true`. This proves the `/health` check is actually reading live env.

**Failure modes to watch:**
- Tool's CLI live mode is read-only by design (Stripe's `rk_live_*` from `stripe login`). That is correct security — do not try to work around it by creating a full `sk_live_*` key. Flag as LIVE-DASHBOARD-ONLY and move on.
- Deploy-preview URLs change per PR. Stripe webhook endpoints need a **stable** URL — always target the production URL even for the test-mode webhook. Test events sent to prod URL are rejected with signature mismatch (benign).
- `netlify env:set` with no `--context` defaults to "all contexts". Combined with `--secret`, that fails. Always list contexts explicitly when using `--secret`.

### harden `deploy` / `final` / build `ship`
Reversibility guard fires (see §Pre-flight). AUTO items still run, but merge/deploy needs explicit `y` from founder.

## Special behaviour
- If invoked without a `<gate-name>`, list the available gates and ask which one.
- If the previous gate's checklist hasn't been satisfied, warn: "Gate N-1 hasn't passed yet — are you skipping ahead?"
- If the project has no `package.json` or no test script, skip AUTO test runs and mark the test item as MANUAL with a note: "no test infra found — skipping auto-run".

## What NOT to do
- Do not auto-approve based on tests passing. "Tests pass" is one item on the Live Demo gate — not the whole gate.
- Do not compress the checklist. Every item comes from the runbook verbatim.
- Do not ask the founder to run terminal commands when the agent can run them. Only ask for manual action on judgment/visual items.
- Do not "fix" failing tests without founder approval — investigate each failure first (real bug vs. stale test vs. env issue).
- Do not leave the codebase in a broken state after a "deliberate break" — always revert before returning control to the founder, even if the gate fails.
